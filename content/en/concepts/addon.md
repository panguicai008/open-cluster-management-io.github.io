---
title: Add-ons
weight: 4
---

## What is an add-on?

Open-cluster-management has a built-in mechanism named [addon-framework](https://github.com/open-cluster-management-io/addon-framework)
to help developers to develop an extension based on the foundation components 
for the purpose of working with multiple clusters in custom cases. A typical
addon should consist of two kinds of components:

- __Addon Agent__: A kubernetes controller *in the managed cluster* that manages
  the managed cluster for the hub admins. A typical addon agent is expected to 
  be working by subscribing the prescriptions (e.g. in forms of CustomResources) 
  from the hub cluster and then consistently reconcile the state of the managed
  cluster like an ordinary kubernetes operator does.
  
- __Addon Manager__: A kubernetes controller *in the hub cluster* that applies 
  manifests to the managed clusters via the [ManifestWork](../manifestwork)
  api. In addition to resource dispatching, the manager can optionally manage
  the lifecycle of CSRs for the addon agents or even the RBAC permission bond
  to the CSRs' requesting identity.
  
In general, if a management tool working inside the managed cluster needs to
discriminate configuration for each managed cluster, it will be helpful to model 
its implementation as a working addon agent. The configurations for each agent
are supposed to be persisted in the hub cluster, so the hub admin will be able
to prescribe the agent to do its job in a declarative way. In abstraction, via
the addon we will be decoupling a multi-cluster control plane into (1) strategy
dispatching and (2) execution. The addon manager doesn't actually apply any
changes directly to the managed cluster, instead it just places its prescription
to a dedicated namespace allocated for the accepted managed cluster. Then the 
addon agent pulls the prescriptions consistently and does the execution.

In addition to dispatching configurations before the agents, the addon manager 
will be automatically doing some fiddly preparation before the agent bootstraps,
such as:

- CSR applying, approving and signing.
- Injecting and managing client credentials used by agents to access the hub
  cluster.
- The RBAC permission for the agents both in the hub cluster or the managed
  cluster.
- Installing strategy.

## Architecture

The following architecture graph shows how the coordination between addon manager
and addon agent works.

<div style="text-align: center; padding: 20px;">
   <img src="/addon-architecture.png" alt="Addon Architecture" style="margin: 0 auto; width: 80%">
</div>

## Add-on enablement

From a user's perspective, to install the addon to the hub cluster the hub admin 
should register a globally-unique `ClusterManagementAddon` resource as a singleton 
placeholder in the hub cluster. For instance, the [helloworld](https://github.com/open-cluster-management-io/addon-framework/tree/main/examples/helloworld) 
add-on can be registered to the hub cluster by creating:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  addOnMeta:
    displayName: helloworld
```

The addon manager running on the hub is taking responsibility of configuring the
installation of addon agents for each managed cluster. When a user wants to enable 
the add-on for a certain managed cluster, the user should create a 
`ManagedClusterAddon` resource on the cluster namespace. The name of the 
`ManagedClusterAddon` should be the same name of the corresponding 
`ClusterManagementAddon`. For instance, the following example enables `helloworld` 
add-on in "cluster1":

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddon
metadata:
  name: helloworld
  namespace: cluster1
spec:
  installNamespace: helloworld
```

Last but not least, a neat uninstallation of the addon is also supported by simply
deleting the corresponding `ClusterManagementAddon` resource from the hub cluster
which is the "root" of the whole addon. The OCM platform will automatically sanitize
the hub cluster for you after the uninstalling by removing all the components either
in the hub cluster or in the manage clusters.

### Add-on healthiness

The healthiness of the addon instances are visible when we list the addons via 
kubectl:

```shell
$ kubectl get managedclusteraddon -A
NAMESPACE   NAME                     AVAILABLE   DEGRADED   PROGRESSING
<cluster>   <addon>                  True                   
```

The addon agent are expected to report its healthiness periodically as long as it's
running. Also the versioning of the addon agent can be reflected in the resources 
optionally so that we can control the upgrading the agents progressively.


## Examples

Here's a few examples of cases where we will need add-ons:

1. A tool to collect alert events in the managed cluster, and send to the hub 
   cluster.
2. A network solution that uses the hub to share the network info and establish 
   connection among managed clusters. See [cluster-proxy](https://github.com/open-cluster-management-io/cluster-proxy)
3. A tool to spread security policies to multiple clusters.

## Add-on framework

[Add-on framework](https://github.com/open-cluster-management-io/addon-framework) 
provides a library for developers to develop an add-ons in open-cluster-management 
more easily. Take a look at the [helloworld example](https://github.com/open-cluster-management-io/addon-framework/tree/main/examples/helloworld) 
to understand how the add-on framework can be used.

### Custom signers

The original Kubernetes CSR api only supports three built-in signers:

- "kubernetes.io/kube-apiserver-client"
- "kubernetes.io/kube-apiserver-client-kubelet"
- "kubernetes.io/kubelet-serving"
  
However in some cases, we need to sign additional custom certificates for the 
addon agents which is not used for connecting any kube-apiserver. The addon 
manager can be serving as a custom CSR signer controller based on the 
addon-framework's extensibility by implementing the signing logic. Note that 
after successfully signing the certificates, the framework will also keep 
rotating the certificates automatically for the addon.


### Hub credential injection

The addon manager developed base on [addon-framework](https://github.com/open-cluster-management-io/addon-framework)
will automatically persist the signed certificates as secret resource to the 
managed clusters after signed by either original Kubernetes CSR controller or 
custom signers. The injected secrets will be:

- For "kubernetes.io/kube-apiserver-client" signer, the name will be "<addon name>
  -hub-kubeconfig" with properties:
  - "kubeconfig": a kubeconfig file for accessing hub cluster with the addon's 
    identity.
  - "tls.crt": the signed certificate.
  - "tls.key": the private key.
- For custom signer, the name will be "<addon name>-<signer name>-client-cert"
  with properties:
  - "tls.crt": the signed certificate.
  - "tls.key": the private key.

### Auto-install by cluster discovery

The addon manager can automatically install an addon to the managed clusters
upon discovering new clusters by setting the `InstallStrategy` from the 
[addon-framework](https://github.com/open-cluster-management-io/addon-framework).
On the other hand, the admin can also manually install the addon for the 
clusters by applying `ManagedClusterAddon` into their cluster namespace.
