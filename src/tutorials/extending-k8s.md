# Extending k8s api like a pro.

# Links
* [Video by Shahrooz Aghili](https://www.youtube.com/watch?v=)

# Learning Resources

1. [Extending k8s](https://kubernetes.io/docs/concepts/extend-kubernetes/)

# Notes

> Kubernetes is highly configurable and extensible. As a result, there is rarely a need to fork or submit patches to the Kubernetes project code.
 
> Kubernetes is designed to be automated by writing client programs. Any program that reads and/or writes to the Kubernetes API can provide useful automation. Automation can run on the cluster or off it.

> There is a specific pattern for writing client programs that work well with Kubernetes called the controller pattern. Controllers typically read an object's `.spec`, possibly do things, and then update the object's `.status`.

> Extension Points [Details](https://kubernetes.io/docs/concepts/extend-kubernetes/#key-to-the-figure)
<img src="../assets/extension-points.png" alt="K8s Extension Points" width="100%">

# In this tutorial we will be focusing on extension point 2 (Kubernetes API).

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

Delete custom resource 
```shell
kubectl delete CronTab my-new-cron-object
```
