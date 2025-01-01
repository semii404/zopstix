
# Cluster upgrade guide
Updated Kubernetes Cluster Upgrade Guide ( Ubuntu )
This guide has been updated to optimize the repository configuration steps. When upgrading Kubernetes versions incrementally, only the repository URL in the kubernetes.list file needs to be updated after the initial GPG key setup.

### Step-by-Step Upgrade Process
#### 1. Initial Setup
For the first upgrade, configure the repository GPG key:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
Add the repository URL for the current version:

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.25/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list sudo apt update
```
 
#### 2. Upgrade Kubernetes Components
Note: After installing or upgrading kubeadm, it's crucial to verify the version and ensure the correct binary is being used. This will confirm that the installation or upgrade was successful and that your system is running the expected version.
 
Locate the kubeadm Binary: Use the which command to identify the location of the kubeadm binary currently in use:
Ensure the Correct kubeadm Path:
By default, the kubeadm binary should be located in /usr/bin/.
```bash
which kubeadm
```
If it's not found there, copy it manually:
```bash
sudo cp /usr/bin/kubeadm /usr/local/bin/
``` 
Check available versions for kubeadm:

```bash
sudo apt-cache madison kubeadm
```
Install the required version (e.g., 1.25.x):
```bash
sudo apt-get install -y kubeadm='1.25.3-*'
```
Plan the upgrade:
```bash
sudo kubeadm upgrade plan
```
Apply the upgrade:
```bash
sudo kubeadm upgrade apply v1.25.3
```
Upgrade kubelet and kubectl:
```bash
sudo apt-get install -y kubelet='1.25.3-*' kubectl='1.25.3-*' 
sudo systemctl daemon-reload sudo systemctl restart kubelet.service
```

Verify cluster state:
```bash
kubectl get nodes
``` 
 
#### 3. Upgrade to the Next Version
For subsequent upgrades, only update the repository URL to point to the next version. There's no need to update the GPG key again.
For example, to upgrade from v1.25.x to v1.26.x:
```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.26/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list 
sudo apt update
```
Then, repeat the steps for installing kubeadm, planning and applying the upgrade, and updating kubelet and kubectl.
 
 
### Troubleshooting
#### Issue: kubelet Service Down (v1.24 → v1.25)
Cause: Deprecated flags in /etc/default/kubelet.
Solution:
Remove unrecognized arguments:
```bash
sudo nano /etc/default/kubelet
```
Reload and restart kubelet:
```bash
sudo systemctl daemon-reload 
sudo systemctl restart kubelet.service
``` 
Issue: Pods Crash During Upgrade
Cause: Inconsistent kubelet state.
Solution: Reload and restart kubelet:
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet.service
``` 
Notes
After the initial GPG key configuration, simply update the kubernetes.list file for subsequent versions.

Ensure the target versions for kubeadm, kubelet, and kubectl match during each upgrade step.

Incremental upgrades help maintain compatibility and reduce the risk of breaking changes.
