
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
