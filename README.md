# Setup Rancher Kubernetes Engine Cluster 2

## Prequisites

Make sure you have no local Kubernetes API server installed

## Install Fresh Rancher Kubernetes Engine 2

```bash
curl -sfL https://get.rke2.io | sh -
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service
```

Monitor the installatin with `sudo journalctl -u rke2-server -f`

## Setup Kubeconfig Locally

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
export KUBECONFIG=~/.kube/config
```

## Verify that the Nodes are Up and Runnng

```bash
kubectl get nodes
```

## Install Cert-Manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.2/cert-manager.crds.yaml
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.18.2 \
  --set installCRDs=false

kubectl get pods -n cert-manager
```

## Install Rancher GUI

```bash
ip addr show
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
kubectl create namespace cattle-system
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=<YOUR_IP>.sslip.io \
  --set replicas=1 \
  --set bootstrapPassword=admin123  # Use a strong password
  --version 2.12.2

kubectl -n cattle-system rollout status deploy/rancher
kubectl -n cattle-system get pods.
```

## Access the Rancher WEB UI

- URL: `https://<YOUR_IP>.sslip.io` (accept self-signed cert).
- Login: `admin` / `admin123` (change immediately).
- UI will auto-import your RKE2 cluster for management.
