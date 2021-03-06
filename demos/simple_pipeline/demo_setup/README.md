# Kubeflow demo - Simple pipeline

This repository contains a demonstration of Kubeflow capabilities, suitable for
presentation to public audiences.

The base demo includes the following steps:

1. [Setup your environment](#1-setup-your-environment)
1. [Create a GKE cluster and install Kubeflow](#2-create-a-gke-cluster-and-install-kubeflow)
1. [Install pipelines on GKE](#3-install-pipelines-on-gke)

## 1. Setup your environment

Clone the [kubeflow/kubeflow](https://github.com/kubeflow/kubeflow) repo and
checkout the
[`v0.3.3`](https://github.com/kubeflow/kubeflow/releases/tag/v0.3.3) branch.
Clone the [kubeflow/pipelines](https://github.com/kubeflow/pipelines) repo and
checkout the
[`0.1.2`](https://github.com/kubeflow/pipelines/releases/tag/0.1.2) branch.

Ensure that the repo paths, project name, and other variables are set correctly.
When all overrides are set, source the environment file:

```
source kubeflow-demo-simple-pipeline.env
```

Create a clean python environment for installing Kubeflow Pipelines:

```
conda create --name kfp python=3.6
source activate kfp
```

Install the Kubeflow Pipelines SDK:

```
pip install https://storage.googleapis.com/ml-pipeline/release/0.1.2/kfp.tar.gz --upgrade
```

## 2. Create a GKE cluster and install Kubeflow

Creating a cluster with click-to-deploy does not yet support the installation of
pipelines. It is not useful for demonstrating pipelines, but is still worth showing.

### Click-to-deploy

Generate a web app Client ID and Client Secret by following the instructions
[here](https://www.kubeflow.org/docs/started/getting-started-gke/#create-oauth-client-credentials).
Save these as environment variables for easy access.

In the browser, navigate to the
[Click-to-deploy app](https://deploy.kubeflow.cloud/). Enter the project name,
along with the Client ID and Client Secret previously generated. Select the
desired ${ZONE} and latest version of Kubeflow, then click _Create Deployment_.

In the [GCP Console](https://console.cloud.google.com/kubernetes), navigate to the
Kubernetes Engine panel to watch the cluster creation process. This results in a
full cluster with Kubeflow installed.

### kfctl

While node autoprovisioning is in beta, it must be enabled manually. To create
a cluster with autoprovisioning, run the following commands, which will take
around 30 minutes:

```
gcloud container clusters create ${CLUSTER} \
  --project ${DEMO_PROJECT} \
  --zone ${ZONE} \
  --cluster-version 1.11.2-gke.9 \
  --num-nodes=8 \
  --scopes cloud-platform,compute-rw,storage-rw \
  --verbosity error

# scale down cluster to 3 (initial 8 is just to prevent master restarts due to upscaling)
# we cannot use 0 because then cluster autoscaler treats the cluster as unhealthy.
# Also having a few small non-gpu nodes is needed to handle system pods
gcloud container clusters resize ${CLUSTER} \
  --project ${DEMO_PROJECT} \
  --zone ${ZONE} \
  --size=3 \
  --node-pool=default-pool

# enable node auto provisioning
gcloud beta container clusters update ${CLUSTER} \
  --project ${DEMO_PROJECT} \
  --zone ${ZONE} \
  --enable-autoprovisioning \
  --max-cpu 20 \
  --max-memory 200 \
  --max-accelerator=type=nvidia-tesla-k80,count=8
```

Once the cluster has been created, install GPU drivers:

```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/master/daemonset.yaml
```

Add RBAC permissions, which allows your user to install kubeflow components on
the cluster:

```
kubectl create clusterrolebinding cluster-admin-binding-${USER} \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)
```

Setup kubectl access:

```
kubectl create namespace kubeflow
./create_context.sh gke ${NAMESPACE}
```

Setup OAuth environment variables ${CLIENT_ID} and ${CLIENT_SECRET} using the
instructions
[here](https://www.kubeflow.org/docs/started/getting-started-gke/#create-oauth-client-credentials).

```
kubectl create secret generic kubeflow-oauth --from-literal=client_id=${CLIENT_ID} --from-literal=client_secret=${CLIENT_SECRET}
```

Create service accounts, add permissions, download credentials, and create secrets:

```
ADMIN_EMAIL=${CLUSTER}-admin@${PROJECT}.iam.gserviceaccount.com
USER_EMAIL=${CLUSTER}-user@${PROJECT}.iam.gserviceaccount.com
ADMIN_FILE=${HOME}/.ssh/${ADMIN_EMAIL}.json
USER_FILE=${HOME}/.ssh/${ADMIN_EMAIL}.json

gcloud iam service-accounts create ${CLUSTER}-admin --display-name=${CLUSTER}-admin
gcloud iam service-accounts create ${CLUSTER}-user --display-name=${CLUSTER}-user

gcloud projects add-iam-policy-binding ${PROJECT} \
  --member=serviceAccount:${ADMIN_EMAIL} \
  --role=roles/storage.admin
gcloud projects add-iam-policy-binding ${PROJECT} \
  --member=serviceAccount:${USER_EMAIL} \
  --role=roles/storage.admin

gcloud iam service-accounts keys create ${ADMIN_FILE} \
  --project ${PROJECT} \
  --iam-account ${ADMIN_EMAIL}
gcloud iam service-accounts keys create ${USER_FILE} \
  --project ${PROJECT} \
  --iam-account ${USER_EMAIL}

kubectl create secret generic admin-gcp-sa \
  --from-file=admin-gcp-sa.json=${ADMIN_FILE}
kubectl create secret generic user-gcp-sa \
  --from-file=user-gcp-sa.json=${USER_FILE}
```

Install kubeflow with the following commands:

```
kfctl init ${CLUSTER} --platform gcp
cd ${CLUSTER}
kfctl generate k8s
kfctl apply k8s
```

## 3. Install pipelines on GKE

```
kubectl create clusterrolebinding sa-admin --clusterrole=cluster-admin --serviceaccount=kubeflow:pipeline-runner
cd ks_app
ks registry add ml-pipeline "${PIPELINES_REPO}/ml-pipeline"
ks pkg install ml-pipeline/ml-pipeline
ks generate ml-pipeline ml-pipeline
ks param set ml-pipeline namespace kubeflow
ks apply default -c ml-pipeline
```

View the installed components in the GCP Console. In the
[Kubernetes Engine](https://console.cloud.google.com/kubernetes)
section, you will see a new cluster ${CLUSTER}. Under
[Workloads](https://console.cloud.google.com/kubernetes/workload),
you will see all the default Kubeflow and pipeline components.


