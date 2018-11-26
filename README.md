# Monitoring Prometheus and Grafana without persistentVolumes
*Full Docs : https://w3.nue.suse.com/~mnapp/2018-11-19/book.caasp.admin/cha.admin.monitoring.html
This version is for a slim version of the original deployment. This does not require any storage and secret.
This version also disabled alertManager in Prometheus.*
#### 1. Create a new namespace for monitoring 
```kubectl create namespace monitoring```
#### 2. create prometheus-config.yaml
```
# Alertmanager configuration
alertmanager:
  enabled: false

# Create a specific service account
serviceAccounts:
  nodeExporter:
    name: prometheus-node-exporter

# Allow scheduling of node-exporter on master nodes
nodeExporter:
  hostNetwork: false
  hostPID: false
  podSecurityPolicy:
    enabled: true
  tolerations:
    - key: node-role.kubernetes.io/master
      operator: Exists
      effect: NoSchedule

# Disable Pushgateway
pushgateway:
  enabled: false

# Prometheus configuration
server:
  service:
    type: NodePort
  persistentVolume:
    enabled: false
```
#### 3. Create a Prometheus using helm with prometheus-config.yaml
```helm install stable/prometheus --namespace monitoring --name prometheus --values prometheus-config.yaml```
#### 4. Check if everything is deployed well
```kubectl -n monitoring get po | grep prometheus```
#### 5. Check the promethus web using the follwoing url and port
```
export NODE_PORT=$(kubectl get --namespace monitoring -o jsonpath="{.spec.ports[0].nodePort}" services prometheus-server)
export NODE_IP=$(kubectl get nodes --namespace monitoring -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT
```
#### 5-1. On Prometheus from url #5, you can query to Prometheus (optional).
#### 6. Create prometheus-grafana-datasource.yaml using url:port from #5
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
      url: http://<prometheus.server.ip.address>:<port>
      access: proxy
      orgId: 1
      isDefault: true
  ```
#### 7. Create a configMap for mapping between Prometheus and Grafana
```kubectl create -f prometheus-grafana-datasource.yaml``` 
#### 8. Create grafana-config.yaml
```
service:
  type: NodePort
  
# Configure admin password
adminPassword: admin

# Ingress configuration
ingress:
  enabled: false

# Configure persistent storage
persistence:
  enabled: false

# Enable sidecar for provisioning
sidecar:
  datasources:
    enabled: true
    label: grafana_datasource
  dashboards:
    enabled: true
    label: grafana_dashboard
```
#### 9. Create Grafana using helm with grafana-config.yaml
```helm install stable/grafana --namespace monitoring --name grafana --values grafana-config.yaml``` 
#### 10. grafana will take up to 10 min. check if all the three pods are deployed.
```kubectl -n monitoring get po | grep grafana```
#### 11. Login to grafana webr with username/password - admin/admin as a default from grafana-config.yaml. Get url from the following: 
```
export NODE_PORT=$(kubectl get -o jsonpath="{.spec.ports[0].nodePort}" services grafana -n monitoring)
export NODE_IP=$(kubectl get nodes -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT/
```
#### 12. Add a new dashboard in the garfana web - "Kubernetes All Nodes" will be shown "System load" panel only.
```
    1. On the Grafana from url #11 with admin/admin (username/password), enter new password.
    2. hover your mousecursor over the + button on the left sidebar and click on the importmenuitem.
    3. Paste the URL (for example) https://grafana.com/dashboards/3131 into the first input field to import the "Kubernetes All Nodes" Grafana Dashboard. After pasting in the url, the view will change to another form.
    4. Now select the "Prometheus" datasource in the prometheus field and click on the import button.
```
#### 13. There are extra dashboards configured for CAASP. To appply this
```
    1. git clone https://github.com/kubic-project/monitoring.git
    2. kubectl apply -f monitoring/grafana-dashboards-caasp-cluster.yaml
    3. kubectl apply -f monitoring/grafana-dashboards-caasp-namespaces.yaml
    4. kubectl apply -f monitoring/grafana-dashboards-caasp-nodes.yaml
    5. kubectl apply -f monitoring/grafana-dashboards-caasp-pods.yaml
```
#### 14. All the configured dashboards will be shown in grafana 
