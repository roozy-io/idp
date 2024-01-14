# GitOps and Argo CD

## Links

* [Video by Shahrooz Aghili on Argo CD](https://youtu.be/YjW_EL9rqCg)
* [Argo CD](https://argoproj.github.io/cd/)
* [Docs](https://argo-cd.readthedocs.io/en/stable/)
* [The DevOps Toolkit](https://www.youtube.com/c/DevOpsToolkit/videos) has several great explanations and tutorials around GitOps and ArgoCD

## Why Argo CD

> **Argo CD** represents a significant leap in the domain of DevOps, particularly in the adoption of GitOps principles.

Argo CD is a part of the Argo project and a [CNCF Graduated project](https://landscape.cncf.io/?selected=argo).

![Pull vs. Push](../assets/image.png)

## Pull vs. Push Model in GitOps

* **Enhanced Security and Autonomy**:
  * Pull model enhances security by reducing attack vectors.
  * Allows for cluster autonomy by applying changes internally.

* **Self-Management and Consistency**:
  * Clusters are self-managed, updating without external dependencies.
  * Ensures a consistent state with the repository.

* **Resilience and Reduced Credential Exposure**:
  * Pull systems are resilient, not relying on external services.
  * Minimizes credential exposure outside the cluster.

* **Scalability and Dynamic Updates**:
  * Better scalability by offloading work to individual clusters.
  * Supports event-driven, dynamic updates.

* **Operational Efficiency**:
  * Reduces overhead on CI systems, with the cluster managing deployment work.
  * Leads to lower operational complexity.

## Benefits

* **Robust Security and Automated Synchronization**:
  * Utilizes Git's security features for safe deployments.
  * Automatically syncs states as defined in Git.

* **Ease of Operations**:
  * Facilitates easy rollbacks.
  * Encourages declarative infrastructure setup.
  * Simplifies complex deployments.

* **Cost-Effective and Community-Driven**:
  * Open-source nature allows free usage and community contributions.

* **Real-Time Oversight and Multi-Cluster Management**:
  * Offers real-time application status monitoring.
  * Manages deployments across multiple clusters.

* **Self-Healing and Community Support**:
  * Automatically corrects state drifts.
  * Benefits from the support and innovation of the GitOps community.

[Hands on Exercise](https://killercoda.com/shahrooz33ce/scenario/argo_cd_intro)
