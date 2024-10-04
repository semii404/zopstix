
# Deploying MariaDB using FluxCD

This guide walks through deploying MariaDB in Kubernetes using FluxCD for continuous deployment. It covers setting up FluxCD, bootstrapping it with a GitHub repository, and deploying MariaDB with GitOps.

## Prerequisites

- Kubernetes Cluster – A running Kubernetes cluster.

- kubectl – Installed and configured with your cluster.

- Flux CLI – For managing FluxCD operations.

- GitHub Account – A personal access token (PAT) for authentication

## Install Flux CD

### First SSH into the Master Node then follow the below steps.

###### 1. Install the Flux CLI by running the following command and verify the Installation

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
flux --version
```

###### 2. Pre-checks for Flux Installation in Kubernetes Cluster
Before bootstrapping Flux, ensure all necessary pre-checks pass:

```
flux check --pre
```

###### 3. Export GitHub Credentials
Need to export your GitHub credentials (Personal Access Token and username) as environment variables for Flux to authenticate with your repository.


```
export GITHUB_TOKEN=<your-github-pat>
export GITHUB_USER=<your-github-username>
```

###### 4. Edit CoreDNS ConfigMap (Optional)

If your pods face internet connectivity issues or if your nodes have multiple nameservers, you may need to modify the CoreDNS ConfigMap to adjust the DNS settings:

```
kubectl edit configmap coredns -n kube-system
```

###### 5. Bootstrap FluxCD
Bootstrap Flux to connect your Kubernetes cluster to your GitHub repository. This creates the necessary resources in your cluster and links them to a GitHub repository that Flux will monitor for changes.

```
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=<repository-name> \
  --branch=master \
  --path=clusters/dev/<cluster-name> \
  --personal
  ```

##### Note: Create Namespace, Create PersistentVolume (PV) and PersistentVolumeClaim (PVC)

```
kubectl create namespace database
```

For PV and PVC the yaml file available in "fluxCD_mariaDB" Directory.


###### 6. Clone Repository Locally
###### 7. Create MariaDB Source in Flux using Flux CLI
To deploy MariaDB, first create a Git source pointing to the repository where the MariaDB manifests are stored. This instructs Flux to sync with the repository.

```
flux create source git infra-mariadb \
  --url=https://github.com/$GITHUB_USER/<repository-name> \
  --branch=<branch> \
  --interval=30s \
  --export > clusters/dev/<folder-name>/infra-mariadb.yaml
```

The file infra-mariadb.yaml defines the source of the MariaDB manifests and sets the sync interval to 30 seconds.

###### 8. Create a Kustomization for MariaDB
Now, create a Kustomization to apply the MariaDB manifests from the specified source.

```
flux create kustomization k-infra-mariadb \
  --target-namespace=flux-system \
  --source=infra-mariadb \ # source name you can see above after command "flux create source git" in step 7
  --path="apps/infra-mariadb" \
  --prune=true \
  --interval=5m \
  --export > clusters/dev/<folder-name>/k-infra-mariadb.yaml
```

The kustomization.yaml defines how Flux should apply the manifests in the specified path, prune any outdated resources, and sync every 5 minutes.


###### 9. Commit and Push Changes

After making the changes to the repository (adding the infra-mariadb.yaml and kustomization files), commit and push the updates to GitHub:

```
git add .
git commit -m "Add MariaDB deployment with FluxCD"
git push origin master
```



Flux will automatically sync the changes and deploy MariaDB to your Kubernetes cluster.