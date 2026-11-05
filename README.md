# GitOps with Monitoring

GitOps on AWS EKS: Terraform provisions the cluster, ArgoCD deploys from Git, and Prometheus + Grafana monitor it. Includes a small Node.js web app as the demo workload.

## Stack

AWS EKS · Terraform · ArgoCD · Prometheus & Grafana (Helm) · Kubernetes · Docker · Node.js

## Repository layout

```
app/        Node.js web app + Dockerfile
manifests/  Kubernetes Deployment and Service for the app
argocd/     ArgoCD Applications (webapp, monitoring) and ServiceMonitors
terraform/  VPC and EKS cluster
```

## Prerequisites

- AWS account and credentials configured (`aws configure`)
- Docker Hub account
- Installed: `docker`, `terraform`, `aws` CLI, `kubectl`
- A fork or copy of this repository (ArgoCD syncs from your Git repo)

## Deploy

### 1. Build and push the image

```bash
cd app
docker build -t <your-dockerhub-username>/gitops-webapp:latest .
docker push <your-dockerhub-username>/gitops-webapp:latest
```

Set the same image in `manifests/deployment.yaml`, and make sure `repoURL` in `argocd/app.yaml` points to your repository. Commit and push.

### 2. Create the cluster

```bash
cd terraform
terraform init
terraform apply
aws eks update-kubeconfig --name gitops-with-monitoring --region us-east-1
kubectl get nodes
```

This creates a VPC and an EKS cluster (Kubernetes 1.32) with 2 `t3.medium` nodes. It takes about 15-20 minutes.

### 3. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Get the URL and the admin password:

```bash
kubectl get svc argocd-server -n argocd
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Open `https://<argocd-loadbalancer>` and log in as `admin`. A certificate warning is expected.

### 4. Deploy the apps

```bash
kubectl apply -f argocd/app.yaml
kubectl apply -f argocd/monitoring.yaml
kubectl apply -f argocd/service-monitor.yaml
kubectl get pods -n webapp
kubectl get pods -n monitoring
```

Wait until both applications show Healthy and Synced in ArgoCD.

### 5. Open the apps

```bash
kubectl get svc -n webapp                          # web app
kubectl get svc -n monitoring monitoring-grafana   # Grafana
```

Grafana login: `admin` / `admin123` (set in `argocd/monitoring.yaml`; change it for anything beyond a demo).

In Grafana, go to Dashboards → New → Import and load ID `14584` or `19993` for ArgoCD metrics. The built-in "Kubernetes / Compute Resources / Cluster" dashboard shows cluster usage.

## Cleanup

Delete the LoadBalancers first so AWS can remove the VPC, then destroy the infrastructure:

```bash
kubectl delete svc argocd-server -n argocd
kubectl delete svc -n webapp --all
kubectl patch svc monitoring-grafana -n monitoring -p '{"spec": {"type": "ClusterIP"}}'
cd terraform
terraform destroy
```

If `terraform destroy` hangs, delete any leftover load balancers in the AWS Console (EC2 → Load Balancers) and retry.

## Troubleshooting

- **App not syncing:** check that `repoURL` and `path` in `argocd/app.yaml` match your repository.
- **Pods in `CrashLoopBackOff` or `ImagePullBackOff`:** run `kubectl describe pod <pod> -n <namespace>` and check the image name in `manifests/deployment.yaml`.
- **LoadBalancer URL not reachable:** wait 2-3 minutes for provisioning.

## Notes

- Running this on AWS costs money (roughly $2-3/day). Clean up when done.
- This is a demo setup, not production-hardened.

## Author

**Shyama7004** — [GitHub](https://github.com/Shyama7004)
