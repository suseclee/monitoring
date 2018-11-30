#### 1. Create a new namespace for monitoring 
```kubectl create namespace monitoring```
#### 2. git clone for a local monitor chart
```git clone https://github.com/suseclee/monitoring.git```
#### 3. Move to local chart repo 
```cd monitoring/charts```
#### 4. Install Prometheus and Grafana in the monitor chart
```helm install  ./monitor --namespace monitoring --name monitor```
#### 5. Check if prometheus is deployed well 
```kubectl -n monitoring get po | grep prometheus```
#### 6. Check if grafana is deployed well
```kubectl -n monitoring get po | grep grafana```
#### 6-1. you can try queries in Prometheus web: 10.17.2.0:31313 (optional).
#### 7. Login to grafana web: 10.17.2.0:31314
   * Login to grafana web with admin/admin (default username/password), enter new password.   
#### 8. Add a new dashboard in the garfana web - "Kubernetes All Nodes" will be shown "System load" panel only.
```
    1. hover your mousecursor over the + button on the left sidebar and click on the import.
    2. Paste the URL (for example) https://grafana.com/dashboards/3131 into the first input field (Grafana.com Dashboard) to import the "Kubernetes All Nodes" Grafana Dashboard. After pasting in the url, the view will change to another form.
    3. Now select the "Prometheus" datasource in the prometheus field and click on the import button.
```
#### 9. There are extra dashboards configured for CAASP. To apply these:
```
    1. git clone https://github.com/kubic-project/monitoring.git
    2. kubectl apply -f monitoring/grafana-dashboards-caasp-cluster.yaml
    3. kubectl apply -f monitoring/grafana-dashboards-caasp-namespaces.yaml
    4. kubectl apply -f monitoring/grafana-dashboards-caasp-nodes.yaml
    5. kubectl apply -f monitoring/grafana-dashboards-caasp-pods.yaml
```
#### 10. All the configured dashboards will be shown in grafana 
