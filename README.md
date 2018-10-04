# Overview

Planet9 is a low-code, rapid application development software that unifies the
end-user experience and app development to integrate with any cloud, any backend
and any architecture across mobile, desktop and offline environments.

[Learn more](https://www.neptune-software.com/).

## About Google Click to Deploy

Popular open stacks on Kubernetes packaged by Google.

# Installation

## Quick install with Google Cloud Marketplace

Get up and running with a few clicks! Install Planet9 to a
Google Kubernetes Engine cluster using Google Cloud Marketplace. Follow the
[on-screen instructions](https://console.cloud.google.com/marketplace/details/google/planet9).

## Command line instructions

There are two ways to set this up. The first is the manual approach where you
run all the commands explicitly. The second approach uses a supplied makefile
which should enable you to publish the image using only a few commands.

### Prerequisites

#### Set up command-line tools

You'll need the following tools in your development environment:
- [gcloud](https://cloud.google.com/sdk/gcloud/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)
- [docker](https://docs.docker.com/install/)
- [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

Configure `gcloud` as a Docker credential helper:

```shell
gcloud auth configure-docker
```

#### Create a Google Kubernetes Engine cluster

Create a new cluster from the command line:

```shell
export CLUSTER=planet9-cluster
export ZONE=us-west1-a

gcloud container clusters create "$CLUSTER" --zone "$ZONE"
```

Configure `kubectl` to connect to the new cluster.

```shell
gcloud container clusters get-credentials "$CLUSTER" --zone "$ZONE"
```

#### Clone this repo

Clone this repo and the associated tools repo:

```shell
git clone --recursive https://github.com/bergheim/planet9-k8s
```

#### Install the Application resource definition

An Application resource is a collection of individual Kubernetes components,
such as Services, Deployments, and so on, that you can manage as a group.

To set up your cluster to understand Application resources, enter the directory
of the repo you just cloned and run the following commands:

```shell
git submodule init
git submodule sync --recursive
git submodule update --init --recursive
kubectl apply -f k8s/vendor/marketplace-tools/crd/
```

You need to run this command once.

The Application resource is defined by the
[Kubernetes SIG-apps](https://github.com/kubernetes/community/tree/master/sig-apps) community. The source code can be found on
[github.com/kubernetes-sigs/application](https://github.com/kubernetes-sigs/application).

### Install the Application

Navigate to the `planet9` directory:

```shell
cd k8s/planet9
```

#### Configure the app with environment variables

Choose an instance name and
[namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
for the app. The default namespace is simply `default`, but we recommend you to
change it to something else.

```shell
export APP_INSTANCE_NAME=planet9-1
export NAMESPACE=default
```

Specify the number of replicas for the Planet9 cluster:

```shell
export REPLICAS=2
```

Configure the container images:

```shell
export IMAGE_PLANET9="gcr.io/neptune-software/planet9:latest"
export IMAGE_POSTGRESQL="launcher.gcr.io/google/postgresql9:latest"
```

The images above are referenced by
[tag](https://docs.docker.com/engine/reference/commandline/tag). We recommend
that you pin each image to an immutable
[content digest](https://docs.docker.com/registry/spec/api/#content-digests).
This ensures that the installed application always uses the same images,
until you are ready to upgrade. To get the digest for the image, use the
following script:

```shell
for i in "IMAGE_PLANET9" "IMAGE_POSTGRESQL"; do
  repo=$(echo ${!i} | cut -d: -f1);
  digest=$(docker pull ${!i} | sed -n -e 's/Digest: //p');
  export $i="$repo@$digest";
  env | grep $i;
done
```

#### Create namespace in your Kubernetes cluster

If you use a different namespace than the `default`, run the command below to create a new namespace:

```shell
kubectl create namespace "$NAMESPACE"
```

#### Expand the manifest template

Use `envsubst` to expand the template. We recommend that you save the
expanded manifest file for future updates to the application.

```shell
awk 'BEGINFILE {print "---"}{print}' manifest/* \
  | envsubst '$APP_INSTANCE_NAME $NAMESPACE $IMAGE_PLANET9 $IMAGE_POSTGRESQL $REPLICAS' \
  > "${APP_INSTANCE_NAME}_manifest.yaml"
```

#### Apply the manifest to your Kubernetes cluster

Use `kubectl` to apply the manifest to your Kubernetes cluster:

```shell
kubectl apply -f "${APP_INSTANCE_NAME}_manifest.yaml" --namespace "${NAMESPACE}"
```

#### View the app in the Google Cloud Console

To get the Console URL for your app, run the following command:

```shell
echo "https://console.cloud.google.com/kubernetes/application/${ZONE}/${CLUSTER}/${NAMESPACE}/${APP_INSTANCE_NAME}"
```

To view the app, open the URL in your browser.

### Using the Makefile

After the repository is cloned and gcloud is set up as described above, enter
the directory and download the support tools:

```shell
make
```

Enter the Planet9 config directory:

```shell
cd k8s/Planet9
export APP_INSTANCE_NAME=<<projectname>>
make app/install
```

### (Optional) Expose the Planet9 service externally

By default, the application does not have an external IP. To create an external
IP address, run the following command:

```shell
kubectl patch svc "$APP_INSTANCE_NAME-planet9-service" \
  --namespace "$NAMESPACE" \
  --patch '{"spec": {"type": "LoadBalancer"}}'
```

It might take some time for the external IP address to be created.

# Get the Planet9 URL

If you run your Planet9 cluster behind a LoadBalancer service, use the
following command to get the IP address. You can use the IP address to run
administrative operations using the REST API:

```
SERVICE_IP=$(kubectl get svc $APP_INSTANCE_NAME-planet9-service \
  --namespace $NAMESPACE \
  --output jsonpath='{.status.loadBalancer.ingress[0].ip}')

PLANET9_URL="http://${SERVICE_IP}:9200"
```

If you haven't exposed your Planet9 service externally, use a local proxy to access the service. In a separate terminal, run the following command:

```shell
# select a local port as the proxy
KUBE_PROXY_PORT=8080
kubectl proxy -p $KUBE_PROXY_PORT
```

In your main terminal, run the following commands:

```shell
KUBE_PROXY_PORT=8080
PROXY_BASE_URL=http://localhost:$KUBE_PROXY_PORT/api/v1/proxy
PLANET9_URL=$PROXY_BASE_URL/namespaces/$NAMESPACE/services/$APP_INSTANCE_NAME-planet9-service:http
```

In both cases, the `PLANET9_URL` environment variable points to your
Planet9 base URL. Verify the variable using `curl`:

```shell
curl "${PLANET9_URL}"
```

In the response, you should get the initial HTML for the login screen for Planet9:

### Scale the cluster

Scale the number of master node replicas by the following command:

```
kubectl scale deployment "$APP_INSTANCE_NAME-planet9" \
  --namespace "$NAMESPACE" --replicas=<new-replicas>
```

By default, there are 1 replica set up. Increase the number of replicas as you
need more power.

# Updating the app

1. Stop Planet9
2. Run the upgrade
3. Start Planet9

# Uninstall the Application

## Using the Google Cloud Platform Console

1. In the GCP Console, open [Kubernetes Applications](https://console.cloud.google.com/kubernetes/application).

1. From the list of applications, click **Planet9**.

1. On the Application Details page, click **Delete**.

## Using the command line

### Prepare the environment

Set your installation name and Kubernetes namespace:

```shell
export APP_INSTANCE_NAME=planet9-1
export NAMESPACE=default
```

### Delete the resources

> **NOTE:** We recommend to use a kubectl version that is the same as the version of your cluster. Using the same versions of kubectl and the cluster helps avoid unforeseen issues.

To delete the resources, use the expanded manifest file used for the
installation.

Run `kubectl` on the expanded manifest file:

```shell
kubectl delete -f ${APP_INSTANCE_NAME}_manifest.yaml --namespace $NAMESPACE
```

If you don't have the expanded manifest file, delete the resources using types and a label:

```shell
kubectl delete application,secret,service,deployments,jobs,sts,application \
  --namespace $NAMESPACE \
  --selector app.kubernetes.io/name=$APP_INSTANCE_NAME
```
