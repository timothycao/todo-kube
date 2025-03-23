# todo-kube

This project packages a pre-built Flask + MongoDB todo app into Docker containers and deploys it to Kubernetes. It supports both local deployment with Minikube and cloud deployment on AWS EKS.

## Features

- **Dockerized Flask and MongoDB** with `Dockerfile` and `docker-compose.yml`
- **Kubernetes deployments** using manifests for pods, services, and persistent storage
- **Persistent storage** with dynamic volume provisioning:
  - Local PVC on Minikube with default StorageClass (`standard`)
  - EBS-backed PVC on EKS with custom StorageClass (`gp2`)
- **Zero-downtime rolling updates** using `RollingUpdate` strategy
- **Liveness and readiness probes** with a failure toggle (`FLASK_FAILURE`) for testing
- **Prometheus monitoring and Slack alerting** using `kube-prometheus-stack` Helm chart

## Setup Overview

### 1. Build and Push Docker Image

```bash
docker build -t todo-kube .
# docker build --platform linux/amd64 -t todo-kube .  # Required for EKS if building on Apple Silicon
docker tag todo-kube <dockerhub-username>/todo-kube:latest
docker push <dockerhub-username>/todo-kube:latest
```

### 2. Deployment

```bash
# Minikube
minikube start  # Create Minikube cluster and connect to it
kubectl apply -f kube/namespace.yaml  # Create todo-kube and monitoring namespaces
kubectl apply -f kube/ --prune --selector 'app!=mongo-storage-class'  # Skip EKS-only StorageClass
minikube service flask-service --url  # Expose service and get access URL

# EKS
aws eks update-kubeconfig --region <aws-region> --name <eks-cluster-name>  # Connect to EKS cluster
kubectl apply -f kube/namespace.yaml  # Create todo-kube and monitoring namespaces
kubectl apply -f kube/
```

### 3. Monitoring & Alerts

> Update the Slack `api_url` and `channel` in `alertmanager-values.yaml`.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values helm/alertmanager-values.yaml  # Apply value override file
```

### 4. UI Access (Port Forwarding)

```bash
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring # Grafana UI
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090 -n monitoring  # Prometheus UI
kubectl port-forward svc/prometheus-kube-prometheus-alertmanager 9093:9093 -n monitoring  # Alertmanager UI
```

## Cleanup Commands

```bash
# Minikube
minikube delete

# EKS
kubectl delete all --all -n todo-kube
kubectl delete pvc --all -n todo-kube

# Prometheus
helm uninstall prometheus -n monitoring

# Namespaces
kubectl delete namespace todo-kube
kubectl delete namespace monitoring
```

## Notes

- All application resources run in the `todo-kube` namespace
- All monitoring resources run in the `monitoring` namespace
- Namespace definitions are applied using `namespace.yaml`
- Slack alerts are triggered on liveness and readiness probe failures