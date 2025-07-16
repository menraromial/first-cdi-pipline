## **Guide : Mettre en place un Horizontal Pod Autoscaler (HPA) basé sur la puissance électrique**

### **Objectif**

Ce guide explique comment configurer un Horizontal Pod Autoscaler (HPA) dans Kubernetes pour faire varier le nombre de réplicas d'une application en fonction de sa consommation électrique réelle, mesurée par Kepler et exposée via Prometheus.

### **Prérequis**

Avant de commencer, votre cluster Kubernetes doit disposer des composants suivants, installés et fonctionnels :
1.  **Prometheus** : Installé et configuré pour collecter les métriques du cluster.
2.  **Kepler** : Déployé et exportant les métriques de consommation électrique (`kepler_*`) vers Prometheus.
3.  **Prometheus Adapter** : Installé dans le cluster.

### **Architecture du flux de métriques**

Le flux de données suit ce chemin :

`Kepler` → `Prometheus` → `Prometheus Adapter` → `API Custom Metrics de Kubernetes` → `Contrôleur HPA`

### **Étape 1 : Vérification de la source de données**

La première étape est de s'assurer que Prometheus reçoit bien les métriques de puissance de Kepler pour les pods que l'on souhaite scaler.

1.  Accédez à l'interface de Prometheus.
2.  Exécutez la requête PromQL suivante pour vérifier la présence des métriques de puissance (zone `psys`) pour les pods avec les labels `pod_name` et `pod_namespace` :

    ```promql
    kepler_pod_cpu_watts{zone="psys", pod_name!="", pod_namespace!=""}
    ```

3.  Vous devriez voir une liste de résultats. Par exemple, pour une application de test nommée `charge-app` :

    ```
    kepler_pod_cpu_watts{..., pod_name="charge-app-56b8d77645-tmzvj", pod_namespace="default", ..., zone="psys"} 0.173
    ```
    Si cette requête ne renvoie rien, le problème se situe au niveau de Kepler ou de la configuration de scraping de Prometheus.

### **Étape 2 : Configuration du Prometheus Adapter**

Nous devons dire à l'adaptateur comment trouver, interpréter et exposer notre métrique de puissance à l'API de Kubernetes.

#### A. Création de l'APIService pour les métriques personnalisées

Cet objet enregistre l'API `custom.metrics.k8s.io` et dit à Kubernetes de diriger les requêtes vers notre `prometheus-adapter`.

Créez un fichier `custom-metrics-apiservice.yaml` :
```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  # Le nom doit être <version>.<group>
  name: v1beta1.custom.metrics.k8s.io
spec:
  # Le groupe d'API que nous enregistrons
  group: custom.metrics.k8s.io
  # Le service qui répondra aux requêtes pour ce groupe
  service:
    name: prometheus-adapter
    namespace: monitoring # Adaptez si votre adaptateur est dans un autre namespace
  # Version de l'API
  version: v1beta1
  # Priorités pour l'API Server
  groupPriorityMinimum: 100
  versionPriority: 100
  # Indique de ne pas vérifier le certificat TLS de l'adaptateur (courant pour les services internes)
  insecureSkipTLSVerify: true
```
Appliquez-le :
`kubectl apply -f custom-metrics-apiservice.yaml`

#### B. Configuration des règles dans la ConfigMap

Modifiez la `ConfigMap` utilisée par le `prometheus-adapter` (généralement nommée `adapter-config`) pour y ajouter la règle suivante. Cette règle définit notre métrique `pod_power_psys_watts`.

```yaml
# Extrait à ajouter dans la ConfigMap du prometheus-adapter, sous la clé "data.config.yaml"
rules:
  # Règle pour exposer une métrique de puissance par pod
  - seriesQuery: 'kepler_pod_cpu_watts{pod_name!="", pod_namespace!=""}'
    resources:
      # Fait le lien entre les ressources K8s et les labels Prometheus
      overrides:
        pod_namespace: {resource: "namespace"}
        pod_name: {resource: "pod"}
    name:
      # Nom de la métrique telle qu'elle sera connue par Kubernetes
      as: "pod_power_psys_watts"
    # Requête PromQL finale, utilisant des placeholders pour être générique et propre
    metricsQuery: 'sum(kepler_pod_cpu_watts{<<.LabelMatchers>>, zone="psys"}) by (<<.GroupBy>>)'
```
**Action cruciale :** Après avoir modifié la `ConfigMap`, redémarrez le pod du `prometheus-adapter` pour qu'il recharge la configuration :
`kubectl delete pod -n <namespace-monitoring> -l app.kubernetes.io/name=prometheus-adapter`

### **Étape 3 : Validation de l'exposition des métriques**

Vérifiez que l'API expose maintenant correctement la métrique.

```bash
# Remplacez "default" par le namespace de vos pods si nécessaire
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/pod_power_psys_watts" | jq .
```
Un résultat positif est une liste JSON contenant des `items`. Une erreur `NotFound` ou une liste vide (`"items": []`) indique un problème dans les étapes précédentes.

### **Étape 4 : Déploiement d'une application de test**

Pour observer le HPA en action, déployons une application qui consomme du CPU (et donc de la puissance).

Créez un fichier `charge-app.yaml` :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: charge-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: charge-app
  template:
    metadata:
      labels:
        app: charge-app
    spec:
      containers:
      - name: charge-container
        image: busybox
        command: ["/bin/sh", "-c", "while true; do i=0; while [ $i -lt 10000 ]; do i=$((i+1)); j=$((i*i)); done; sleep 0.1; done;"]
        resources:
          requests:
            cpu: "100m"
          limits:
            cpu: "500m"
```
Appliquez-le : `kubectl apply -f charge-app.yaml`

### **Étape 5 : Création et déploiement du HPA**

Enfin, créons l'objet HPA qui utilisera notre métrique personnalisée.

Créez un fichier `charge-app-hpa.yaml` :
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: charge-app-hpa-power
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: charge-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        # Le nom doit correspondre EXACTEMENT au "name.as" de la règle de l'adaptateur
        name: "pod_power_psys_watts"
      target:
        type: AverageValue
        # Cible de consommation moyenne par pod.
        # NOTE : Utilisez le format "Quantité" de K8s. "100m" signifie 100 milli-watts.
        # Réglez cette valeur en dessous de la consommation actuelle pour déclencher un scale-up.
        averageValue: "100m" 
```
Appliquez-le : `kubectl apply -f charge-app-hpa.yaml`

### **Étape 6 : Vérification du fonctionnement**

Surveillez le comportement du HPA et des pods.

1.  **Observer le HPA :**
    ```bash
    kubectl get hpa charge-app-hpa-power -n default -w
    ```
    Regardez la colonne `TARGETS`. Elle passera de `<unknown>/100m` à une valeur réelle comme `173m/100m`. La colonne `REPLICAS` s'ajustera en conséquence.

2.  **Observer les Pods :**
    ```bash
    kubectl get pods -n default -w
    ```
    Vous verrez de nouveaux pods `charge-app-...` être créés.

3.  **Analyser les événements :**
    ```bash
    kubectl describe hpa charge-app-hpa-power -n default
    ```
    La section `Events` vous donnera la raison exacte du scaling (ex: `custom metric pod_power_psys_watts above target`).

