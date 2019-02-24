# Openshift Operator Demos

### Introduction
This is a discussion around Kubernetes/Openshift [Operators](https://coreos.com/blog/introducing-operator-framework) including instructions and a recording of how to create a demonstration of them.
We'll use Red Hat [Openshift](https://www.openshift.com/), the world's leading commercial Kubernetes distribution. As Openshift is a complete Kubernetes distribution, containing the Kube API and CLI, we'll use kubectl for most of our commands.

An [Operator](https://coreos.com/blog/introducing-operator-framework) is a piece of software that runs on Kubernetes that embeds operational knowledge around your containerised applications. An operator allows you to automate and re-use that knowledge to manage your application on an ongoing basis. Tasks typically undertaken by operators include:
- when and how to upgrade your application as new underlying components become available
- application and data backup management
- application patching 
- application scaling - horizontal and vertical

An Operator is like an extension your engineering team that watches over the Kubernetes environment and uses its current state to make decisions in milliseconds.

Currently there are 3 Operator implemenation options of varying levels of maturity and complexity

They are
- Helm (for running Helm charts in a secure way not requiring Tiller which needs root access and as such could be considered a security risk)
- Ansible - offering further control by calling out to Ansible playbooks and roles.
- Go - which could be considered the Gold Standard of operators - allowing you to embed your specific fine grained control logic.

![1-OperatorSDK.png](https://github.com/tnscorcoran/openshift-operators/blob/master/images/1-OperatorSDK.png)

You can see they feed into testing and verification facilities allowing them to be stored on a public Operator Hub, though this is out of scope for this article.

See also - the Operator Maturity Model - detailing the capabilities unleashed by the 3 operators: 
![2-Operator-Maturity-Model.png](https://github.com/tnscorcoran/openshift-operators/blob/master/images/2-Operator-Maturity-Model.png)

Today, I will take two of these operator types:
- a Helm based NGINX operator 
- and an Ansible based GOGs operator
and add context around them as well as building their respective demos.

Let's get started!

### 1 - Helm based NGINX operator

This discussion and demo is based on an existing [Helm demo](https://github.com/operator-framework/operator-sdk/blob/master/doc/helm/user-guide.md).
As shown in the diagrams above, the Helm operator is the simplest of the 3 Operator types supported by Openshift. That said, it's a great way to run Helm charts in Openshift without Tiller, which as mentioned, requires root access and as such can be considered a security risk.   
The following shows the steps we will follow:
![](https://github.com/tnscorcoran/openshift-operators/blob/master/images/3-nginx-helm-operator.png)

SSH into your RHEL box and set up Go Environment (version 1.11.2)

```
cd $HOME
mkdir -p $HOME/gopath/bin
echo "export GOPATH=$HOME/gopath" >> $HOME/.bashrc
echo "export PATH=$HOME/gopath/bin:/usr/local/go/bin:$PATH" >> $HOME/.bashrc
source $HOME/.bashrc

wget https://dl.google.com/go/go1.11.2.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.11.2.linux-amd64.tar.gz
rm -f $HOME/go1.11.2.linux-amd64.tar.gz
```

Install dep (Go Dependency Manager) and clean up any lingering operators
```
curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh
sudo rm -r $GOPATH/src/github.com/operator-framework
```

> Note future versions of Openshift starting at 4.0 will use the *Operator Lifecycle Manager* but as this is Tech Preview still (3.11), let's manually build our operators using the SDK

Download and build your Operator SDK. 
```
mkdir -p $GOPATH/src/github.com/operator-framework
cd $GOPATH/src/github.com/operator-framework
git clone https://github.com/operator-framework/operator-sdk
cd operator-sdk
git checkout master
make dep
make install
```

Build your local operator definition. note the *api-version* and *kind* indicate what the operator will watch for and act on

# !!!!!!!!! VERIFY THIS 
 
```
operator-sdk new nginx-operator --api-version=example.com/v1alpha1 --kind=Nginx --type=helm
cd nginx-operator
```
Modify your custom resource instance - setting replicaCount to 2 and port to 8080
```
vi deploy/crds/example_v1alpha1_nginx_cr.yaml
```

As a non-privileged, developer user and create a project 
```
oc login -u andrew
oc new-project nginx-operator
```
Now, as an administrator, create the Custom Resource Definition (or blueprint for the objects operator will act on)
```
oc login -u system:admin
kubectl create -f deploy/crds/example_v1alpha1_nginx_crd.yaml
```

Install and enable Docker

```
yum -y install docker
systemctl enable docker
systemctl start docker
```

We will create an image for our operator and push it to a registry to make it accessible. We'll use https://quay.io and my user is tnscorcoran. Modify as appropriate for your implementation.

```
docker login -u tnscorcoran quay.io
operator-sdk build quay.io/tnscorcoran/nginx-operator:v0.0.1
docker push quay.io/tnscorcoran/nginx-operator:v0.0.1
```

> 2 things to note. In quay.io:
	- Under Tags, check the 'v0.0.1' checkbox
	- Under Settings, make the repo public 


You will build your operator in Openshift using a Kubernetes Deployment object 
# !!!!!!!!! VERIFY THIS 
that references your newly pushed image in Quay. Modify your deployment object to reflect your image:

```
sed -i 's|REPLACE_IMAGE|quay.io/tnscorcoran/nginx-operator:v0.0.1|g' deploy/operator.yaml
```

As described in the Demo flow diagram above, we need to create a *Service Account* and a *Role* - meaning a non-human user and rights to perform actions, respectively. These 2 are joined and applied to our Operator in our *Role Binding*

> Note. This raw Kubernetes based example requires a small modification to run on Openshift. The Nginx application which the operator manages, requires root access. Openshift will not allow this by default - a prudent security feature of Openshift. As this is just a demo, we'll lift the restriction, running the command 'oc adm policy....'. In production, you should source images not requiring root access, like those in the Red Hat Container catalog. 

Run the following commands to setup your objects and customer resource our operator will watch.



```
kubectl create -f deploy/service_account.yaml
oc adm policy add-scc-to-user anyuid -z nginx-operator
kubectl create -f deploy/role.yaml
kubectl create -f deploy/role_binding.yaml
kubectl create -f deploy/operator.yaml
kubectl apply -f deploy/crds/example_v1alpha1_nginx_cr.yaml
```
# !!!!!!!!! VERIFY WHy we're watching these
```
kubectl get deployment
kubectl get pods
kubectl get service
```

Now change replicas to 3 and remove spec.service then apply that

```
vi deploy/crds/example_v1alpha1_nginx_cr.yaml
kubectl apply -f deploy/crds/example_v1alpha1_nginx_cr.yaml
```
Notice how the Openshift operator acts on your changes bring actual to new desired state. 

Now it's time to clean up. Note all of the dependant objects need to be deleted before we can delete our project.

```
kubectl delete -f deploy/crds/example_v1alpha1_nginx_cr.yaml
kubectl delete -f deploy/operator.yaml
kubectl delete -f deploy/role_binding.yaml
kubectl delete -f deploy/role.yaml
kubectl delete -f deploy/service_account.yaml
kubectl delete -f deploy/crds/example_v1alpha1_nginx_cr.yaml
```

Wait until all pods are deleted then delete th project.

```
kubectl get pods
oc delete project nginx-operator
```

That's it - your simple Helm operator demo complete!å

=======================================================================================
=======================================================================================
=======================================================================================
=======================================================================================
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
