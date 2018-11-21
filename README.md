# Monitoring Prometheus and Grafana
Full Docs : https://w3.nue.suse.com/~mnapp/2018-11-19/book.caasp.admin/cha.admin.monitoring.html
### This version is for a slim version of the original deployment. This does not require any storage and secret.
### This version also disabled alertManager in Prometheus.
1. ```kubectl create namespace monitoring```
2. Create rbac.yaml file
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring
roleRef:
  kind: ClusterRole
  name: suse:caasp:psp:privileged
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: default
  namespace: monitoring
- kind: ServiceAccount
  name: prometheus-alertmanager
  namespace: monitoring
- kind: ServiceAccount
  name: prometheus-kube-state-metrics
  namespace: monitoring
- kind: ServiceAccount
  name: prometheus-node-exporter
  namespace: monitoring  
- kind: ServiceAccount
  name: prometheus-pushgateway
  namespace: monitoring  
- kind: ServiceAccount
  name: prometheus-server
  namespace: monitoring  
- kind: ServiceAccount
  name: grafana
  namespace: monitoring 
  ```
3. ```kubectl apply -f  rbac.yaml```
4. create prometheus-config.yaml
```
# Alertmanager configuration
alertmanager:
  enabled: false
# Prometheus configuration
server:
  persistentVolume:
    enabled: false
```
5. ```helm install stable/prometheus --namespace monitoring --name prometheus --values prometheus-config.yaml```
6. Check if everything is deployed well
```kubectl -n monitoring get po | grep prometheus```
7. Get the ip address for prometheus-server
```kubectl get service -n monitoring| grep prometheus-server|awk '{print $3}```
8. Create prometheus-grafana-datasource.yaml
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: grafana-datasources
  namespace: monitoring
  labels:
     grafana_datasource: "1"
data:
  datasource.yaml: |-
    apiVersion: 1
    deleteDatasources:
      - name: Prometheus
        orgId: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://<prometheus.server.ip.address>:80
      access: proxy
      orgId: 1
      isDefault: true
  ```
9. ```kubectl create -f prometheus-grafana-datasource.yaml``` 
10. Create grafana-config.yaml
```
service:
  type: NodePort
persistence:
  enabled: false
adminUser: admin
adminPassword: admin
# Enable sidecar for provisioning
sidecar:
  datasources:
    enabled: true
    label: grafana_datasource
  dashboards:
    enabled: true
    label: grafana_dash
```
11. ```helm install stable/grafana --namespace monitoring --name grafana --values grafana-config.yaml``` 
12. grafana will take up to 10 min. check if all the three pods are deployed.
```kubectl -n monitoring get po | grep grafana```
13. Login to grafana-server with username/password from grafana-config.yaml
```
export NODE_PORT=$(kubectl get -o jsonpath="{.spec.ports[0].nodePort}" services grafana -n monitoring)
export NODE_IP=$(kubectl get nodes -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT/
```

14. Add a new dashboard - "Kubernetes All Nodes" will be shown "System load" panel only.
    ```
    1. On the home page of Grafana, hover your mousecursor over the + button on the left sidebar and click on the importmenuitem.
    2. Paste the URL (for example) echo http://$NODE_IP:$NODE_PORT/into the first input field to import the "Kubernetes All Nodes" Grafana Dashboard. After pasting in the url, the view will change to another form.
    3. Now select the "Prometheus" datasource in the prometheus field and click on the import button.
    ```
15. There are extra dashboards configured for CAASP. To appply this
    ```
    1. git clone https://github.com/kubic-project/monitoring.git
    2. kubectl apply -f monitoring/grafana-dashboards-caasp-cluster.yaml
    3. kubectl apply -f monitoring/grafana-dashboards-caasp-namespaces.yaml
    4. kubectl apply -f monitoring/grafana-dashboards-caasp-nodes.yaml
    5. kubectl apply -f monitoring/grafana-dashboards-caasp-pods.yaml
    ```
16. All the configured dashboards will be shown in grafana 
