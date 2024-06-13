# Kubernetes Webhooks.

## Links

1. [Video by Shahrooz Aghili](https://youtu.be/)
2. [Killercoda Hands-on Lab](https://killercoda.com/aghilish/scenario/k8s_webhooks)

## Learning Resources

1. [Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
2. [Dynamic Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
3. [Kubebuilder Docs](https://kubebuilder.io/cronjob-tutorial/webhook-implementation)

# Notes

> Kubernetes is highly configurable and extensible. As a result, there is rarely a need to fork or submit patches to the Kubernetes project code.

> Admission Controllers [Details](https://kubernetes.io/docs/concepts/extend-kubernetes/#key-to-the-figure)
<img src="../assets/extension-points.png" alt="k8s Extension Points" width="100%">

## 1. A Look into Custom Resource Definition (CRD) API

We need a test cluster. Kind is a good option.
```shell
kind create cluster
```