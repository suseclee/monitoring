# Monitoring Prometheus and Grafana with NFS persistentVolumes

*Full Docs : https://w3.nue.suse.com/~mnapp/2018-11-19/book.caasp.admin/cha.admin.monitoring.html. This version is for a slim version of the original deployment. This does not require any storage and secret. This version also disabled alertManager in Prometheus. NFS server is 10.86.1.244:/var/nfs which is behind SUSE VPN.*

#### 1. Create a new namespace for monitoring 
```kubectl create namespace monitoring```
#### 2. Create PV and PVC  
```kubectl apply -f pv-pvc-monitoring.yaml```
#### 3. Verify the PV and PVC  
+ ```kubectl get pv```  
+ ```kubectl get pvc -n monitoring```  
#### 4. Testing persistentVolumeClaim with test-pod.yaml  
   1. ```kubectl apply -f test-pod.yaml```  
   2. check the test-pod  
      + if you see tes-pod is a pending status, then there is a problem with server connection, configuration to NFS client, or not enough storage space.  
      ```kubectl get pod -n monitoring```    
      + if you see test-pod is completed, then you are ready to use nfs PVC.   
      ```kubectl get pod -n monitoring --show-all```       
   3. check the NFS-SUCCESS file created by test-pod.yaml  
      + ```sudo mount -t nfs 10.86.1.244:/var/nfs /mnt```   
      + you should see new file name NFS-SUCCESS in ```ls -al /mnt```   
      + ```sudo rm /mnt/NFS-SUCCESS```
      + ```sudo umount /mnt```   
#### 5. Create a Prometheus using helm with prometheus-config.yaml
```helm install stable/prometheus --namespace monitoring --name prometheus --values prometheus-config.yaml```
#### 6. Check if everything is deployed well
```kubectl -n monitoring get po | grep prometheus```
#### 7. Check the promethus web using the follwoing url and port
```
export NODE_PORT=$(kubectl get --namespace monitoring -o jsonpath="{.spec.ports[0].nodePort}" services prometheus-server)
export NODE_IP=$(kubectl get nodes --namespace monitoring -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT
```
#### 7-1. On Prometheus from url #7, you can query to Prometheus using Promethues web (optional).
#### 8. Replace ```<prometheus.server.ip.address>:<port>``` into ```<url>:<port>``` from #7 in prometheus-grafana-datasource.yaml 
      url: http://<prometheus.server.ip.address>:<port>
#### 9. Create a configMap for mapping between Prometheus and Grafana
```kubectl create -f prometheus-grafana-datasource.yaml``` 
#### 10. Create Grafana using helm with grafana-config.yaml
```helm install stable/grafana --namespace monitoring --name grafana --values grafana-config.yaml``` 
#### 11. grafana will take up to 10 min if Grafana has a persistent option. check if all the three pods are deployed.
```kubectl -n monitoring get po | grep grafana```
#### 12. Login to grafana web
   * Get Grafana url from the following:  
     ~~~
      export NODE_PORT=$(kubectl get -o jsonpath="{.spec.ports[0].nodePort}" services grafana -n monitoring)
      export NODE_IP=$(kubectl get nodes -o jsonpath="{.items[0].status.addresses[0].address}")
      echo http://$NODE_IP:$NODE_PORT/
     ~~~
   * Login to grafana web with admin/admin (default username/password), enter new password.   
#### 13. Add a new dashboard in the garfana web - "Kubernetes All Nodes" will be shown "System load" panel only.
```
    1. hover your mousecursor over the + button on the left sidebar and click on the importmenuitem.
    2. Paste the URL (for example) https://grafana.com/dashboards/3131 into the first input field to import the "Kubernetes All Nodes" Grafana Dashboard. After pasting in the url, the view will change to another form.
    3. Now select the "Prometheus" datasource in the prometheus field and click on the import button.
```
#### 14. There are extra dashboards configured for CAASP. To appply this
```
    1. git clone https://github.com/kubic-project/monitoring.git
    2. kubectl apply -f monitoring/grafana-dashboards-caasp-cluster.yaml
    3. kubectl apply -f monitoring/grafana-dashboards-caasp-namespaces.yaml
    4. kubectl apply -f monitoring/grafana-dashboards-caasp-nodes.yaml
    5. kubectl apply -f monitoring/grafana-dashboards-caasp-pods.yaml
```
#### 15. All the configured dashboards will be shown in grafana 

