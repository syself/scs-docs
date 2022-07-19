---
title: Standardizing all parts of container layer with cluster stacks and an SCS KaaS operator
version: 2022-07-24-001
authors: Sven Batista Steinbach, Janis Kemper
state: Draft
---

# Theoretic background
## Parts of container layer to standardize

There are different parts of the container layer that can be standardized. Each of these parts, in theory, could be certified separately based on one of the four strategies above.

### Inputs to define a cluster

The inputs that the user takes to define a cluster can be standardized partly or fully.

### Cluster components

OS, CNI, etc. can be standardized and configured in a production-ready and optimal way, or only high-level decisions (should fulfill xyz) are taken.

### Cluster lifecycle

Kubeadm vs Talos, Cluster API vs Gardener

### Application configuration

For a standardized application configuration across different infrastructure providers, a standardized way of defining service objects and storage classes would be necessary.

# Our approach

With our approach to KaaS, we choose a standardization and certification approach that works with E2E tests and clearly defines the architecture and (software) components. We aim to standardize all relevant parts of the container layer that are specified above. This might be the approach that requires the most changes on the provider side, but is also the one that would generate the most benefit for those who commit to it. SCS can communicate concise standardizations and certifications to all stakeholders so that the overall project will profit. Users see the actual benefit and trust in SCS infrastructure. Clusters are from the user's perspective 100% independent of the provider. SCS develops production-ready and secure infrastructure developed with the combined knowledge of all providers. SCS provides guidance to the service providers and brings them value through implementing modern processes (automated E2E tests of updates, etc.).

# Main goals

- Provider-independent clusters for users
- Production-ready & secure clusters
- Testable and tested infrastructure

# Key architectural decisions

- Cluster stacks define a set of software components and configurations that are working well together and define a secure and production-ready Kubernetes cluster
- The process of creating the cluster based on a cluster stack and additional inputs of the user, as well as the whole lifecycle of the cluster is carried out by a Kubernetes operator

# Which role do stakeholders play?

## SCS

SCS defines cluster stacks and provides documentation on how to implement them. For users, there will be an overview of what a cluster stack contains. SCS provides a common Kubernetes API (i.e. CRDs) with all relevant objects. SCS provides a Golang library with functions that can be used to implement the operator (alternative: SCS implements the operator and providers have to fulfill only an interface. SCS provides further documentation on how to implement the operator for a provider.

### Service Providers

Service providers implement and release the cluster stacks. They implement the operator if they are responsible for that (one could say that all providers that implement a Cluster API Provider Integration also have to implement the SCS KaaS Operator). For both tasks, they use SCS documentation as a foundation. If they have some suggestions or improvements, SCS can take them into the account and improve the cluster stacks or the operator.

### Users

Users start clusters by installing the different components (CAPI-Operator and KaaS Operator SCS) into a management cluster. From there, they can use the objects of the SCS API to define target clusters in a declarative way. The Provider KaaS Operator (or operators when multiple providers are used at the same time) creates the desired target cluster.

# Deep dive cluster stacks

Cluster stacks are implemented by providers. There is a one-to-one relationship to CAPI provider integrations. If such a provider integration has to be implemented for a provider, they also need to implement cluster stacks. Cluster stacks consist of three parts. Scripts for provisioning node images, Kubernetes-related configuration, and minimal necessary applications.

- Node images are very provider-specific. They need to be implemented by a provider in a way that node images could be built from it. Right now, we know about three ways to build node images: with packer, which requires only the packer.json written by the provider, pre- and postKubeadmCommands from Cluster API (cloud-init), and upload of an already prepared image. All of these could work with the basic bash script commands. Some aspects of the node image (e.g. the MTU value), might depend on the service provider, even though the service provider uses OpenStack and OpenStack cluster stacks. In this case, there could be some additional configuration in the cluster stacks, so that the OpenStack cluster stack can take the MTU value as input.
- Kubernetes-related configurations like that of kubeadm are mostly independent of the provider and could therefore be passed through.
- Applications that are necessary for every production-ready cluster, e.g. a CNI, could also be passed through. If a provider needs to change, for example, the mtu value, he could overlay these specific configuration parts. Also, the provider needs to add his ccm.

Because cluster stacks follow a split responsibility model, security can be tackled by the community as a whole. Patches are possible within hours and rolled out to all providers. Cluster stacks, defined and implemented by SCS, leverage the possibility of penetration tests and of making security audits in a central place without the requirement to do it on every provider. 

Providers use the implemented cluster stack and only overlay with their provider-specific configuration. So an e2e test that checks whether all components in the release of a provider are still there would be enough to be compliant with the base cluster stack. 

Furthermore, the process of integrating and testing patches of a cluster stack could be easily automated and a release bot (like dependabot or renovatebot) could directly create a PR if a new cluster stack version is released by SCS. The PR would also directly start e2e tests. Therefore, in the best case, the PR could directly be merged and the cluster stack released. 

This procedure is possible because we assume that provider-specific configurations don’t change that often in the lifecycle of a cluster stack. So once set up, the cluster stack could get security updates from upstream. 

# Deep dive KaaS Operator

There is a one-to-one relationship between Cluster API Provider Integrations and KaaS operators. Each service provider that needs a CAPI provider integration also needs to implement a KaaS operator.

SCS tries to take as much load off from the service providers as possible. The support is three-fold: First, there is extensive documentation on how to write the KaaS operator. Then, there is a Go library with general, not provider-dependent functions that facilitate the implementation of the operators. This library is, of course, also mentioned in the documentation. Third, SCS extends the Kubernetes API with relevant CRDs. An example of what such an API looks like can be found in [https://github.com/kubernetes-sigs/cluster-api/tree/main/api/v1beta1](https://github.com/kubernetes-sigs/cluster-api/tree/main/api/v1beta1). 

We are currently evaluating another approach of writing the operators, where all of the main code is written by SCS. In that approach, the providers don’t actually have to write Kubernetes operators (with Reconcile functions, etc.), but they just have to write some functions to fulfill an interface. This approach would make life even easier for providers.

### Installing one or many KaaS operators

As there will be multiple KaaS operators (one for each CAPI provider integration), users have to choose which one(s) they want to install. This could be done via a CR similar to the Cluster API operator. It will be possible to have multiple operators running in parallel so that one management cluster is enough to deal with multiple target clusters on different service providers.

The co-existence of multiple operators works as follows: If the user applies an object containing his/her inputs to define a cluster (the KaaS object), there is also one parameter specifying the provider. The respective operator of this provider watches all of these objects and creates/updates the cluster. This behavior is actually similar to the Ingress API, where there can be multiple ingress controllers simultaneously and one controller only reconciles an object if its name is referenced in the object’s specs.

### The KaaS custom resource

Apart from a reference to the provider, the object contains the reference to the cluster stack, it describes how the cluster should look like (number of control planes, machine deployments, etc.) and an advanced configuration, e.g. for OIDC. 

Kubernetes natively supports validating and defaulting for its objects. This is why we create an API with SCS KaaS objects that are relevant to everybody (i.e. the object that defines a target cluster). These objects can be validated through webhooks and defaults can be chosen. As the webhooks and defaulting logic are written in Go, they can be arbitrarily complex.

### Tasks of an operator

The operator of one provider takes over a lot of general logic implemented by SCS. However, it also has to do some provider-specific tasks that have to be implemented by the provider. Namely, to translate the generic user input into, for example, objects of the respective CAPI provider integration.

The operator manages the lifecycle of the cluster. One of the main duties is to manage the lifecycle of CAPI objects, including provider-specific ones. It creates new objects when an update happens or a new machine should be created. Updates usually happen for two reasons: The user changes the version of cluster stack or Kubernetes, or the user changes some configuration of his/her machines.

As part of the machines’ lifecycle, it also creates node images for new or updated machines. This process is usually heavily dependent on the provider and differs, for example, a lot between Hetzner and OpenStack.

Additionally, the operator manages the lifecycle of some mandatory applications that are specified in the cluster stack and that are necessary for a production-ready cluster. This part of the operator is independent of the provider and can be implemented fully by SCS. 

## Alternatives to an operator

The alternatives to an operator have to fulfill the same tasks as the operator: creating clusters from user input and managing their lifecycles. This can be done, in theory, with scripts or a non-operator approach with a programming language. The issue with all of these approaches is that not all logic is in the same position. Only an operator can do all the tasks needed by itself. Other approaches might have to work with an external Continuous Delivery system to trigger scripts or functions at the right point. 

Only an operator is designed to work with each eventuality that might happen in practice. All other approaches will create a mass of different cases and unrelated pieces of code to take care of them. In the case of scripts, this would not even be testable code at all.