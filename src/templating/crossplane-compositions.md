# Crossplane Compositions

## Links

* [Video By Shahrooz Aghili](https://www.youtube.com/watch?v)
* [Crossplane](https://crossplane.io/)
* [Docs](https://docs.crossplane.io/)

Highlights and Intro:
> **Crossplane** is an advanced tool for managing infrastructure in the cloud-native ecosystem. 
> Just like terraform encourages using modules for bundling related resources, crossplane offers compositions.
> `Platform engineers` can define their compositions and provide the `Devs` a simple claim api.
> `Devs` claim their resources and the composition takes care of the rest.


### start minikube
```
minikube start
```

### install crossplane
```
helm repo add crossplane-stable \
    https://charts.crossplane.io/stable

helm repo update

helm upgrade --install \
    crossplane crossplane-stable/crossplane \
    --namespace crossplane-system \
    --create-namespace \
    --wait
```


### create GCP credentials secret for crossplane

```

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
    create gcp-creds.json \
    --project $PROJECT_ID \
    --iam-account $SA

kubectl --namespace crossplane-system \
    create secret generic gcp-creds \
    --from-file creds=./gcp-creds.json
```

### clone playbook
```
git clone https://github.com/aghilish/crossplane-compositions-vs-terraform-modules.git

cd crossplane-compositions-vs-terraform-modules
```

### install dependencies
```
kubectl apply --filename dependencies.yaml
```

### wait for all the packages to become healthy
```
watch kubectl get pkgrev
```

### configure provider
```
PROJECT_ID=$(gcloud config get-value project)

echo "apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: $PROJECT_ID
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: gcp-creds
      key: creds" \
    | kubectl apply --filename -
```

### apply XRD
```
kubectl apply --filename xrd.yaml
```

### apply composition
```
kubectl apply --filename composition.yaml
```


### create infra namespace
```
kubectl create namespace a-team
```


### apply claim

```
kubectl apply --filename a-team/claim.yaml -n a-team
```

### verify resources


```
kubectl get clusterclaims.example.org -n a-team
```

```
kubectl describe CompositeCluster.example.org
```

```
kubectl get clusters,nodepools
```

### access the GKE cluster
```
kubectl --namespace a-team  \
    get secret a-team-gke-cluster \
    --output jsonpath="{.data.kubeconfig}" \
    | base64 -d \
    | tee kubeconfig.yaml

export KUBECONFIG=$PWD/kubeconfig.yaml

kubectl get nodes

kubectl get namespaces

```

### destroy infrastructure

```
unset KUBECONFIG
kubectl delete -n a-team --filename a-team/claim.yaml
```