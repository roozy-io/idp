# Kubernetes Webhooks.

## Links

1. [Video by Shahrooz Aghili](https://youtu.be/)
2. [Killercoda Hands-on Lab](https://killercoda.com/aghilish/scenario/k8s_webhooks)

## Learning Resources

1. [Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
2. [Dynamic Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
3. [Kubebuilder Docs](https://kubebuilder.io/cronjob-tutorial/webhook-implementation)

## What are Admission Controllers

> An admission controller is a piece of code that intercepts requests to the Kubernetes API server prior to persistence of the object, but after the request is authenticated and authorized. Admission controllers may be validating, mutating, or both. Mutating controllers may modify objects related to the requests they admit; validating controllers may not. Admission controllers limit requests to create, delete, modify objects. Admission controllers do not (and cannot) block requests to read (get, watch or list) objects. 

> The admission controllers in Kubernetes 1.30 consist of the list below, are compiled into the kube-apiserver binary, and may only be configured by the cluster administrator.

## Why do we need them ?
> Why do I need them? Several important features of Kubernetes require an admission controller to be enabled in order to properly support the feature. As a result, a Kubernetes API server that is not properly configured with the right set of admission controllers is an incomplete server and will not support all the features you expect.

## How can we turn them on ?
```bash
kube-apiserver --enable-admission-plugins=NamespaceLifecycle,LimitRanger ...
```
## Which ones are enabled by default ?
```bash
# to get the list we can run 
kube-apiserver -h | grep enable-admission-plugins
# these are enabled by default

#CertificateApproval, CertificateSigning, CertificateSubjectRestriction, DefaultIngressClass, DefaultStorageClass, DefaultTolerationSeconds, LimitRanger, MutatingAdmissionWebhook, NamespaceLifecycle, PersistentVolumeClaimResize, PodSecurity, Priority, ResourceQuota, RuntimeClass, ServiceAccount, StorageObjectInUseProtection, TaintNodesByCondition, ValidatingAdmissionPolicy, ValidatingAdmissionWebhook

```

## Dynamic Admission Control
> In addition to compiled-in admission plugins, admission plugins can be developed as extensions and run as webhooks configured at runtime. This page describes how to build, configure, use, and monitor admission webhooks.

## What are Admission Webhooks
> Admission webhooks are HTTP callbacks that receive admission requests and do something with them. You can define two types of admission webhooks, validating admission webhook and mutating admission webhook. Mutating admission webhooks are invoked first, and can modify objects sent to the API server to enforce custom defaults. After all object modifications are complete, and after the incoming object is validated by the API server, validating admission webhooks are invoked and can reject requests to enforce custom policies.

> Create a figure for admission webhook request (admission review)
> Create a figure for admission webhook response (admission review)
> Admission Webhooks can run inside or outside the cluster. If We deploy them inside the cluster, we can leverage cert manager for automatically inject certificate


> Admission Controllers [Details](https://kubernetes.io/docs/concepts/extend-kubernetes/#key-to-the-figure)
<img src="../assets/extension-points.png" alt="k8s Extension Points" width="100%">

## TODO: after deploying a webhook, let's create an object where the admission webhook (validating and mutating is called) and inspect the logs of the kube-api server or look at the etcd for the admission review request and respons


## 1. Create a webhook for our operator

We need a test cluster. Kind is a good option.
```shell
kind create cluster
```

let's pick up where we left off last time by cloning into our ghost operator tutorial.

```bash
git clone https://github.com/aghilish/operator-tutorial.git
cd operator-tutorial
```

let's create our first webhook
```bash
kubebuilder create webhook --kind Ghost --group blog --version v1 --defaulting --programmatic-validation
```
Let's have a look at the `api/v1/ghost_webhook.go` file, you should notice that some boilerplate code was generated for us to implement Mutating and Validating webhook logic.

Next we are gonna add a new field to our ghost spec called `replicas`. As you might have guessed this field is there to set the number of replicas on the managed deployment resource which we set on our ghost resource.

ok let's add the following line to our `GhostSpec` struct `api/v1/ghost_types.go:30`.

```go
//+kubebuilder:validation:Minimum=1
Replicas int32 `json:"replicas"`
```
Please note the kubebuilder marker. It validates the replicas value to be at least 1. But it's an optional value.
This validation marker will then translate into our custom resource definition. This type of validation is called declarartive validation.
With `Validating Admission Webhooks` we can validate our resources in a programmatic way meaning the webhook can return errors with custome messages if programmatic validation fails. In our `Mutating Validation Webhook` we can mutate our resource or set a default for replias if nothing is set on the custom resource manifest. 


## 2. Update controller

Before we implement the validation logic let us handle the new replicas field in our controller.

let's render new manifests first

```bash
make manifests
```
Now are CRD is updated.

and update our controller as follows.
Update `generateDesiredDeployment` at `internal/controller/ghost_controller.go:243`

with the following
```go
replicas := ghost.Spec.Replicas
```

and the update condition at `addOrUpdateDeployment` at `internal/controller/ghost_controller.go:243`
```go
existingDeployment.Spec.Template.Spec.Containers[0].Image != desiredDeployment.Spec.Template.Spec.Containers[0].Image ||
			*existingDeployment.Spec.Replicas != *desiredDeployment.Spec.Replicas
```

and let's make sure we don't have any build error by running 
```bash
make
```

## 3. Implement Webhook Logic
Next we are gonna add a defaulting and validating logic to our admission webhook. If the replicas is set to 0 we set it to 2 and if during create or update the replicas is set to a value bigger than 5 the validation fails. let's replace the content of our webhook at `api/v1/ghost_webhook.go` with the following.

```go
package v1

import (
	"fmt"

	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/webhook"
	"sigs.k8s.io/controller-runtime/pkg/webhook/admission"
)

// log is for logging in this package.
var ghostlog = logf.Log.WithName("ghost-resource")

// SetupWebhookWithManager will setup the manager to manage the webhooks
func (r *Ghost) SetupWebhookWithManager(mgr ctrl.Manager) error {
	return ctrl.NewWebhookManagedBy(mgr).
		For(r).
		Complete()
}

// TODO(user): EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!

//+kubebuilder:webhook:path=/mutate-blog-example-com-v1-ghost,mutating=true,failurePolicy=fail,sideEffects=None,groups=blog.example.com,resources=ghosts,verbs=create;update,versions=v1,name=mghost.kb.io,admissionReviewVersions=v1

var _ webhook.Defaulter = &Ghost{}

// Default implements webhook.Defaulter so a webhook will be registered for the type
func (r *Ghost) Default() {
	ghostlog.Info("default", "name", r.Name)

	if r.Spec.Replicas == 0 {
		r.Spec.Replicas = 2
	}
}

// TODO(user): change verbs to "verbs=create;update;delete" if you want to enable deletion validation.
//+kubebuilder:webhook:path=/validate-blog-example-com-v1-ghost,mutating=false,failurePolicy=fail,sideEffects=None,groups=blog.example.com,resources=ghosts,verbs=create;update,versions=v1,name=vghost.kb.io,admissionReviewVersions=v1

var _ webhook.Validator = &Ghost{}

// ValidateCreate implements webhook.Validator so a webhook will be registered for the type
func (r *Ghost) ValidateCreate() (admission.Warnings, error) {
	ghostlog.Info("validate create", "name", r.Name)
	return validateReplicas(r)
}

// ValidateUpdate implements webhook.Validator so a webhook will be registered for the type
func (r *Ghost) ValidateUpdate(old runtime.Object) (admission.Warnings, error) {
	ghostlog.Info("validate update", "name", r.Name)
	return validateReplicas(r)
}

// ValidateDelete implements webhook.Validator so a webhook will be registered for the type
func (r *Ghost) ValidateDelete() (admission.Warnings, error) {
	ghostlog.Info("validate delete", "name", r.Name)

	// TODO(user): fill in your validation logic upon object deletion.
	return nil, nil
}

func validateReplicas(r *Ghost) (admission.Warnings, error) {
	if r.Spec.Replicas > 5 {
		return nil, fmt.Errorf("ghost replicas cannot be more than 5")
	}
	return nil, nil
}

```

## 4. Deploy webhook
Once our webhook is implemented, all that’s left is to create the `WebhookConfiguration` manifests required to register our webhooks with Kubernetes. The connection between the kubernetes api and our webhook server needs to be secure and encrypted. This can easily happen if we use certmanager togehter with the powerful scaffolding of kubebuilder.

We need to enable the cert-manager deployment via kubebuilder, in order to do that we should edit `config/default/kustomization.yaml` and `config/crd/kustomization.yaml` files by uncommenting the sections marked by [WEBHOOK] and [CERTMANAGER] comments.

We also add a new target to our make file for installing cert-manager using a helm command.

So let's add the following to the botton of our make file.

```bash
##@ Helm

HELM_VERSION ?= v3.7.1

.PHONY: helm
helm: ## Download helm locally if necessary.
ifeq (, $(shell which helm))
        @{ \
        set -e ;\
        curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash ;\
        }
endif

.PHONY: install-cert-manager
install-cert-manager: helm ## Install cert-manager using Helm.
		helm repo add jetstack https://charts.jetstack.io
		helm repo update
		helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.15.0 --set crds.enabled=true

.PHONY: uninstall-cert-manager
uninstall-cert-manager: helm ## Uninstall cert-manager using Helm.
		helm uninstall cert-manager --namespace cert-manager
		kubectl delete namespace cert-manager
```