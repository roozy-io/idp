# Extending k8s api like a pro.

# Links
* [Video by Shahrooz Aghili](https://www.youtube.com/watch?v=)

# Learning Resources

1. [Extending k8s](https://kubernetes.io/docs/concepts/extend-kubernetes/)

# Notes

> Kubernetes is highly configurable and extensible. As a result, there is rarely a need to fork or submit patches to the Kubernetes project code.
 
> Kubernetes is designed to be automated by writing client programs. Any program that reads and/or writes to the Kubernetes API can provide useful automation. Automation can run on the cluster or off it.

> There is a specific pattern for writing client programs that work well with Kubernetes called the controller pattern. Controllers typically read an object's `.spec`, possibly do things, and then update the object's `.status`.

> Extension Points
<img src="../assets/extension-points.png" alt="K8s Extension Points" width="100%">

1. Users often interact with the Kubernetes API using kubectl. Plugins customise the behaviour of clients. There are generic extensions that can apply to different clients, as well as specific ways to extend kubectl.

2. The API server handles all requests. Several types of extension points in the API server allow authenticating requests, or blocking them based on their content, editing content, and handling deletion. These are described in the API Access Extensions section.

3. The API server serves various kinds of resources. Built-in resource kinds, such as pods, are defined by the Kubernetes project and can't be changed. Read API extensions to learn about extending the Kubernetes API.

4. The Kubernetes scheduler decides which nodes to place pods on. There are several ways to extend scheduling, which are described in the Scheduling extensions section.

5. Much of the behavior of Kubernetes is implemented by programs called controllers, that are clients of the API server. Controllers are often used in conjunction with custom resources. Read combining new APIs with automation and Changing built-in resources to learn more.

6. The kubelet runs on servers (nodes), and helps pods appear like virtual servers with their own IPs on the cluster network. Network Plugins allow for different implementations of pod networking.

7. You can use Device Plugins to integrate custom hardware or other special node-local facilities, and make these available to Pods running in your cluster. The kubelet includes support for working with device plugins.

The kubelet also mounts and unmounts volume for pods and their containers. You can use Storage Plugins to add support for new kinds of storage and other volume types.
