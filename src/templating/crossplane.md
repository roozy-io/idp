# Crossplane

## Links

* [Video By Shahrooz Aghili](https://www.youtube.com/watch?v)
* [Crossplane](https://crossplane.io/)
* [Docs](https://docs.crossplane.io/)

Highlights and Intro:
> **Crossplane** is an advanced tool for managing infrastructure in the cloud-native ecosystem. 

It distinguishes itself from traditional Configuration Management and Infrastructure as Code (IaC) tools. Platforms like Chef, Puppet, and Ansible have become less prevalent, while Terraform and Pulumi, despite their popularity, lack efficient drift detection. This means if a resource is modified outside Terraform, it won't detect the change. Crossplane, on the other hand, excels in detecting drifts and integrates seamlessly with GitOps workflows, allowing infrastructure management directly through kubectl.

**Benefits**:

* Manages infrastructure in a cloud-native way using kubectl.
* Supports major cloud providers and is continually updated with new ones.
* Allows building custom infrastructure abstractions.
* Based on Custom Resource Definitions (CRD), making it extensible and runnable anywhere.
* Acts as a single source of truth for infrastructure management.
* Enables management of policies through custom APIs, simplifying infrastructure complexity.
* Offers Declarative Infrastructure Configuration: self-healing and accessible via kubectl and YAML.
* Integrates with leading GitOps tools like Flux & ArgoCD.

## Install Crossplane using Helm

Crossplane is an extension to the k8s api, therefore we need a cluster (any cluster would do) to install it. This cluster is some time refered to as operations cluster. In this tutorial we are going to use minikube.

```jsx
minikube start
```

Once minikube is up and running, we can install cross-plane by running a **helm** command.

```jsx
helm repo add \
crossplane-stable https://charts.crossplane.io/stable

helm repo update

helm install crossplane \
crossplane-stable/crossplane \
--namespace crossplane-system \
--create-namespace
```

Now, Let's see if crossplane is running without any issues.

```jsx
kubectl get pods -n crossplane-system
```

Crossplane is extending the kubernetes api. let's checkout the new custom resource definitions and api.

```jsx
kubectl api-resources  | grep crossplane
kubectl get crds | grep crossplane
```

## Setting up the Hyperscaler (GCP)

Next, we need to enable crossplane access to our Hyperscaler of choice. All we need is a secret in the `crossplane-system` namespace.
The secret should contain a service account key.json.
Please note that the service account should be granted proper permissions for resoruce adminstration.

```jsx
export PROJECT_ID="YOUR-PROJECT-ID"
export SA_NAME="YOUR-SA-NAME"

export SA="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud iam service-accounts \
    create $SA_NAME \
    --project $PROJECT_ID

export ROLE=roles/admin

gcloud projects add-iam-policy-binding \
    --role $ROLE $PROJECT_ID \
    --member serviceAccount:$SA

gcloud iam service-accounts keys \
    create creds.json \
    --project $PROJECT_ID \
    --iam-account $SA

kubectl create secret \
generic gcp-secret \
-n crossplane-system \
--from-file=creds=./creds.json
```

## Setting up the GCP storage provider

Next, we install and configure a crossplane provider that specializes in managing storage related services in GCP.

```jsx
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp-storage
spec:
  package: xpkg.upbound.io/upbound/provider-gcp-storage:v0.35.0
EOF
```

Let's give it some time until our provider becomes healthy.

```jsx
kubectl get providers --watch
```

Next, we need to configure the provider and make it aware of the GCP secret which we created in the previous step   .

```jsx
cat <<EOF | kubectl apply -f -
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: $PROJECT_ID
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: gcp-secret
      key: creds
EOF
```

## Creating a Managed Storage Bucket

The provider is ready to listen to our api requests, Let's create a bucket

```jsx

BUCKET_NAME=my-crossplane-bucket
cat <<EOF | kubectl create -f -
apiVersion: storage.gcp.upbound.io/v1beta1
kind: Bucket
metadata:
  name: $BUCKET_NAME
  labels:
    docs.crossplane.io/example: provider-gcp
spec:
  forProvider:
    location: US
  providerConfigRef:
    name: default
EOF
```

## Drift Detection

If our managed resouces get modified or deleted outside the scope of crossplane, crossplane provider can react to the drift and reconsile the infrastructure to the desired config.
Let's try deleting the bucket from the GCP console. Once done, we can watch the managed bucket again:

```jsx
watch kubectl get buckets
```

It takes a few minutes unitl the bucket is ready again,

## Cleanup

```jsx

kubectl delete bucket $BUCKET_NAME
minikube stop
minikube delete  
```
