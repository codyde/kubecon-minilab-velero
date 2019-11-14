---
description: >-
    Velero (formerly known as Heptio Ark) gives users tools to back up and restore Kubernetes cluster resources and persistent volumes. 
---

# Project Velero: Introduction

Velero (formerly known as Heptio Ark) gives users tools to back up and restore Kubernetes cluster resources and persistent volumes. Velero can be used both in public cloud as well as with on-premises Kubernetes clusters. Backups can be scheduled on demand or scheduled. Once these backups are obtained, cluster operators can restore these backups to their existing cluster as a recovery tool, or other clusters as a migration (or recovery) tool.

Velero consists of a local server instance that runs within your cluster as well as a command line utility for interacting with the service.

## Key Capabilities

* **Backup and Recovery of Resources and Persistent Volumes** - Allows users to backup Kubernetes pods and/or persistent volumes.
* **Recovery/Migration** - Restore your pods to your existing cluster, or migrate the application to another cluster. Useful during cluster upgrades or greenfield scenarios.

### Environment Overview

our lab environment is hosted within public cloud. It is a single node Kubernetes instance. All necessary binaries have been installed for Kubernetes to function on this instance. In order to access this environment we will be leveraging a platform called [Strigo.io](https://strigo.io). When accessing via Strigo, you will notice you have several windows available to assist you with this lab.

* Terminal Windows
* Code Editor Window
* Web Browser Windows

## Lab Goals

In this lab we will be deploying a set of pods and services to our cluster, and backing them up with Velero. As we are working in a smaller lab environment, we will be using [MinIO](https://min.io) within our cluster as a storage location for our backups. MinIO is compatible with the Amazon S3 APIs, so we will also be leveraging the AWS CLI to interact with the platform.

### Step 1: Start Kubernetes

Execute the following command to get started in the lab!

```bash
k8s-start

```

### Step 2: Set our Variables

Execute the following command to save our External IP and External FQDN as variables for usage later.

```bash
export externalip=$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)
export externalfqdn=$(curl -s http://169.254.169.254/latest/meta-data/public-hostname)

```

**Note:** - You can also see this information by running the following command. You will need the external FQDN to use with the web browser on the workstation to view the page that's updated.

```bash
lab-info

```

### Step 3: Contour Quick-Start

We're going to be accessing a web based application in this lab, and we will leverage [Project Contour](https://projectcontour.io) to access the workload. We will configure the entire application upfront - which includes some specific configurations to make it functional for this lab.

```bash
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
wget https://gist.githubusercontent.com/codyde/5cc4eea515dba6970ef7e39848b73042/raw/e925ca9ec0d623572c1aa768cc0287f904f87b0a/envoy-update.yaml
sed -i 's/REPLACEME/'"$externalip"'/g' envoy-update.yaml
kubectl apply -f envoy-update.yaml --force

```

After these are applied, we can confirm our service has been updated with the external IP by entering...

```bash
kubectl get svc -n projectcontour
```

You should see the envoy service listed running on ports 80 and 443, example output is below (your IP will be different).

```bash
NAME      TYPE        CLUSTER-IP       EXTERNAL-IP    PORT(S)          AGE
contour   ClusterIP   10.96.102.194    <none>         8001/TCP         11m
envoy     ClusterIP   10.106.231.185   13.57.49.145   443/TCP,80/TCP   2m41s
```

If all was configured correct, you'll see the nodes External IP listed for connectivity.

### Step 4: Preparing Our Environment

We need to perform a few tasks to get our environment ready for Velero

* Install and configure AWS CLI
* Deploy our application
* Install MinIO in our Kubernetes cluster

Execute the below set of commands to perform these tasks.

```bash
sudo apt install -y awscli
kubectl apply -f https://raw.githubusercontent.com/codyde/kubecon-contour-lab/master/pods-and-services.yaml
wget https://raw.githubusercontent.com/codyde/kubecon-contour-lab/master/contour-final-state.yaml
sed -i 's/REPLACEME/'"$externalfqdn"'/g' contour-route.yaml
kubectl apply -f contour-final-state.yaml
kubectl apply -f https://gist.githubusercontent.com/codyde/222ad38e6331181aac41ef7df643d6bd/raw/b322be100ecd3b32f23efe6a5dca0e5081e911a2/minio-nopvc.yaml
wget https://gist.githubusercontent.com/codyde/fc61f8dd77830e67db3a72feea628216/raw/1d5d522652bd96fa2cd94f13da1236298e92bdfb/minio-service.yaml
sed -i 's/REPLACEME/'"$externalip"'/g' minio-service.yaml

```

Leveraging an secondary browser window, navigate to the FQDN you pulled down earlier. You should see our application deployed successfully! If you append :9000 to the end of the URL, you should see the MinIO UI.

We now need to configure the AWS CLI we installed previously

```bash
aws configure

```

This will ask you for a few pieces of information. Typically we would provide our content from AWS, or another provider, but in this case we will be using our MinIO credentials.

**AWS Access Key ID: minio**
**AWS Secret Access Key: minio123**
**Default region name: us-west-1**
**Default output format: leave blank**

With these credentials in place, we will use the AWS CLI to create our Velero bucket.

```bash
aws --endpoint-url http://$externalip:9000 s3 mb s3://velero
aws --endpoint-url http://$externalip:9000 s3 ls

```

Assuming the command was successful, you should see your bucket listed as the command output. We are ready to install Velero!

### Step 5: Installing Velero

Execute the following commands to pull down the Velero binaries

```bash
wget https://github.com/vmware-tanzu/velero/releases/download/v1.1.0/velero-v1.1.0-linux-amd64.tar.gz
tar -zxvf velero-v1.1.0-linux-amd64.tar.gz
sudo mv velero-v1.1.0-linux-amd64/velero /usr/local/bin/velero
```

We will need to create credentials for Velero to use to communicate with MinIO. Execute the following commands in your terminal window.

```bash
cat <<EOF > velero-credentials.ini
[default]
aws_access_key = minio
aws_secret_access_key = minio123
EOF

```

With these preparations in place, we can perform the installation of Velero on our target cluster.

```bash
velero install  --provider aws --bucket velero \
--secret-file ./credentials-velero.ini \
--use-volume-snapshots=false \
--use-restic \
--backup-location-config \
region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000,publicUrl=http://$externalip:9000

```

We can verify that Velero has completed it installation using the following command

```bash
kubectl get pods -n velero
```

You should see the restic provider, as well as the velero pod in a running state.

### Step 6: Our First Velero Backup

Velero uses labels/selectors on an application to create backups. Our application we deployed previously leverages labels for each tier, however if your application used a common label for all components, all of those components would be captured in the Velero backup task. In our case, let's create a backup job for the main frontend of our application.

```bash
velero backup create kubecon-app-backup --selector app=kubecon-minilab-app

```

When you execute the above command, the backup job will be requested. If we use the following command, we can check the status of the backup

```bash
velero backup describe kubecon-app-backup

```

Below is a sample output

```bash
Name:         kubecon-app-backup
Namespace:    velero
Labels:       velero.io/storage-location=default
Annotations:  <none>

Phase:  Completed

Namespaces:
  Included:  *
  Excluded:  <none>

Resources:
  Included:        *
  Excluded:        <none>
  Cluster-scoped:  auto

Label selector:  app=kubecon-minilab-app

Storage Location:  default

Snapshot PVs:  auto

TTL:  720h0m0s

Hooks:  <none>

Backup Format Version:  1

Started:    2019-11-14 00:07:57 +0000 UTC
Completed:  2019-11-14 00:08:00 +0000 UTC

Expiration:  2019-12-14 00:07:57 +0000 UTC

Persistent Volumes: <none included>
```

With the phase indicating Completed, we know the backup completed successfully. You can login to the MinIO UI at your `externalip:9000` to inspect the actual backup files, or use the AWS ClI. We can also use the following command to output all backups taken...

```bash
velero get backups

```

Which provides the following sample output

```bash
$ velero get backups
NAME                 STATUS      CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
db-backup            Completed   2019-11-12 06:55:23 +0000 UTC   27d       default            tier=backend
db-backup-role       Completed   2019-11-12 07:03:24 +0000 UTC   27d       default            role=db
kubecon-app-backup   Completed   2019-11-14 00:07:57 +0000 UTC   29d       default            app=kubecon-minilab-app
nginx-backup         Completed   2019-11-11 22:28:25 +0000 UTC   27d       default            app=nginx
```

### Step 7: Deleting and Restoring Our Application

Crisis has struck. Well... not really, but we can make it look like it has!

```bash
kubectl delete pod kubecon-minilab-app
kubectl get pods

```

When we refresh our browser Window for our application, we can see the main page no longer loads. The pod listing shows that our kubecon-minilab-app pod has been deleted. Fortunately, Velero can get us back in business quickly!

```bash
velero restore create kubecon-minilab-app --from-backup kubecon-app-backup

```

We can then check on the restore status with

```bash
velero restore describe kubecon-minilab-app

```

This should produce output similar to below!

```bash
Name:         kubecon-minilab-app
Namespace:    velero
Labels:       <none>
Annotations:  <none>

Phase:  Completed

Backup:  kubecon-app-backup

Namespaces:
  Included:  *
  Excluded:  <none>

Resources:
  Included:        *
  Excluded:        nodes, events, events.events.k8s.io, backups.velero.io, restores.velero.io, resticrepositories.velero.io
  Cluster-scoped:  auto

Namespace mappings:  <none>

Label selector:  <none>

Restore PVs:  auto
```

And finally, when we list out our pods, we should see our application has returned

```bash
kubectl get pods

```

Sample output is below

```bash
$ kubectl get pods -A
NAMESPACE        NAME                                       READY   STATUS      RESTARTS   AGE
default          kubecon-minilab-app                        1/1     Running     0          115s
default          kubecon-minilab-brm                        1/1     Running     1          4h20m
default          kubecon-minilab-kc                         1/1     Running     1          4h20m
default          kubecon-minilab-ll                         1/1     Running     1          4h20m
```

### Step 8: Daily Backup with Velero

In the previous example we used the on demand backup capability of Velero, but we can also schedule backups, as well as schedule how long to keep the backups.

```bash
velero schedule create kubecon-app-backup --selector app=kubecon-minilab-app --schedule "0 7 * * *" --ttl 24h0m0s

```

This schedule will create a backup on a daily schedule, and keep each backup for 24 hours. We can output our Velero backup schedules by using the following command

```bash
velero get schedules

```

Which provides the following sample output

```bash
$ velero get schedules
NAME                 STATUS    CREATED                         SCHEDULE    BACKUP TTL   LAST BACKUP   SELECTOR
kubecon-app-backup   Enabled   2019-11-14 17:00:30 +0000 UTC   0 7 * * *   24h0m0s      8s ago        app=kubecon-minilab-app
```

This concludes our Velero mini-lab. For more information on Velero, check out the projects webpage at [Velero.io](https://velero.io)
