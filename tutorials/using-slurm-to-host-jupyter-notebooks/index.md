---
title: Using the Slurm Resource Manager to host Jupyter Notebooks
description: Learn how to run your Jupyter Notebooks on a Compute Engine instance managed by the Slurm Resource Manager
author: wardharold
tags: GCE, Slurm, Jupyter
date_published: 2018-12-14
---
This tutorial shows you how to run a [Jupyter Notebook]() as a job managed by the [Slurm Resource Manager]().
Slurm is a popular resource manager used in many High Performance Computing centers. Jupyter notebooks are a
favorite tool of Machine Learning and Data Science specialists. While they are often run on an individual user's
laptop there are situations that call for specialized hardware, e.g., GPUs, or more memory or cores than are
available locally. In those situations Slurm can allocate a compute instance has the requisite hardware
or memory/cpu resources to run the user's notebook for a bounded time period.

[![button](http://gstatic.com/cloudssh/images/open-btn.png)](https://console.cloud.google.com/cloudshell/open?git_repo=https://github.com/GoogleCloudPlatform/community&page=editor&tutorial=tutorials/using-slurm-to-host-jupyter-notebooks/index.md)

## (OPTIONAL) Create a project with a billing account attached 
**(you can also use an existing project and skip to the next step)**

Edit <walkthrough-editor-open-file filePath="community/tutorials/using-slurm-to-host-jupyter-notebooks/env.sh">env.sh</walkthrough-editor-open-file> setting these variables to reflect your environment.
- <walkthrough-editor-select-regex filePath="community/tutorials/using-slurm-to-host-jupyter-notebooks/env.sh" regex="\[YOUR_ORG\]">organization</walkthrough-editor-select-regex>
- <walkthrough-editor-select-regex filePath="community/tutorials/using-slurm-to-host-jupyter-notebooks/env.sh" regex="\[YOUR_BILLING_ACCOUNT_NAME\]">billing account</walkthrough-editor-select-regex>
- <walkthrough-editor-select-regex filePath="community/tutorials/using-slurm-to-host-jupyter-notebooks/env.sh" regex="\[NAME FOR THE PROJECT YOU WILL CREATE\]">project name</walkthrough-editor-select-regex>
- <walkthrough-editor-select-regex filePath="community/tutorials/using-slurm-to-host-jupyter-notebooks/env.sh" regex="\[COMPUTE ZONE YOU WANT TO USE\]">zone</walkthrough-editor-select-regex>

```bash
source ./env.sh
```
```bash
gcloud projects create $PROJECT --organization=$ORG
```
```bash
gcloud beta billing projects link $PROJECT --billing-account=$(gcloud beta billing accounts list | grep $BILLING_ACCOUNT | awk '{print $1}')
```
```bash
gcloud config configurations create -- activate $PROJECT
```
```bash
gcloud config set compute/zone $ZONE
```

## Enable the required Google APIs
```bash
gcloud services enable compute.googleapis.com
```

## Create a Slurm cluster

1. Clone the Slurm for GCP Git repository
```bash
git clone https://github.com/schedmd/slurm-gcp.git
```
```bash
cd slurm-gcp
```

2. Modify slurm-cluster.yaml for your environment

You need to customize the <walkthrough-editor-open-file filePath="community/tutorials/using-slurm-to-host-jupyter-notebooks/slurm-gcp/slurm-cluster.yaml">slurm-cluster.yaml</walkthrough-editor-open-file> file
for your environment before you deploy your cluster.

* Uncomment this <walkthrough-editor-select-regex filePath="community/tutorials/using-slurm-to-host-jupyter-notebooks/slurm-gcp/slurm-cluster.yaml" regex="login_node_count">line</walkthrough-editor-select-regex>.
If you want more than one login node modify the value of ```login_node_count``` accordingly.
* Add a comma separated list of the user ids authorized to use your cluster on the
<walkthrough-editor-select-regex filePath="community/tutorials/using-slurm-to-host-jupyter-notebooks/slurm-gcp/slurm-cluster.yaml" regex="default_user">default user</walkthrough-editor-select-regex>
line

You may also want to make one or more optional changes:

* Deploy your cluster to a different region and/or zone by modifying the values
<walkthrough-editor-select-regex filePath="community/tutorials/using-slurm-to-host-jupyter-notebooks/slurm-gcp/slurm-cluster.yaml" regex="region.*:">here</walkthrough-editor-select-regex>
and <walkthrough-editor-select-regex filePath="community/tutorials/using-slurm-to-host-jupyter-notebooks/slurm-gcp/slurm-cluster.yaml" regex="zone.*:">here</walkthrough-editor-select-regex>
* Use a different type of <walkthrough-editor-select-regex filePath="community/tutorials/using-slurm-to-host-jupyter-notebooks/slurm-gcp/slurm-cluster.yaml" regex="compute_machine_type">compute node</walkthrough-editor-select-regex>, 
e.g., if you need more cores or memory than are available in the default choice of ```n1-standard-2```
* Use an existing <walkthrough-editor-select-regex filePath="community/tutorials/using-slurm-to-host-jupyter-notebooks/slurm-gcp/slurm-cluster.yaml" regex="vpc_net">VPC network</walkthrough-editor-select-regex> and
<walkthrough-editor-select-regex filePath="community/tutorials/using-slurm-to-host-jupyter-notebooks/slurm-gcp/slurm-cluster.yaml" regex="vpc_subnet">VPC subnet</walkthrough-editor-select-regex> combination. The
network/subnet requirements are described in the file ```slurm.jinja.scheme```
* Specify a different <walkthrough-editor-select-regex filePath="community/tutorials/using-slurm-to-host-jupyter-notebooks/slurm-gcp/slurm-cluster.yaml" regex="slurm_version">version</walkthrough-editor-select-regex>
of Slurm for your cluster. By default the latest stable version, 17.11.8 at the time of this writing, will be deployed

3. Patch the Slurm startup-script

You need to patch the script that is run on each cluster node at startup. The changes in the patch setup the symbolic links
and directories to support the installation of software packages shared across nodes and the [environment modules]() uesd
to access those packages.

```bash
patch scripts/startup-script.py ../startup-script.patch
```

3. Deploy Slurm using Deployment Manager

4. Verify that your cluster is operational

## Setup an Anaconda environment module

1. Create the cluster and get its credentials

        CLUSTER=[NAME OF THE KUBERNETES CLUSTER YOU WILL CREATE]
        gcloud container clusters create ${CLUSTER}
        gcloud container clusters get-credentials ${CLUSTER}

2. Grant yourself cluster-admin privileges

        ACCOUNT=$(gcloud config get-value core/account)
        kubectl create clusterrolebinding core-cluster-admin-binding \
            --user ${ACCOUNT} \
            --clusterrole cluster-admin

3. Install [Helm](https://github.com/helm/helm)

    Download the [desired version](https://github.com/helm/helm/releases) and unpack it.

        wget https://storage.googleapis.com/kubernetes-helm/helm-v2.11.0-linux-amd64.tar.gz
        tar xf helm-v2.11.0-linux-amd64.tar.gz

    Add the `helm` binary to `/usr/local/bin`.

        sudo ln -s $PWD/linux-amd64/helm /usr/local/bin/helm

    Create a file named `rbac-config.yaml` containing the following:

        apiVersion: v1
        kind: ServiceAccount
        metadata:
          name: tiller
          namespace: kube-system
        ---
        apiVersion: rbac.authorization.k8s.io/v1beta1
        kind: ClusterRoleBinding
        metadata:
          name: tiller
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: cluster-admin
        subjects:
          - kind: ServiceAccount
            name: tiller
            namespace: kube-system

    Create the `tiller` service account and `cluster-admin` role binding.

        kubectl apply -f rbac-config.yaml

    Initialize Helm.

        helm init --service-account tiller

## Run a Jupyter Notebook as a Slurm batch job

Create an instance of NFS-Client Provisioner connected to the Cloud Filestore instance you created earlier 
via its IP address (`${FSADDR}`). The NFS-Client Provisioner creates a new storage class: `nfs-client`. Persistent
volume claims against that storage class will be fulfilled by creating persistent volumes backed by directories
under the `/volumes` directory on the Cloud Filestore instance's managed storage.

    helm install stable/nfs-client-provisioner --name nfs-cp --set nfs.server=${FSADDR} --set nfs.path=/volumes
    watch kubectl get po -l app=nfs-client-provisioner

Press Ctrl-C when the provisioner pod's status changes to Running.

## Setup an ssh tunnel to your Notebook

While you can use any application that uses storage classes to do [dynamic provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
to test the NFS-Client Provisioner, in this tutorial you will deploy a [PostgreSQL](https://www.postgresql.org/) instance to verify
the configuration.

    helm install --name postgresql --set persistence.storageClass=nfs-client stable/postgresql
    watch kubectl get po -l app=postgresql

Press Ctrl-C when the database pod's status changes to Running.

The PostgreSQL Helm chart creates an 8GB persistent volume claim on Cloud Filestore and mounts it at
`/var/lib/postgresql/data/pgdata` in the database pod.

## Connect to your Notebook

To verify that the PostgreSQL database files were actually created on the Cloud Filestore managed storage you will
create a small Compute Engine instance, mount the Cloud Filestore volume on that instance, and inspect the directory
structure to see that the database files are present.

1. Create an `f1-micro` Compute Engine instance

        gcloud compute --project=${PROJECT} instances create check-nfs-provisioner \
            --zone=${ZONE} \
            --machine-type=f1-micro \
            --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append \
            --image=debian-9-stretch-v20180911 \
            --image-project=debian-cloud \
            --boot-disk-size=10GB \
            --boot-disk-type=pd-standard \
            --boot-disk-device-name=check-nfs-provisioner

2. Install the nfs-common package on check-nfs-provisioner

        gcloud compute ssh check-nfs-provisioner --command "sudo apt update -y && sudo apt install nfs-common -y"

3. Mount the Cloud Filestore volume on check-nfs-provisioner

        gcloud compute ssh check-nfs-provisioner --command "sudo mkdir /mnt/gke-volumes && sudo mount ${FSADDR}:/volumes /mnt/gke-volumes"

4. Display the PostgreSQL database directory structure

        gcloud compute ssh check-nfs-provisioner --command  "sudo find /mnt/gke-volumes -type d"

   You will see output like the following modulo the name of the PostgreSQL pod

        /mnt/gke-volumes
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_twophase
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_notify
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_clog
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_stat_tmp
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_stat
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_serial
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/global
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_commit_ts
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_dynshmem
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_snapshots
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_replslot
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_logical
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_logical/snapshots
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_logical/mappings
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_tblspc
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_subtrans
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/base
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/base/1
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/base/12406
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/base/12407
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_xlog
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_xlog/archive_status
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_multixact
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_multixact/offsets
        /mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_multixact/members


## Clean up

1. Delete the check-nfs-provisioner instance

        gcloud compute instances delete check-nfs-provisioner

2. Delete the Kubernetes Engine cluster

        helm destroy postgresql
        helm destroy nfs-cp
        gcloud container clusters delete ${CLUSTER}

3. Delete the Cloud Filestore instance

        gcloud beta filestore instances delete ${FS} --location ${ZONE}

