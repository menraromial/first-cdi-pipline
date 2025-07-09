# µBench Tutorial

### Run the µBench Container
```zsh
docker run -it -id --name mubench -v ~/.kube/config:/root/.kube/config msvcbench/mubench
```

Update the `server` key of the `config` file with the correct IP address of the master node of the cluster, if necessary. Verify that the µBench container can access your cluster by using the following command from your host:
```zsh
docker exec mubench kubectl get nodes
```

### Enter the µBench Container
```zsh
docker exec -it mubench bash
```

Now your terminal should be in the µBench container from which you will run the next commands:
```zsh
╱╱╱╭━━╮╱╱╱╱╱╱╱╱╱╭╮
╱╱╱┃╭╮┃╱╱╱╱╱╱╱╱╱┃┃
╭╮╭┫╰╯╰┳━━┳━╮╭━━┫╰━╮
┃╰╯┃╭━╮┃┃━┫╭╮┫╭━┫╭╮┃
┃┃┃┃╰━╯┃┃━┫┃┃┃╰━┫┃┃┃
╰┻┻┻━━━┻━━┻╯╰┻━━┻╯╰╯

root@64ae03d1e5b8:~muBench#
```

### Deploy a µBench Example App
```zsh
cd $HOME/muBench
python3 Deployers/K8sDeployer/RunK8sDeployer.py -c Configs/K8sParameters.json
```

### Check the Deployment
```zsh
kubectl get pods
```
You should see the following pods:
```zsh
root@64ae03d1e5b8:~/muBench# k get pods
NAME                        READY   STATUS    RESTARTS   AGE
gw-nginx-5b66796c85-fpqvc   2/2     Running   0          11m
s0-7d7f8c875b-gk2pq         2/2     Running   0          11m
s1-8fcb67d75-pncwq          2/2     Running   0          11m
s2-558f544b94-kft64         2/2     Running   0          11m
s3-79485f9857-5j79h         2/2     Running   0          11m
s4-9b6f9f77b-dklvm          2/2     Running   0          11m
s5-6ccddd9b47-n5pz7         2/2     Running   0          11m
s6-7c87c79cd6-pt26s         2/2     Running   0          11m
s7-5fb7cbff7c-hkd6t         2/2     Running   0          11m
s8-5549949968-72q2z         2/2     Running   0          11m
s9-9576b784c-4npsj          2/2     Running   0          11m
```


## Monitoring

> **_NOTE:_**: Si vous utilisez la stack **prometheus-k8s** pour **Kepler** rassurez vous d'avoir ajouter le label `app: kiali` dans le **NetworkPolicy** de prometheus

Si vous n'avez pas encore Prometheus-Grafana, decommentez la partie du script associée. 


µBench utilise Prometheus, Grafana, Istio, Kiali et Jaeger pour obtenir des métriques et des traces des applications générées, comme décrit ci-dessous.
Le fichier `monitoring-install.sh` installe ce framework dans le cluster. Il peut être exécuté soit depuis le bash Docker de µBench, soit depuis le shell de l’hôte avec :

```zsh
sh ./monitoring-install.sh
```


```zsh
# Secure .kube
chmod go-r -R ~/.kube/

# Prometherus
# kubectl create namespace monitoring
# helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
# helm repo update
# helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring

# Prometheus (30000) and Grafana (30001) NodePort Services
# kubectl apply -f prometheus-nodeport.yaml -n monitoring
# kubectl apply -f grafana-nodeport.yaml -n monitoring

# Istio
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
kubectl create namespace istio-system
helm install istio-base istio/base -n istio-system
helm install istiod istio/istiod -n istio-system --set global.proxy.tracer="zipkin" --wait
helm install istio-ingressgateway istio/gateway -n istio-system
kubectl label namespace default istio-injection=enabled

# Istio - Prometeus integration
kubectl apply -f istio-prometheus-operator.yaml

# Jarger
kubectl apply -f jaeger.yaml

# Jaeger NodePort Service (30002)
kubectl apply -f jaeger-nodeport.yaml

#Kiali
helm repo add kiali https://kiali.org/helm-charts
helm repo update
helm install \
  -n istio-system \
  -f kiali-values.yaml \
  kiali-server \
  kiali/kiali-server

#Kiali NodePort Service (30003)
kubectl apply -f kiali-nodeport.yaml

```


### Ma première application µBench

Depuis le conteneur µBench ou l’hôte, accédez au dossier `muBench` et exécutez :

```zsh
python3 Deployers/K8sDeployer/RunK8sDeployer.py -c Configs/K8sParameters.json
```

Cette commande crée l’application µBench décrite dans `Configs/K8sParameters.json`.
Elle utilise le fichier [Examples/workmodel-serial-10services.json](#) qui spécifie une application composée de 10 microservices.
Les clients envoient des requêtes au service **s0**, et **s0** appelle séquentiellement tous les autres services avant de renvoyer le résultat aux clients.
Chaque service sollicite également le processeur (CPU).



Pour charger l'application, vous pouvez utiliser le [Runner](#runner) de µBench :

```zsh
python3 Benchmarks/Runner/Runner.py -c Configs/RunnerParameters.json
```

Vous devriez voir quelque chose comme ceci :

```zsh
root@64ae03d1e5b8:~/muBench# python3 Benchmarks/Runner/Runner.py -c Configs/RunnerParameters.json
###############################################
############   Run Forrest Run!!   ############
###############################################
Heure de début : 09:13:04.291510 - 23/01/2023
Requête traitée 2, latence 139, requêtes en attente 1
Requête traitée 13, latence 129, requêtes en attente 1
Requête traitée 24, latence 139, requêtes en attente 1
....
```

> **_REMARQUE:_**: N'oubliez pas de changer `ms_access_gateway` dans `Configs/RunnerParameters.json` avec les configs de votre **ingress**



> ***REMARQUE:*** Modifiez `Configs/K8sParameters.json` si votre service de résolution DNS Kubernetes est différent de `kube-dns`. Par exemple, dans certains clusters, il s’appelle `coredns`. Changez egalementles valeurs dans `kiali-values.yaml` pour avoir les bonnes configs de votre cluster

Pour désinstaller l’application µBench, utilisez la commande suivante :

```zsh
kubectl delete -f SimulationWorkspace/yamls/
```

