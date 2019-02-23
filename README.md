# Openshift Operator Demos

### Introduction
This is a discussion around Kubernetes/Openshift [Operators](https://coreos.com/blog/introducing-operator-framework) including instructions and a recording of how to create a demonstration of them on Openshift, the world's leading commercial Kubernetes distribution.
An [Operator](https://coreos.com/blog/introducing-operator-framework) is a piece of software that runs on Kubernetes that embeds operational knowledge around your containerised applications. An operator allows you to automate and re-use that knowledge to manage your application on an ongoing basis. Tasks typically undertaken by operators include:
- when and how to upgrade your application as new underlying components become available
- application and data backup management
- application patching 
- application scaling - horizontal and vertical

An Operator is like an extension your engineering team that watches over the Kubernetes environment and uses its current state to make decisions in milliseconds.

Currently there are 3 Operator implemenation options levels of maturity and complexity

They are
- Helm (for running Helm charts in a secure way not requiring Tiller which needs root access and as such could be considered a security risk)
- Ansible - offering further control by calling out to Ansible playbooks and roles.
- Go - which could be considered the Gold Standard of operators - allowing you to embed your specific fine grained control logic.

![1-OperatorSDK.png](https://github.com/tnscorcoran/openshift-operators/blob/master/images/1-OperatorSDK.png)

You can see they feed into testing and verification facilities allowing them to be stored on a public Operator Hub, though this is out of scope for this article.

See also - the Operator Maturity Model - detailing the capabilities unleashed by the 3 operators. 
![2-Operator-Maturity-Model.png](https://github.com/tnscorcoran/openshift-operators/blob/master/images/2-Operator-Maturity-Model.png)



Today, I will take two of these operator types:
- a Helm based NGINX operator 
- and an Ansible based GOGs operator
and add context around them as well as building their respective demos.

Let's get started!

### 1 - Helm based NGINX operator
















==============================================================================================================================
In this article we will walk you through the steps to create an Ansible Operator

At a high level these are the steps we will follow

- Create Cluster Role and Cluster Role Binding
The Create Cluster Role creation also creates a new *API Group* [https://docs.openshift.com/container-platform/3.11/admin_guide/custom_resource_definitions.html] (a group of related objects in Openshift) called *etcd.database.coreos.com*. An API Group is required to create a new Custom Resource Definition in Kubernetes - required for an Operator.
It effectively defines the Resources (custom and built in) and permissions on those resources. 

TODO - mention what /example/rbac/role-template.yaml does

The Cluster Role Binding binds the resources and permissions defined in the Cluster Role to a Service account [https://docs.openshift.com/container-platform/3.11/dev_guide/service_accounts.html] - a non-human user account required by pods to execute calls on the Openshift API.


FROM: first sentence of: http://www.opentlc.com/operators/04_01_Writing_an_Operator_Lab.html
ADD: These to intro
	- "The Operator then watches for custom resource objects to be created - and when it sees a custom resource 	
	being created it creates the application based on the parameters in the custom resource object."
	- "Red Hat has provided a Software Development Kit that significantly simplifies the creation of Operators. 
	The SDK handles all the core logic while the developer only needs to fill in the actual business logic."
	- "In this lab you are using the Ansible Operator SDK to create an Operator using two Ansible Roles and a 
	Playbook that sets up a PostgreSQL Database and a Gogs Server that uses that database"
	
- install Operator .... future versions of Openshift starting at 4.0 wil use the *Operator Lifecycle Manager* but as this is Tech Preview in my Openshift version (3.11), I'll manually install it
Note it's of *Kind: Deployment*
oc create -f ./example/deployment.yaml

- When starting for the first time, the Operator will create a new CRD - of type *etcdclusters.etcd.database.coreos.com* viewable by running
oc get crds
Describe it:
oc describe crd etcdclusters.etcd.database.coreos.com


- now create an instance of our Custom Resource - in our case EtcdCluster.
oc create -f ./example/example-etcd-cluster.yaml

When we do this, the operator kicks into action and specifies as many EtcdClusters and the version our EtcdCluster Custom Resource Definition specifies.

-  






============

TODO: oc describe crd etcdclusters.etcd.database.coreos.com











### Demo Instructions


TO
