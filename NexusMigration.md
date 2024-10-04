# Nexus Upgrade and Data Migration Guide

This guide walks through the process of upgrading Nexus from an older version (Nexus 3.7.0) to a newer version (Nexus 3.7.2) on a Kubernetes cluster. We will create backups, perform data migration, and switch traffic from the old version to the new version while maintaining data integrity.


## Table of Contents Prerequisitews Steps
- Create Storage Class and Namespace
-  Create Persistent Volumes and Claims
-  Deploy Nexus Versions
-  Create Backup in the Old Nexus Version
-  Copy Backup Files to the Host Machine
-  Clean Up Unnecessary Files
-  Copy Data to the New Nexus Version
-  Restart the New Version
-  Switch Traffic to the New Version
    
    Clean Up & YAML Files
## Prerequisites

Ensure the following prerequisites are met before proceeding with the upgrade and migration:

Ensure the following prerequisites are met:
- A running Kubernetes cluster with multiple worker nodes.
- Access to deploy resources such as Persistent Volumes (PVs) and Persistent Volume Claims (PVCs).
- Two Nexus versions (old: 3.7.0, new: 3.7.2) deployed on separate worker nodes.
- A Storage Class for Nexus PVCs is created.


## Deployment

#### 1. Create Storage Class and Namespace


```bash
  kubectl apply -f <storage-class-file.yaml>

  kubectl create namespace nexus
```

#### 2. Create Persistent Volumes and Claims
Create Persistent Volumes (PV) and Persistent Volume Claims (PVC) for both Nexus versions using the same Storage Class:

```bash
  kubectl apply -f <nexus-pv-pvc-old.yaml>

  kubectl apply -f <nexus-pv-pvc-new.yaml>
```

#### 3. Deploy Nexus Versions
Deploy Nexus 3.70.1 on worker node 1:

```bash
  kubectl apply -f <nexus-old-deployment.yaml>
```


Deploy Nexus 3.72.0 on worker node 2:

```bash
  kubectl apply -f <nexus-new-deployment.yaml>
```

- Exposing Nexus Service Using LoadBalancer (We are using MatelLB)
```bash
  kubectl apply -f <service-expose.yaml>
```

#### 4. Create a Backup in Nexus Admin UI using create Task

- Access Nexus in your browser and log in with your admin credentials.
- Click the gear icon (⚙️) on the left sidebar to access Administration.
- In the Tasks section under System, click Create Task. 
- Name the task (e.g., Nexus Backup Task).
- Set the Backup Location where the backup files will be stored.
- Choose Manual or a recurring schedule for the task.
- Click Create, and then Run the task to start the backup process.
- Check the task log to confirm it completed successfully.
- Verify the backup files in the specified location.


#### 5. Download and Copy DB Migrator to Old Nexus Pod
Using specifc version for nexus 3.70.1 because it uses orientDB.

[Download Link](https://sonatype-download.global.ssl.fastly.net/repository/downloads-prod-group/nxrm3-migrator/nexus-db-migrator-3.70.2-01.jar)

```bash
curl -o <filename.jar> <URL>
```
###### 1. Download the nexus370.jar (DB Migrator) file.
###### 2. Copy the nexus370.jar file into the old Nexus pod at /nexus-data/:

```bash
kubectl cp -n nexus nexus370.jar <nexus3.70.1-podname>:/nexus-data/
```

#### 6. Create Backup Using a Backup Pod
Apply the YAML file to deploy the backup pod:

```bash
kubectl apply -f <backup-pod/migration.yaml>
```

#### 6. Check logs if migration is successfull 
Will see output like this
```
  13:16:47 [main] INFO  o.s.batch.core.job.SimpleStepHandler - Executing step: [resetTableSequencesStep]
  13:16:47 [main] INFO  o.s.batch.core.step.AbstractStep - Step: [resetTableSequencesStep] executed in 30ms
  13:16:47 [main] INFO  o.s.batch.core.job.SimpleStepHandler - Executing step: [finalDatabaseStep]
  13:16:47 [main] INFO  o.s.batch.core.step.AbstractStep - Step: [finalDatabaseStep] executed in 1ms
  13:16:47 [main] INFO  c.s.n.d.m.l.ProvidingJobInfoListener - Migration job finished at Fri Oct 04 13:16:47 GMT 2024
  13:16:47 [main] INFO  c.s.n.d.m.l.ProvidingJobInfoListener - Migration job took 2 seconds to execute
  13:16:47 [main] INFO  c.s.n.d.m.l.ProvidingJobInfoListener - 81 records were processed
  13:16:47 [main] INFO  c.s.n.d.m.l.ProvidingJobInfoListener - 5 records were filtered
  13:16:47 [main] INFO  c.s.n.d.m.l.ProvidingJobInfoListener - 0 records were skipped
  13:16:47 [main] INFO  c.s.n.d.m.l.ProvidingJobInfoListener - 76 records were migrated
  13:16:47 [main] INFO  c.s.n.d.m.l.ProvidingJobInfoListener - Created 'Rebuild repository browse' and 'Rebuild repository search' tasks. They will automatically one-time run after starting your Nexus Repository instance.
  13:16:47 [main] INFO  c.s.n.d.m.l.ProvidingJobInfoListener - Migrated the following formats: [APT, COCOAPODS, CONAN, CONDA, DOCKER, GITLFS, GO, HELM, MAVEN2, NPM, NUGET, P2, PYPI, R, RAW, RUBYGEMS, YUM]
```


#### 7. Copy Data from Backup Pod to Host Machine
Copy the migration data from the backup pod to the host machine using kubectl cp:

```bash
kubectl cp -n nexus <pod-backup>:/nexus370/migration .
```

#### 8. Clean Up Unnecessary Files
Remove any .bak files from the backup folder on the host machine:

```
files_to_remove=$(find . -mindepth 1 -name '*.bak') &&

if [ -n "$files_to_remove" ]; then
    echo "Removing .bak files: $files_to_remove"
    rm -rf $files_to_remove
fi
```

#### 8. Copy Data to the New Nexus Version

```bash
kubectl cp -n nexus . <new-version-nexus-pod>:/nexus-data/db
```

#### 9. Restart the New Version
Restart the new Nexus deployment:

```
kubectl rollout restart deployment -n nexus <deployment-name>
```

Allow one minute for the new Nexus version to stabilize.

#### 10. Switch Traffic to the New Version
Switch traffic to the new Nexus version by updating the service selector:

```
kubectl patch service nexus3-matallb-service -n nexus --type='json' -p='[{"op": "replace", "path": "/spec/selector/app", "value": "<new-version-deployment-name>"}]'
```




## Clean Up

- Remove the backup pod that we used for migration
- Remove the nexus 3.70.1 version if required (Optional)