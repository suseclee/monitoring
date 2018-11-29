# Monitoring Prometheus and Grafana without persistentVolumes
*Full Docs : https://w3.nue.suse.com/~mnapp/2018-11-19/book.caasp.admin/cha.admin.monitoring.html
This version is for a slim version of the original deployment. This does not require any storage and secret.
This version also disabled alertManager in Prometheus.*
#### 1. Create a new namespace for monitoring 
```kubectl create namespace monitoring```
#### 2. Create a Prometheus using helm with prometheus-config.yaml
```helm install stable/prometheus --namespace monitoring --name prometheus --values prometheus-config.yaml```
#### 3. Check if everything is deployed well
```kubectl -n monitoring get po | grep prometheus```
#### 4. Check the promethus web using the follwoing url and port
```
export NODE_PORT=$(kubectl get --namespace monitoring -o jsonpath="{.spec.ports[0].nodePort}" services prometheus-server)
export NODE_IP=$(kubectl get nodes --namespace monitoring -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT
```
#### 4-1. On Prometheus from url #4, you can query to Prometheus in Prometheus web (optional).
#### 5. Create a configMap for mapping between Prometheus and Grafana
```kubectl create -f prometheus-grafana-datasource.yaml``` 
#### 6. Create Grafana using helm with grafana-config.yaml
```helm install stable/grafana --namespace monitoring --name grafana --values grafana-config.yaml``` 
#### 7. Grafana will take up to 10 min. check if all the three pods are deployed.
```kubectl -n monitoring get po | grep grafana```
#### 8. Login to grafana web
   * Get Grafana url from the following:  
     ~~~
      export NODE_PORT=$(kubectl get -o jsonpath="{.spec.ports[0].nodePort}" services grafana -n monitoring)
      export NODE_IP=$(kubectl get nodes -o jsonpath="{.items[0].status.addresses[0].address}")
      echo http://$NODE_IP:$NODE_PORT/
     ~~~
   * Login to grafana web with admin/admin (default username/password), enter new password.   
#### 9. Add a new dashboard in the garfana web - "Kubernetes All Nodes" will be shown "System load" panel only.
```
    1. hover your mousecursor over the + button on the left sidebar and click on the import menuitem.
    2. Paste the URL (for example) https://grafana.com/dashboards/3131  (or just the ID 3131) into the first input field to import the "Kubernetes All Nodes" Grafana Dashboard. After pasting in the url, the view will change to another form.
    3. Now select the "Prometheus" datasource in the prometheus field and click on the import button.
```
#### 11. There are extra dashboards configured for CAASP. To apply these:
```
    1. git clone https://github.com/kubic-project/monitoring.git
    2. kubectl apply -f monitoring/grafana-dashboards-caasp-cluster.yaml
    3. kubectl apply -f monitoring/grafana-dashboards-caasp-namespaces.yaml
    4. kubectl apply -f monitoring/grafana-dashboards-caasp-nodes.yaml
    5. kubectl apply -f monitoring/grafana-dashboards-caasp-pods.yaml
```
#### 12. All the configured dashboards will be shown in grafana 
