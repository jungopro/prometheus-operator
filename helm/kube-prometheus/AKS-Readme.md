# Demo Setup

## Prereqs

- open readme in a seperate vscode window on mac screen, code on main screen with console open side-by-side
- presentation on main screen
- verify cname in godaddy monitoring.jungopro.com points to aks lb ip dns
- check cluster: 
  ```bash
  kubectl get nodes
  ```
- install helm:
  ```bash
  kubectl -n kube-system create sa tiller
  kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
  helm init --service-account tiller
  ```

- verify: 
  ```bash
  kubectl get pod --all-namespaces --watch
  helm version
  ```

## Deploy prometheus operator

- create ns: 
  ```bash
  kubectl create ns monitoring
  ```
- switch ns: 
  ```bash
  kubens monitoring
  ```
- add helm repo: 
  ```bash
  helm repo add coreos https://s3-eu-west-1.amazonaws.com/coreos-charts/stable/
  ```
- install operator: 
  ```bash
  helm install coreos/prometheus-operator --name prometheus-operator --namespace monitoring --timeout 600
  ```
- verify operator: 
  ```bash
  kll --watch
  ```
- show operator logs in a seperate window: 
  ```bash
  kubectl logs -f prometheus-operator...
  ```

## Deploy Kube-Prometheus

### Get Source Code

- get kube-prometheus: 
  ```bash
  helm fetch coreos/kube-prometheus --untar
  cd kube-prometheus/
  ```

### Chart Modifications

- Explain about the concept of umbrella chart and cascading values to child charts
- create demo-value.yaml:

```yaml
deployKubeScheduler: False
deployKubeControllerManager: False
deployKubeDNS: False

exporter-kubelets:
  # Set to false for Azure
  https: false # enable kubelet for nodes, note that 3 targets will be down since we don't have access to the masters

grafana:
  service:
    type: LoadBalancer
    loadBalancerIP: "51.144.126.90" # static AKS LB IP
  image:
    tag: 5.3.2
```

### Chart Deployment

- install kube-prometheus: 
  ```bash
  helm install --name kube-prometheus --namespace monitoring --debug -f demo-values.yaml .
  ```
- verify pods: 
  ```bash
  kll --watch
  ```
- verify grafana loaded: 
  ```bash
  kubectl logs -l app=kube-prometheus-grafana -c grafana
  ```
- view prometheus: 
```bash
kubectl port-forward prometheus-kube-prometheus-0 9090:9090
http://localhost:9090
```
  - view targets, explain about apiserver and kubelet with managed aks
- view grafana:
  - verify loadbalancer: 
  ```bash
  kubectl get svc
  ```
  - login (admin / admin):
    ```bash
    http://monitoring.jungopro.com
    ``` 
  - browse preconfigured charts

### Chart Modifications

- modify chart:

  - copy customization info into grafana folder:
    ```bash
    cp -r ../kube-prometheus-backup/charts/grafana/custom-da* charts/grafana; cp ../kube-prometheus-backup/charts/grafana/templates/custom* charts/grafana/templates
    ```

  - modify demo-values.yaml to include the new data:
    ```yaml
    grafana:
      serverDashboardConfigmaps:
        - custom-dashboards
        - custom-datasources
      extraVars:
      - name: GF_INSTALL_PLUGINS
        value: "grafana-azure-monitor-datasource,digrich-bubblechart-panel,grafana-clock-panel,grafana-piechart-panel,    jdbranham-diagram-panel"
    ```

- show cm status before upgrade: 
```bash
kubectl get cm
```
- upgrade chart:
```bash
helm upgrade kube-prometheus --debug -f demo-values.yaml .
```
- verify
  - check cm got created: helm
  ```bash
  kubectl get cm
  ```
  - check grafana pod updated: 
  ```bash
  kll --watch
  ```
  - verify grafana started successfully: 
  ```bash
  $ kubectl logs -l app=kube-prometheus-grafana -c grafana

  vl=info msg="Initializing HTTP Server" logger=http.server address=0.0.0.0:3000 protocol=http subUrl= socket=
  ```
  - refresh browser window and see the chart in the list
    - verify tags

