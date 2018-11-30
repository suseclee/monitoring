# Caasp-dev-Monitor
CaaSP-dev-monitor provides a Helm chart for deploying Prometheus, Grafana, and Caasp Dashboards.
It deploys both of Prometheus and Grafana without persistVolumes and configures Caasp Dashboards (https://github.com/kubic-project/monitoring). Promemeus and Grafana's NodePorts are fixed relatively to 31313 and 31314.

#### 1. Create a new namespace for monitoring 
```kubectl create namespace monitoring```
#### 2. git clone for a local monitor chart
```git clone https://github.com/suseclee/monitoring.git```
#### 3. Move to local chart repo 
```cd monitoring/charts/caasp-dev-monitor```
#### 4. Install Prometheus and Grafana in the monitor chart
```helm install  . --namespace monitoring --name monitor```
#### 5. Check if Prometheus and Grafana are deployed well. It can take up to 10 min. 
```kubectl -n monitoring get po | grep prometheus```   
```kubectl -n monitoring get po | grep grafana```
#### 6-1. you can try queries in Prometheus web: 10.17.2.0:31313 (optional).
#### 7. Login to grafana web: 10.17.2.0:31314
   * Login to grafana web with admin/admin (default username/password), enter new password.   
#### 8. All the configured dashboards will be shown in Grafana 
