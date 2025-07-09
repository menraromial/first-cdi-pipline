
```bash
git clone --depth 1 https://github.com/prometheus-operator/kube-prometheus; 
cd kube-prometheus;
```

```bash
KEPLER_EXPORTER_GRAFANA_DASHBOARD_JSON=`curl -fsSL https://raw.githubusercontent.com/sustainable-computing-io/kepler/main/grafana-dashboards/Kepler-Exporter.json | sed '1 ! s/^/         /'`
```

```bash
mkdir -p grafana-dashboards
```


```bash
cat - > ./grafana-dashboards/kepler-exporter-configmap.yaml << EOF
apiVersion: v1
data:
    kepler-exporter.json: |-
        $KEPLER_EXPORTER_GRAFANA_DASHBOARD_JSON
kind: ConfigMap
metadata:
    labels:
        app.kubernetes.io/component: grafana
        app.kubernetes.io/name: grafana
        app.kubernetes.io/part-of: kube-prometheus
        app.kubernetes.io/version: 9.5.3
    name: grafana-dashboard-kepler-exporter
    namespace: monitoring
EOF
```

```bash
yq -i e '.items += [load("./grafana-dashboards/kepler-exporter-configmap.yaml")]' ./manifests/grafana-dashboardDefinitions.yaml
yq -i e '.spec.template.spec.containers.0.volumeMounts += [ {"mountPath": "/grafana-dashboard-definitions/0/kepler-exporter", "name": "grafana-dashboard-kepler-exporter", "readOnly": false} ]' ./manifests/grafana-deployment.yaml
yq -i e '.spec.template.spec.volumes += [ {"configMap": {"name": "grafana-dashboard-kepler-exporter"}, "name": "grafana-dashboard-kepler-exporter"} ]' ./manifests/grafana-deployment.yaml
```


```bash
kubectl apply --server-side -f manifests/setup
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
kubectl apply -f manifests/
```

```bash
apt install make 
wget https://go.dev/dl/go1.22.3.linux-amd64.tar.gz
```

```bash
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.22.3.linux-amd64.tar.gz
```

```bash
nano ~/.bashrc
```

```bash
export PATH=$PATH:/usr/local/go/bin
```

```bash
git clone  https://github.com/sustainable-computing-io/kepler.git;
cd ./kepler;
```

```bash
make build-manifest OPTS="BM_DEPLOY PROMETHEUS_DEPLOY"
```

```bash
kubectl apply -f _output/generated-manifest/deployment.yaml
```

```bash
kubectl port-forward --address localhost -n monitoring service/grafana 3000:3000
```

```bash
kubectl port-forward --address localhost -n monitoring service/prometheus-k8s 9090:9090
```

```bash
kubectl port-forward --address localhost -n kepler service/kepler-exporter 9102:9102
```


```bash
curl -G 'http://localhost:9090/api/v1/query' \
    --data-urlencode 'query=kepler_node_core_joules_total{mode="dynamic"}'
```

### Docker local registry setup

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```


```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

```bash
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

```bash
mkdir go/src/sigs.k8s.io
```

```bash
docker push localhost:5000/scheduler-plugins/kube-scheduler:v20241118-
```


```bash
curl -X GET http://localhost:5000/v2/_catalog
```


```bash
git clone https://github.com/menraromial/scheduler-plugins.git
```


```bash

```


### Setup scaphandre

```bash
git clone https://github.com/hubblo-org/scaphandre
cd scaphandre
helm install scaphandre helm/scaphandre
```

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add kube-state-metrics https://kubernetes.github.io/kube-state-metrics
helm repo update

helm install prometheus prometheus-community/prometheus \
--set alertmanager.persistentVolume.enabled=false \
--set server.persistentVolume.enabled=false
```

```bash
kubectl port-forward deploy/prometheus-server 9090:9090
```

```bash
kubectl create configmap scaphandre-dashboard \
    --from-file=scaphandre-dashboard.json=docs_src/tutorials/grafana-kubernetes-dashboard.json
```

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install grafana grafana/grafana --values docs_src/tutorials/grafana-helm-values.yaml
```

```bash
kubectl get secret grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

```bash
kubectl port-forward deploy/grafana 3000:3000
```

```bash
curl -G 'http://localhost:9090/api/v1/query' \
    --data-urlencode 'query=scaph_host_power_microwatts'
```

```bash
curl -G 'http://localhost:9090/api/v1/query' \
    --data-urlencode 'query=scaph_host_power_microwatts{node="worker1"}'
```

#### Cleaning up
```bash
helm delete grafana prometheus scaphandre
```

```bash
kubectl port-forward service/wordpress-service 8080:80
```


### Scaphandre metrics

```bash
scaph_process_power_consumption_microwatts
```

### NextCloud
- Install
  ```bash
  helm repo add nextcloud https://nextcloud.github.io/helm/
  helm repo update
  helm install nextcloud nextcloud/nextcloud
  ```
- Port forward
  ```bash
  sudo kubectl port-forward service/nextcloud 8080:8080
  ```
  

```bash
1. Complete your nextcloud deployment by running:

  export APP_HOST=127.0.0.1
  export APP_PASSWORD=$(kubectl get secret --namespace default nextcloud -o jsonpath="{.data.nextcloud-password}" | base64 --decode)

  ## PLEASE UPDATE THE EXTERNAL DATABASE CONNECTION PARAMETERS IN THE FOLLOWING COMMAND AS NEEDED ##

  helm upgrade nextcloud nextcloud/nextcloud \
    --set nextcloud.password=$APP_PASSWORD,nextcloud.host=$APP_HOST,service.type=ClusterIP,mariadb.enabled=false,externalDatabase.user=nextcloud,externalDatabase.database=nextcloud,externalDatabase.host=YOUR_EXTERNAL_DATABASE_HOST

```


### Mattermost

- Installation
  ```bash
  helm repo add mattermost https://helm.mattermost.com
  helm repo update
  git clone https://github.com/mattermost/mattermost-helm.git
  cd charts/focalboard
  helm install focalboard . -n focalboard -f values.yaml
  cp values.yaml config.yaml
  helm install mattermost/focalboard
  ```
- Port forward
  ```bash
   sudo kubectl port-forward service/mattermost-focalboard 8080:80
  ```
  

```bash
NAME: focalboard
LAST DEPLOYED: Fri Nov 29 12:49:13 2024
NAMESPACE: focalboard
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace focalboard -l "app.kubernetes.io/name=focalboard,app.kubernetes.io/instance=focalboard" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace focalboard $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  kubectl --namespace focalboard port-forward $POD_NAME 80:$CONTAINER_PORT
  echo "Visit http://127.0.0.1:80 to use your application"
```


### STRAPI
Super Admin:

email: admin@strapidemo.com
password: welcomeToStrapi123


### Replace prometheus-k8s network policy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prometheus-k8s
  namespace: monitoring
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/component: prometheus
      app.kubernetes.io/instance: k8s
      app.kubernetes.io/name: prometheus
      app.kubernetes.io/part-of: kube-prometheus
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          app.kubernetes.io/name: prometheus
    ports:
    - protocol: TCP
      port: 8080
    - protocol: TCP
      port: 9090
  - from:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          app.kubernetes.io/name: prometheus-adapter
    ports:
    - protocol: TCP
      port: 9090
  - from:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          app.kubernetes.io/name: grafana
    ports:
    - protocol: TCP
      port: 9090
  egress:
  - to:
    - podSelector: {}
```

### Kepler Container metrics
```bash
kepler_container_joules_total{container="kepler-exporter",
container_id="a90477e08423cfd1cda876bbf6903543ab30e496e0331a94f6bf200852cf3b34",
container_name="cpu-intensive-app",
container_namespace="default",
endpoint="http",
instance="worker1",
job="kepler-exporter",
mode="dynamic",
namespace="kepler",
pod="kepler-exporter-hxh9s",
pod_name="cpu-intensive-app-d5665bc99-7wdpx",
service="kepler-exporter"}
```

1- **Example of queries**
- **get the maximum rate over a 1-hour period**
  ```bash
  max_over_time(irate(kepler_container_joules_total{pod_name=~".*fastapi-user.*"}[1m])[1h:])
  ```
  - **Return the maximum**
    ```bash
    max(max_over_time(irate(kepler_container_joules_total{pod_name=~".*fastapi-user.*"}[1m])[1h:]))
    ```
- **Calculate the 5-minute-averaged irate over a 1-hour period, at a 15-second subquery resolution**
  ```bash
  irate(kepler_container_joules_total{pod_name=~".*fastapi-user.*"}[1m])[1h:15m]
  ```


### Get all RAPL metrics for a specific node
```bash
kubectl get nodes worker2 -o jsonpath='{.metadata.labels}' | grep -oP '"[^"]*rapl[^"]*":\s*"[^"]*"'
```
### Modifier la valeur du label
```bash
kubectl label node worker2 rapl0/constraint_0_power_limit_uw=<new-value> --overwrite
kubectl label node worker2 rapl0/constraint_1_power_limit_uw=<new-value> --overwrite
kubectl label node worker2 rapl1/constraint_0_power_limit_uw=<new-value> --overwrite
kubectl label node worker2 rapl1/constraint_1_power_limit_uw=<new-value> --overwrite
```
### Supprimer les labels
```bash
kubectl label node worker3 rapl0/constraint_0_power_limit_uw-
kubectl label node worker3 rapl0/constraint_1_power_limit_uw-
kubectl label node worker3 rapl1/constraint_0_power_limit_uw-
kubectl label node worker3 rapl1/constraint_1_power_limit_uw-
```
### Supprimer une annotation
```bash
kubectl annotate node worker3 power-manager/initialized-
```
### get network policy as yaml
```bash
kubectl get networkpolicy grafana -n monitoring -o yaml
```

### get service as yaml
```bash
kubectl get svc grafana -n monitoring -o yaml
```

### Expose grafana svc as NodePort
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 11.2.1
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/component: grafana
      app.kubernetes.io/name: grafana
      app.kubernetes.io/part-of: kube-prometheus
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: prometheus
    - ipBlock:
        cidr: 0.0.0.0/0    # Permet l'accès externe
    ports:
    - port: 3000
      protocol: TCP
  egress:
  - {}
---
apiVersion: v1
kind: Service
metadata:
  name: grafana-nodeport
  namespace: monitoring
  labels:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 11.2.1
spec:
  type: NodePort
  ports:
  - name: http
    port: 3000         # Port interne reste 3000
    protocol: TCP
    targetPort: http
    nodePort: 30300    # Port externe 30300
  selector:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
```


### Modify the default retention time of TSDB 
```yml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.54.1
  name: k8s
  namespace: monitoring
spec:
  alerting:
    alertmanagers:
    - apiVersion: v2
      name: alertmanager-main
      namespace: monitoring
      port: web
  enableFeatures: []
  externalLabels: {}
  image: quay.io/prometheus/prometheus:v2.54.1
  nodeSelector:
    kubernetes.io/os: linux
  podMetadata:
    labels:
      app.kubernetes.io/component: prometheus
      app.kubernetes.io/instance: k8s
      app.kubernetes.io/name: prometheus
      app.kubernetes.io/part-of: kube-prometheus
      app.kubernetes.io/version: 2.54.1
  podMonitorNamespaceSelector: {}
  podMonitorSelector: {}
  probeNamespaceSelector: {}
  probeSelector: {}
  replicas: 2
  resources:
    requests:
      memory: 400Mi
  ruleNamespaceSelector: {}
  ruleSelector: {}
  scrapeConfigNamespaceSelector: {}
  scrapeConfigSelector: {}
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  serviceAccountName: prometheus-k8s
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  version: 2.54.1
  retention: 30d

```
### kcas-scheduler conf

```yml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
leaderElection:
  # (Optional) Change true to false if you are not running a HA control-plane.
  leaderElect: false
clientConnection:
  kubeconfig: /etc/kubernetes/scheduler.conf
profiles:
- schedulerName: kcas-scheduler
  plugins:
    queueSort:
      enabled:
      - name: PrioritySort
    preFilter:
      enabled:
      - name: CarbonAware
    filter:
      enabled:
      - name: CarbonAware
    preScore:
      enabled:
      - name: CarbonAware
    score:
      enabled:
      - name: CarbonAware
    reserve:
      enabled:
      - name: CarbonAware
    bind:
      enabled:
      - name: DefaultBinder
    multiPoint:
      disabled:
      - name: "*"
```
---

### Supprimer des Namespace qui sont en mode Terminating

#### Explication du problème

Quand vous demandez la suppression d'un namespace (`kubectl delete ns mon-namespace`), Kubernetes ne le supprime pas immédiatement. Il le passe en état `Terminating` et attend que tous les contrôleurs qui ont des "finaliseurs" sur ce namespace aient terminé leur travail de nettoyage (suppression de LoadBalancers, de CRDs, etc.). Si le contrôleur qui a posé le finaliseur est défaillant, désinstallé ou ne répond plus, le finaliseur n'est jamais retiré et le namespace reste bloqué.

#### Solution : Forcer la suppression (Méthode recommandée)

La méthode la plus propre et la plus rapide est d'utiliser un petit script qui va :
1.  Récupérer la configuration du namespace en JSON.
2.  Supprimer la clé `finalizers` de sa spécification.
3.  Envoyer cette configuration modifiée à l'API Kubernetes via un point d'accès spécial (`finalize`).

Voici un script que vous pouvez exécuter directement dans votre terminal pour forcer la suppression des trois namespaces bloqués (`ingress-nginx`, `kubevirt`, `zipkin`).

**Prérequis :** Assurez-vous d'avoir l'outil `jq` installé. C'est un processeur JSON en ligne de commande très courant. Si vous ne l'avez pas :
*   Sur Debian/Ubuntu : `sudo apt-get install jq`
*   Sur macOS (avec Homebrew) : `brew install jq`
*   Sur RHEL/CentOS : `sudo yum install jq`

#### Script à copier-coller

```bash
# Liste des namespaces à forcer
NAMESPACES_TO_DELETE="ingress-nginx kubevirt zipkin"

for ns in $NAMESPACES_TO_DELETE; do
  echo "Tentative de suppression forcée du namespace: $ns"
  
  # Récupère la définition du namespace, retire les finaliseurs et met à jour via l'endpoint 'finalize'
  kubectl get namespace "$ns" -o json | \
  jq '.spec.finalizers = []' | \
  kubectl replace --raw "/api/v1/namespaces/$ns/finalize" -f -
  
  echo "La commande de suppression forcée pour '$ns' a été envoyée."
done

echo "Vérifiez l'état des namespaces dans quelques instants avec 'kubectl get ns'."
```

Exécutez ce bloc de code dans votre terminal. Normalement, après quelques secondes, les namespaces devraient disparaître lorsque vous refaites un `kubectl get ns`.

#### Alternative : Méthode Manuelle (plus complexe, si vous n'avez pas `jq`)

Si vous ne pouvez pas installer `jq`, vous pouvez le faire manuellement pour chaque namespace, mais c'est plus laborieux.

**Étape 1 : Lancer `kubectl proxy`**

Ouvrez un **nouveau terminal** et lancez :
```bash
kubectl proxy
# Starting to serve on 127.0.0.1:8001
```
Laissez ce terminal ouvert. Il crée un pont sécurisé vers votre API server Kubernetes.

**Étape 2 : Éditer et forcer la mise à jour**

Ouvrez un **autre terminal** et exécutez ces commandes pour chaque namespace à supprimer (remplacez `<NAMESPACE_A_SUPPRIMER>` par `ingress-nginx`, puis `kubevirt`, etc.).

```bash
# Remplacez <NAMESPACE_A_SUPPRIMER> par le nom du namespace, ex: ingress-nginx
NAMESPACE_TO_DELETE="ingress-nginx"

# 1. Récupérer la définition du namespace et retirer le finalizer
kubectl get namespace $NAMESPACE_TO_DELETE -o json | \
  sed 's/"kubernetes"//' > temp_namespace.json
# Note: La commande sed est basique et retire juste le mot "kubernetes". 
# Si vous avez d'autres finalizers, il faudra éditer le fichier manuellement.

# 2. Envoyer la version modifiée à l'API via le proxy
curl -k -H "Content-Type: application/json" -X PUT \
  --data-binary @temp_namespace.json \
  http://127.0.0.1:8001/api/v1/namespaces/$NAMESPACE_TO_DELETE/finalize

# 3. Nettoyer
rm temp_namespace.json
```

