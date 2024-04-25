# Extending k8s api like a pro.

# Links
* [Video by Shahrooz Aghili](https://www.youtube.com/watch?v=)

# Learning Resources

1. [Extending k8s](https://kubernetes.io/docs/concepts/extend-kubernetes/)
2. [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

# Notes

> Kubernetes is highly configurable and extensible. As a result, there is rarely a need to fork or submit patches to the Kubernetes project code.

> Extension Points [Details](https://kubernetes.io/docs/concepts/extend-kubernetes/#key-to-the-figure)
<img src="../assets/extension-points.png" alt="k8s Extension Points" width="100%">

## In this tutorial we will be focusing on extension point 2 (Kubernetes API).
| Declarative APIs | Imperative APIs |
|-----------------|-----------------|
| Your API consists of a relatively small number of relatively small objects (resources). | The client says "do this", and then gets a synchronous response back when it is done. |
| The objects define configuration of applications or infrastructure. | The client says "do this", and then gets an operation ID back, and has to check a separate Operation object to determine completion of the request. |
| The objects are updated relatively infrequently. | You talk about Remote Procedure Calls (RPCs). |
| Humans often need to read and write the objects. | Directly storing large amounts of data; for example, > a few kB per object, or > 1000s of objects. |
| The main operations on the objects are CRUD-y (creating, reading, updating and deleting). | High bandwidth access (10s of requests per second sustained) needed. |
| Transactions across objects are not required: the API represents a desired state, not an exact state. | Store end-user data (such as images, PII, etc.) or other large-scale data processed by applications. |
| | The natural operations on the objects are not CRUD-y. |
| | The API is not easily modeled as objects. |
| | You chose to represent pending operations with an operation ID or an operation object. |

> Kubernetes is designed to be automated by writing client programs. Any program that reads and/or writes to the Kubernetes API can provide useful automation. Automation can run on the cluster or off it.

> There is a specific pattern for writing client programs that work well with Kubernetes called the controller pattern. Controllers typically read an object's `.spec`, possibly do things, and then update the object's `.status`.




```shell
kind create cluster
```

let's start by using the `custom resource definition` api of kubernetes. 
let's create a custom resource called `CronTab`

```shell
kubectl apply -f - << EOF
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # list of versions supported by this CustomResourceDefinition
  versions:
    - name: v1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
      
      additionalPrinterColumns:
        - name: Spec
        type: string
        description: The cron spec defining the interval a CronJob is run
        jsonPath: .spec.cronSpec
        - name: Replicas
        type: integer
        description: The number of jobs launched by the CronJob
        jsonPath: .spec.replicas
        - name: Age
        type: date
        jsonPath: .metadata.creationTimestamp
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
EOF
```
now our CronTab resource type is created. 
```shell
kubectl get crd
```
A new namespaced RESTful API endpoint is created at:

```
/apis/stable.example.com/v1/namespaces/*/crontabs/...
```

Let's verify the k8s api extension by looking at the api server logs:

```shell
kubectl -n kube-system logs -f kube-apiserver-kind-control-plane | grep example.com
```

Now we can create custom objects of our new custom resource defintion.
In the following example, the `cronSpec` and `image` custom fields are set in a custom object of kind `CronTab`. The kind `CronTab` comes from the `spec` of the CustomResourceDefinition object you created above.

```shell
kubectl apply -f - << EOF
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
EOF
```

```shell
kubectl get crontab
```

Let's see how our object is being persisted at the etcd database of k8s.

```shell
kubectl exec etcd-kind-control-plane -n kube-system -- sh -c "ETCDCTL_API=3 etcdctl --cacert /etc/kubernetes/pki/etcd/ca.crt  --key /etc/kubernetes/pki/etcd/server.key --cert  /etc/kubernetes/pki/etcd/server.crt  get / --prefix --keys-only" | grep example.com

kubectl exec etcd-kind-control-plane -n kube-system -- sh -c "ETCDCTL_API=3 etcdctl --cacert /etc/kubernetes/pki/etcd/ca.crt  --key /etc/kubernetes/pki/etcd/server.key --cert  /etc/kubernetes/pki/etcd/server.crt  get /registry/apiextensions.k8s.io/customresourcedefinitions/crontabs.stable.example.com --prefix -w json" | jq ".kvs[0].value" | cut -d '"' -f2 | base64 --decode | yq > crd.yml

kubectl exec etcd-kind-control-plane -n kube-system -- sh -c "ETCDCTL_API=3 etcdctl --cacert /etc/kubernetes/pki/etcd/ca.crt  --key /etc/kubernetes/pki/etcd/server.key --cert  /etc/kubernetes/pki/etcd/server.crt  get /registry/apiregistration.k8s.io/apiservices/v1.stable.example.com --prefix -w json" | jq ".kvs[0].value" | cut -d '"' -f2 | base64 --decode | yq > api-registration.yml

kubectl exec etcd-kind-control-plane -n kube-system -- sh -c "ETCDCTL_API=3 etcdctl --cacert /etc/kubernetes/pki/etcd/ca.crt  --key /etc/kubernetes/pki/etcd/server.key --cert  /etc/kubernetes/pki/etcd/server.crt  get /registry/stable.example.com/crontabs/default/my-new-cron-object --prefix -w json" | jq ".kvs[0].value" | cut -d '"' -f2 | base64 --decode | yq > mycron.yml
```

Delete custom resource 
```shell
kubectl delete CronTab my-new-cron-object
```

//TODO: add some text  why we need kubebuilder and what is an operator application and the relationship between a controller and an operator etc.
let's start by installing kubebuilder
//TODO: add kubebuilder to the killercoda
```shell
# download kubebuilder and install locally.
curl -L -o kubebuilder "https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)"
chmod +x kubebuilder && mv kubebuilder /usr/local/bin/
```

let's scaffold a kubebuilder application
```shell
mkdir operator-tutorial
cd operator-tutorial
kubebuilder init --repo example.com
```

let's have a closer look at the make file first.
make targets are the commands that are used for different development lifecycle steps
```shell
make help
```

to run your kubebuilder application locally
```shell
make run
```

now let's have a look at the `run` target and all the prerequisite comamnds that need to run
it looks something like this
```shell
.PHONY: run
run: manifests generate fmt vet ## Run a controller from your host.
	go run ./cmd/main.go
```

> so the targets that need to run before we can run our applications are 
> 1. `manifests` and `generate` which both have controller-gen as prerequisite and generate some golang code and yaml manifests 
> 2. the code is formatted by `fmt` 
> 3. validated by `vet` 
> 4. run will run the go application by refering to the application entrypoint at ./cmd/main.go 

## Now we have a working yet empty go application. 
let's add some meaningful code to it 

Let's imagine we are a working at company where our colleagues are heavy users of the `ghost` blogging application.
Our job is to provide them with ghost instances whenever and whereever they want it. We are infra gurus and through years of
experience have learned that building an automation for such a task can save us a lot of toil and manual labor.

Our operator will take care of the following: 
1. create a new instance of the ghost application as a website in our cluster if our cluster doesn't have it already
2. update our ghost application when our ghost application custom resource is updated.
3. delete the ghost application upon request 

Kubebuilder provides a command that allows us to create a custom resource and a process that keeps maintaing (reconciling) that resouce.
If we choose to create a new resouces (let's call it `Ghost`) kubebuilder will create a blog controller for it automatically.
If we want to attach our own controllers to the exisiting k8s resources say `Pods` that's posssible too! :D 

```shell
kubebuilder create api \
  --kind Ghost \
  --group blog \
  --version v1 \
  --resource true \
  --controller true
```
At this stage, Kubebuilder has wired up two key components for your operator:

A Resource in the form of a Custom Resource Definition (CRD) with the kind `Ghost`.
A Controller that runs each time a `Ghost` CRD is create, changed, or deleted.

The command we ran added a Golang representation of the `Ghost` Custom Resource Definition (CRD) to our operator scaffolding code.
To view this code, navigate to your Code editor tab under `api` > `v1` > `ghost_types.go`.

Let's have a look at the `type GhostSpec struct`. 
This is the code definition of the Kubernetes object spec. This spec contains a field named `foo` which is defined in `api/v1/ghost_types.go:32`. 
There is even a helpful comment above the field describing the use of foo.


now let's see how kubebuilder can generate a yaml file for our `Custom Resource Definition`
```shell
make manifests
```
you will find the generated crd at `config/crd/bases/blog.example.com_ghosts.yaml`
see how kubebuilder did all the heavylifting we had to previously do for the crontab example! lovely!


## Now let's install the CRD into our cluster
let's notice the difference by looking at our kubernetes 