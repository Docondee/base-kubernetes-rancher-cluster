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

## Setup Local Storage

### Installation and Testing of Local-Path Storage Provisioner in Rancher

Based on the troubleshooting, the official manifest from the local-path-provisioner GitHub repo (v0.0.32, stable as of September 2025) deploys correctly as a Deployment (for single-node local setups like your RKE2). It creates the Namespace, RBAC, ConfigMap, StorageClass ("local-path"), and Deployment. The WaitForFirstConsumer mode is intentional for node-local PVs, so PVCs bind only when a pod mounts them—your test succeeded once the pod was created.

Here's the full, verified process (run with kubeconfig set: `export KUBECONFIG=~/.kube/config`). This takes ~2-3 minutes.

#### Step 1: Clean Up (If Retrying)
```
kubectl delete ns local-path-storage --ignore-not-found=true
kubectl delete sc local-path --ignore-not-found=true
```

#### Step 2: Deploy the Official Manifest
```
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.32/deploy/local-path-storage.yaml
```
- **What It Does**:
  - Namespace: `local-path-storage`.
  - ServiceAccount: `local-path-provisioner-service-account`.
  - RBAC: Role, ClusterRole, RoleBinding, ClusterRoleBinding (`local-path-provisioner-role/bind`).
  - ConfigMap: `local-path-config` (config for provisioner).
  - StorageClass: `local-path` (provisioner `rancher.io/local-path`, reclaim=Delete, binding=WaitForFirstConsumer, no expansion).
  - Deployment: `local-path-provisioner` (image `rancher/local-path-provisioner:v0.0.32`, args `--debug start --service-account-name=local-path-provisioner-service-account`, hostPath `/opt/local-path-provisioner`).

#### Step 3: Verify Deployment
1. **Rollout Status** (Use Deployment, Not DaemonSet):
   ```
   kubectl -n local-path-storage rollout status deployment/local-path-provisioner
   ```
   - Expected: "deployment 'local-path-provisioner' successfully rolled out".

2. **Pods and StorageClass**:
   ```
   kubectl get pods -n local-path-storage  # Pod "Running" (1/1 READY)
   kubectl get sc local-path  # Shows "local-path" with PROVISIONER "rancher.io/local-path"
   ```

3. **Logs** (Check for Errors):
   ```
   kubectl logs -n local-path-storage deployment/local-path-provisioner
   ```
   - Expected: "Starting provisioner" and debug output (no panics). If permission errors (e.g., SELinux), run `sudo mkdir -p /opt/local-path-provisioner && sudo chmod 755 /opt/local-path-provisioner` on the node, then restart: `kubectl rollout restart deployment/local-path-provisioner -n local-path-storage`.

#### Step 4: Test PVC Provisioning
1. **Create Test PVC**:
   ```
   kubectl apply -f - <<EOF
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: test-pvc
   spec:
     accessModes: [ReadWriteOnce]
     resources:
       requests:
         storage: 1Gi
     storageClassName: local-path
   EOF
   ```
   - Status: "Pending" initially (waiting for consumer).

2. **Trigger Binding with Test Pod**:
   ```
   kubectl apply -f - <<EOF
   apiVersion: v1
   kind: Pod
   metadata:
     name: test-pod
   spec:
     containers:
     - name: test-container
       image: busybox
       command: ["/bin/sh", "-c", "sleep 3600"]
       volumeMounts:
       - mountPath: /test-data
         name: test-volume
     volumes:
     - name: test-volume
       persistentVolumeClaim:
         claimName: test-pvc
   EOF
   ```
   - Wait 10-30s for scheduling.

3. **Check Binding**:
   ```
   kubectl get pvc test-pvc  # STATUS: "Bound" with VOLUME (e.g., pvc-uuid)
   kubectl get pod test-pod  # "Running"
   kubectl describe pvc test-pvc  # Events: "provisioning succeeded"
   ```
   - Test Mount: `kubectl exec test-pod -- touch /test-data/test-file && kubectl exec test-pod -- ls /test-data` (shows file on local PV).

4. **Cleanup**:
   ```
   kubectl delete pod test-pod
   kubectl delete pvc test-pvc
   ```

#### Notes
- **Local-Path Behavior**: PVs are node-local (`/opt/local-path-provisioner/<uuid>` on the pod's node)—great for dev/single-node RKE2, but not shared across nodes. For multi-node shared storage, install Longhorn via Rancher Apps.
- **Uninstall**: `kubectl delete -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.32/deploy/local-path-storage.yaml`.

Your storage is now ready—proceed to Jenkins install with `jenkins-values.yaml` (`controller.persistence.storageClass: local-path`). The Jenkins PVC will bind when the Deployment pod starts. If any errors, share `kubectl describe pvc <name>`.hen the Deployment pod starts. If any errors, share `kubectl describe pvc <name>`.
