# Monitoring Prometheus and Grafana with NFS persistentVolumes

*Full Docs : https://w3.nue.suse.com/~mnapp/2018-11-19/book.caasp.admin/cha.admin.monitoring.html. This version is for a slim version of the original deployment. This does not require any storage and secret. This version also disabled alertManager in Prometheus. NFS server is 10.86.1.244:/var/nfs which is behind SUSE VPN.*

#### 1. Create a new namespace for monitoring 
```kubectl create namespace monitoring```
#### 2. Create pv-pvc-monitoring.yaml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
  labels:
    pv-name: nf-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteMany
  nfs:
    server: 10.86.1.244
    path: /var/nfs
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    pv.kubernetes.io/bind-completed: "yes"
    pv.kubernetes.io/bound-by-controller: "yes"
    volume.beta.kubernetes.io/storage-class: ssd
    volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/nfs
  name: nfs-pvc
  namespace: monitoring
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  volumeName: nfs-pv
status:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 8Gi
  phase: Bound
```
#### 3. Create PV and PVC  
```kubectl apply -f pv-pvc-monitoring.yaml```
#### 4. Verify the PV and PVC  
+ ```kubectl get pv```  
+ ```kubectl get pvc -n monitoring```  
#### 5. Testing persistentVolumeClaim with test-pod.yaml  
   1. create test-pod.yaml file
   ~~~
    kind: Pod
    apiVersion: v1
    metadata:
      name: test-pod
      namespace: monitoring
    spec:
      containers:
      - name: test-pod
        image: gcr.io/google_containers/busybox:1.24
        command:
          - "/bin/sh"
        args:
          - "-c"
          - "touch /mnt/NFS-SUCCESS && exit 0 || exit 1"
        volumeMounts:
          - name: vnfs
            mountPath: /mnt
      restartPolicy: "Never"
      volumes:
        - name: vnfs
          persistentVolumeClaim:
            claimName: nfs-pvc
   ~~~
   2. ```kubeclt apply -f test-pod.yaml```  
   3. check the test-pod  
      + if you see tes-pod is pending status, then there is a problem with server connection or configuration to NFS.  
      ```kubectl get pod -n monitoring```    
      + if you see test-pod is completed, then you are ready to use nfs PVC.   
      ```kubectl get pod -n monitoring --show-all```       
   4. check the NFS-SUCCESS file created by test-pod.yaml  
      + ```sudo mount -t nfs 10.86.1.244:/var/nfs /mnt```   
      + you should see new file name NFS-SUCCESS in ```ls -al /mnt```   
      + ```umount /mnt```   
#### 6. Create prometheus-config.yaml
```
# Alertmanager configuration
alertmanager:
  enabled: true
  ingress:
    enabled: false
  persistentVolume:
    enabled: true
    existingClaim: nfs-pvc

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
    enabled: true
    existingClaim: nfs-pvc
```

#### 7. Create a Prometheus using helm with prometheus-config.yaml
```helm install stable/prometheus --namespace monitoring --name prometheus --values prometheus-config.yaml```
#### 8. Check if everything is deployed well
```kubectl -n monitoring get po | grep prometheus```
#### 9. Check the promethus web using the follwoing url and port
```
export NODE_PORT=$(kubectl get --namespace monitoring -o jsonpath="{.spec.ports[0].nodePort}" services prometheus-server)
export NODE_IP=$(kubectl get nodes --namespace monitoring -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT
```
#### 9-1. On Prometheus from url #5, you can query to Prometheus (optional).
#### 10. Create prometheus-grafana-datasource.yaml using url:port from #5
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
#### 11. Create a configMap for mapping between Prometheus and Grafana
```kubectl create -f prometheus-grafana-datasource.yaml``` 
#### 12. Create grafana-config.yaml
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
#### 13. Create Grafana using helm with grafana-config.yaml
```helm install stable/grafana --namespace monitoring --name grafana --values grafana-config.yaml``` 
#### 14. grafana will take up to 10 min. check if all the three pods are deployed.
```kubectl -n monitoring get po | grep grafana```
#### 15. Login to grafana-server with username/password from grafana-config.yaml
```
export NODE_PORT=$(kubectl get -o jsonpath="{.spec.ports[0].nodePort}" services grafana -n monitoring)
export NODE_IP=$(kubectl get nodes -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT/
```
#### 16. Add a new dashboard in the garfana web - "Kubernetes All Nodes" will be shown "System load" panel only.
```
    1. On the Grafana from url #11 with admin/admin (username/password), enter new password.
    2. hover your mousecursor over the + button on the left sidebar and click on the importmenuitem.
    3. Paste the URL (for example) https://grafana.com/dashboards/3131 into the first input field to import the "Kubernetes All Nodes" Grafana Dashboard. After pasting in the url, the view will change to another form.
    4. Now select the "Prometheus" datasource in the prometheus field and click on the import button.
```
#### 17. There are extra dashboards configured for CAASP. To appply this
```
    1. git clone https://github.com/kubic-project/monitoring.git
    2. kubectl apply -f monitoring/grafana-dashboards-caasp-cluster.yaml
    3. kubectl apply -f monitoring/grafana-dashboards-caasp-namespaces.yaml
    4. kubectl apply -f monitoring/grafana-dashboards-caasp-nodes.yaml
    5. kubectl apply -f monitoring/grafana-dashboards-caasp-pods.yaml
```
#### 18. All the configured dashboards will be shown in grafana 

