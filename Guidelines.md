# Operator Guidelines

Kubernetes provides ability to extend a cluster's functionality by adding new Operators (Custom
Resource Definitions + associated controllers). Such an extended Kubernetes cluster essentially 
represents a purpose-built platform.

Here we provide guidelines for developing Kubernetes Operators that help in
bringing consistency when multiple such Operators are used 
in a single Kubernetes cluster to form a purpose-built platform.

We have come up with these guidelines based on our study of various Operators
written by the community and through our experience of building a 
[discovery and provenance tool for Kubernetes.](https://github.com/cloud-ark/kubediscovery)


## 1) Use declarative specification as the state update mechanism for Custom Resources

Define the desired states of a Custom Resource as declarative Spec in its Type definition.
Any updates to the state of a Custom Resource instance should be defined solely as the desired
state in the declarative Spec of the resource.
Users should not be concerned with the procedural details of specifying changes from the previous state.
For example, to add a new user to a Postgres custom resource, 
users should just update the yaml definition of Postgres resource instance adding a 
[new name in the users list](https://github.com/cloud-ark/kubeplus/blob/master/postgres-crd-v2/artifacts/examples/add-user.yaml)


## 2) Implement Custom Controller to use diff-based logic for updating Custom Resource State

Custom controller code should be written such that it reconciles the current state
with the desired state as defined in the Custom resource's Spec by identifying a diff
of the current state with the desired state. Life-cycle actions should be embedded in the controller
logic when it reconciles actual state to desired state.
The controller code should be written to perform diff of the current users with the desired user 
and perform the required actions (such as add the new user) based on the received desired state.

Note that this means you should avoid using something like [JSON PATCH](https://tools.ietf.org/html/rfc6902#section-4) in the Spec of Custom Resource. This is because it will make your Custom Resource Specs
[non-repeatable and non-shareable](https://medium.com/@cloudark/evolution-of-paases-to-platform-as-code-in-kubernetes-world-74464b0013ca).


## 3) Use OwnerReference on Custom Resource instances

A custom controller will typically create one or more Kubernetes resources, such as Pod, Service, Deployment, Secret, Ingress, etc., as part of instantiation of its custom resource. It is important for a user to understand this underlying composition of a custom resource. The controller should be written to set OwnerReference on custom
or native Kubernetes Kinds that it would create. These references aid with supporting
[discovery of information](https://github.com/cloud-ark/kubediscovery), such as the Object composition tree, for custom resource instances.


## 4) Use ConfigMaps or Annotations for setting underlying resource's configuration parameters 

A controller should be written such that it takes inputs for underlying resource's
configuration parameters through ConfigMap(s) or Annotations. 
The ConfigMap/Annotation can be used for customizing some selection of configuration
parameters of the underlying resource (e.g.: [Nginx Operator](https://github.com/nginxinc/kubernetes-ingress/tree/master/examples/customization)). Or you can use ConfigMap to pass in an entire
configuration file (e.g.: [MySQL Operator](https://github.com/oracle/mysql-operator/blob/master/docs/user/clusters.md)).



## 5) Use ConfigMap to pass parameters that modify the behavior of the Controller

A Custom controller will typically support some form of customization. For example, 
[this MySQL Operator](https://github.com/oracle/mysql-operator/blob/master/docs/tutorial.md#configuration) supports following customization settings: whether to deploy
the Operator cluster-wide or within a particular namespace, which version of MySQL should be installed, etc.
Use ConfigMap for such passing in such customization parameter values.


## 6) Expose Monitoring hooks for Custom Controller

Your controller would run as a Deployment/ReplicaSet in a Cluster. You would want to monitor the health and
status of your controller. One way to perform such monitoring would be to expose 
Prometheus monitoring metrics for your controller. 
You can see this in the [MySQL Operator](https://github.com/oracle/mysql-operator/blob/master/docs/setup/monitoring.md).


## 7) Use kube-openapi annotations in Custom Resource Type definition

When defining the types corresponding to your custom resources, you should use
[kube-openapi annotation](https://github.com/kubernetes/kube-openapi/issues/96) of ``+k8s:openapi-gen=true'' 
in the type definition to [enable generating documentation for the custom resource.](https://medium.com/@cloudark/understanding-kubectl-explain-9d703396cc8)


## 8) Record events in the Controller using Event Recorder

When developing your controller use Kubernetes's in-built event recording tools to
[record the events and actions](https://github.com/cloud-ark/kubeplus/blob/master/postgres-crd-v2/controller.go#L414) that your controller takes. This information will aid in generating audit log / provenance
for the custom resource instances in your cluster.

Apart from these you should follow the [general guidelines](https://github.com/kubernetes/community/blob/master/contributors/devel/controllers.md) for developing Kubernetes controllers.
