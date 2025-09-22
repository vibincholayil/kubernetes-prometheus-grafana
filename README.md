# Use-case: Kubernetes Monitoring with Prometheus & Grafana
Set up a monitoring stack on Kubernetes using Prometheus and Grafana. I will demostrate, how to monitor CPU, Memory, and Disk usage of Linux/Windows nodes, visualize them in Grafana, configure variables for filtering, and set alerts for thresholds.  

### Structure
k8s-monitoring-project/
│── terraform/                # Optional: Use Terraform to provision cluster (EKS/AKS/Minikube)
│── helm/                     # Helm charts & values
│   ├── prometheus-values.yaml
│   ├── grafana-values.yaml
│── manifests/                # K8s manifests for exporters/configs
│   ├── node-exporter-daemonset.yaml
│   ├── windows-exporter-daemonset.yaml
│   ├── configmap-scrape.yaml
│── dashboards/               # JSON dashboards (import into Grafana)
│   ├── linux-dashboard.json
│   ├── windows-dashboard.json
│── alerts/                   # Example alert rules
│   ├── cpu-alert.yaml
│   ├── memory-alert.yaml
│── README.md                 # Documentation

### Steps
#### 1. Provision Kubernetes Cluster

environment: Minikube (local testing) and AKS (Azure)  

(Minikube): minikube start --memory=4096 --cpus=2
kubectl get nodes

#### 2. Install Prometheus & Grafana (Helm)

Add Helm repo & update:

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update


Install stack with custom values:

helm upgrade --install monitor prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f helm/prometheus-values.yaml


Expose Grafana & Prometheus with NodePort:

# grafana-values.yaml
service:
  type: NodePort
  nodePort: 32000


Check services:

kubectl get svc -n monitoring

3. Deploy Exporters

Linux Node Exporter (runs as DaemonSet):

# node-exporter-daemonset.yaml
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter
        ports:
        - containerPort: 9100
```

Windows Exporter runs similarly (port 9182).

Apply:

kubectl apply -f manifests/node-exporter-daemonset.yaml

4. Configure Scrape Targets (ConfigMap)

Instead of editing values.yaml, use a ConfigMap:

# configmap-scrape.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-additional-scrape-configs
  namespace: monitoring
data:
  web.json: |
    - job_name: 'web'
      static_configs:
        - targets: ['node1:9100','node2:9100']
```
5. Create Grafana Dashboards

Import from dashboards/linux-dashboard.json or use PromQL queries:

CPU Usage Query

100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)


Memory Usage Query

node_memory_Active_bytes / node_memory_MemTotal_bytes * 100

6. Add Variables & Alerts

Variable Example (instance filter):

label_values(node_cpu_seconds_total, instance)


Alert Example (CPU > 80%):

# cpu-alert.yaml
```
groups:
- name: cpu-alerts
  rules:
  - alert: HighCPUUsage
    expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "High CPU usage on {{ $labels.instance }}"
      description: "CPU usage is above 80% for 1m"
```
7. (Optional) SMTP for Email Alerts

Create Kubernetes Secret for SMTP:

apiVersion: v1
kind: Secret
metadata:
  name: smtp-secret
  namespace: monitoring
type: Opaque
stringData:
  smtp_user: "your@email.com"
  smtp_pass: "yourpassword"


Configure Grafana to use it under Contact Points.

✅ Deliverables

Prometheus & Grafana deployed on Kubernetes.

Node Exporter (Linux) & Windows Exporter collecting metrics.

Grafana dashboards showing CPU + Memory utilization.

Variable dropdowns to filter servers.

Alerts for CPU > 80%, Memory > 75%.

Optional: Email/SMS alerts via SMTP.
