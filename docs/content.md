# Hosted Control Plane Clusters
Advanced Cluster Management for Kubernetes can deploy OpenShift clusters with two different control plane paradigms.  Standalone, uses a virtual machine or physical machine to host the OpenShift control plane.  Starting with ACM 2.5, support for provisioning Hosted Control Planes is supported.  What this form of provisioning does, is provision the OpenShift control plane as pods on a Hosting Service Cluster.  The Hosting Service Cluster, can be the ACM Hub or one of the OpenShift clusters under Hub management.

The Control Plane is run as pods, contained in a single namespace associated to the Hosted Control Plane Cluster.  This hosted cluster type of OpenShift, then provisions its worker node independent of the control plane, with support for AWS, Azure, Kubevirt and Bare Metal.

Benefits of Hosted Control Plane Clusters:
  * Saves cost, by removing the need to host independent control plane nodes (virtualized)
  * Creates a distinct division of control between the control plane and the worker nodes. Control plane can run in one account or cloud and the workers in another
  * Decreases provisioning time to ~10min
  * Highly available upgrades
  * Cross platform hosting (control plane can run in a different provider then the workers, cost savings and availability)
  * Fully configurable OpenShift
  * Turnkey deployments or fully customized OpenShift provisioning

# Supported platforms
* AWS
* Azure
* Bare Metal
* Kubevirt

# Key terms (API)
[API reference link](tba)

### HypershiftDeployment:
    The HypershiftDeployment kind is the entry point to Hosted Control Plane clusters. It offers both turn key and custom provisioning of OpenShift clusters. This resource contains details about the infrastructure, hosted control plane and node pools. Its status reflects the health and availability of the system.

### HostedCluster:
    The HostedCluster kind is the custom resource that represents the Hosted Control Plane. The `Spec` for this resource is initially populated from the HypershiftDeployment resource. This resource has all the control plane configuration options and references. The creation, update and deletion of this resource directly affects the OpenShift control plane for a cluster. The control plane includes etcd, OpenShift API server, etc.

### NodePools:
    The NodePool kind is the custom resource that represents the pool of worker nodes in an OpenShift cluster. You can have zero or more node pools, each with different worker node variables (configurations). This `Spec` for this resource is continually populated from the HypershiftDeployment resource.

### Hosting Service Cluster:
    A cluster designated to host control planes. Any cluster managed by ACM, and is a supported platform, can be activated as a Hosting Service Cluster (including the Hub)

### Hypershift-operator:
    This is a controller that runs on the Hosting Service Cluster and facilitates the lifecycle of a HostedCluster's control plane

### Hypershift-addon-controller:
    This is responsible for lifecycling the Hypershift-operator on the Hosting Service Clusters

### ManifestWork:
    This is a custom resource kind that is created by the hypershfit-deployment-controller. It uses details in the HypershiftDeployment custom resource, to deliver the HostedCluster and NodePool custom resources to the Hosting Service Cluster. The delivered resources enable the Hypershift-operator to provision Hosted Control Plane clusters.

![HypershiftDeployment flow](./hostedcontrolplanecluster-flow.jpg)

# Execution flow
1. Create a `HypershiftDeployment` kind custom resource
2. In processing the HypershiftDeployment, the infrastructre specified may be configured
   * VPC's, resource groups, are created and configured
   * These operations can be skipped and the details of the infrastructure resources are instead provided in the HypershiftDeployment resource
3. A ManifestWork kind custom resource is created, in its payload you find:
    * HostedCluster resource
    * Zero or more NodePool resources
    * ConfigMaps and Secrets used to configure and customize the OpenShift deployment
4. The ManifestWork applies the payload to the Hosted Service Cluster that was specified in the HypershiftDeployment custom resource
5. The Hypershift-operator detects the HostedCluster and NodePool custom resources and provisions the OpenShift cluster
6. The ManifestWork tracks the status of HostedCluster and NodePool custom resources and resturns that information to the Advanced Cluster Management Hub
7. Once the Hypershift-operator has deployed OpenShift, the new cluster is automatically imported into ACM
8. When the HypershiftDeployment resource is updated, the changes are made to the ManifestWork, which applies those changes to the Hosting Service Cluster, in affect modifying the Hosted Control Plane Cluster.
    * Grow and shrink existing node pools
    * Create or remove node pools
    * Update the version of OpenShift for the Control Plane
    * Update the version of OpenShift for each Node Pool
9. Delete of the HypershiftDeployment resource, this causes the ManifestWork to delete the HostedCluster and NodePool(s) custom resources. This deprovisions the OpenShift cluster

# Infrastructure Configuration turn key
When deploying to AWS or Azure, it is possible to have the ACM Hub to create the needed Cloud Provider infrastructure for the OpenShift deployment. The following parameters are available in the HypershiftDeployment.Spec
| Parameter path     | Descritpion                                       | Default | Required |
| ---------------- | ----------------------------------------------- | ----- | ------ |
| `hostingNamespace` | This is the namespace on the Hosting Service Cluster where the ManifestWork will create the HostedCluster, NodePools, configMaps and Secrets | If not provided, the namespace of the HypershiftDeployment custom resource is used | X |
| `hostingCluster`   | The name of the Hosting Service Cluster where an instance of OpenShift will be deployed | None | X|
| `override`         | This allows for special cases: `ORPHAN` The ManifestWork items are left behind. `INFRA-ONLY` configures infrastructure, but does not create a ManifestWork | None | |
|`infrastructure.cloudProvider.name` | This is the ACM Cloud Provider secret name, this is used when `configure: True` is chosen. It is a credential composed by ACM for AWS or Azure | None | X * |
| `infrastructure.configure` | When `True` ACM will configure the AWS or Azure infrastructure to prepare for an OpenShift provisioning. When `False` the user must provide the infrastructure details to ACM via the `HosteClusterSpec` and `NodePoolSpec`. When `False` the `infrastructure.cloudProvider.name` is not required unless using Azure | None | x |
| `platform.aws.region` | When using AWS, this is the region where the infrastructure for the control plane exists or will be created | None | X |
| `platform.azure.location` | When using Azure, this is the location where the infrastructure for the control plane exists or will be created | None | X |
* Not required when `configure: False`, Link to ACM Cloud Provider credentials

# Monitoring deployment status
The output from the HypershiftDeployment custom resource gives you the major details to monitor provisioning.
It provides status during a get for:
* Infrastructure configuration (`configure: True`)
* Infrastructure IAM configuration (`configure: True`)
* ManifestWork creation and application
* HostedCluster progress
* HostedCluster availability (api server)

```shell
oc -n PROJECT_NAME get hypershiftDeployment NAME
```

There is further details available, including node pool status via the describe command
```shell
oc -n PROJECT_NAME describe hypershiftDeployment NAME
```