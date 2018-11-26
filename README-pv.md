# Monitoring Prometheus and Grafana with NFS persistentVolumes

*Full Docs : https://w3.nue.suse.com/~mnapp/2018-11-19/book.caasp.admin/cha.admin.monitoring.html. This version is for a slim version of the original deployment. This does not require any storage and secret. This version also disabled alertManager in Prometheus. NFS server is 10.86.1.244:/var/nfs which is behind SUSE VPN.*

1. Create pv-pvc-monitoring.yaml
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
2. Create PV and PVC
```kubectl applt -f pv-pvc-monitoring.yaml```
3. Verify the PV and PVC
```kubectl get pv```
```kubectl get pvc -n monitoring```
4. Testing persistentVolumeClaim with test-pod.yaml
    1. create test-pof.yaml file
    ```
    kind: Pod
apiVersion: v1
metadata:
  name: test-pod
  #  namespace: kube-system
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
        claimName: prometheus-prometheus-k8s-db-prometheus-prometheus-k8s-0
     ``` 
   
