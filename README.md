# Openshift Operator Demos

## Introduction
This is a discussion around Kubernetes/Openshift [Operators](https://coreos.com/blog/introducing-operator-framework) including instructions and a recording of how to create a demonstration of them.
We'll use Red Hat [Openshift](https://www.openshift.com/), the world's leading commercial Kubernetes distribution. As Openshift is a complete Kubernetes distribution, containing the Kube API and CLI, we'll use kubectl for most of our commands.

An [Operator](https://coreos.com/blog/introducing-operator-framework) is a piece of software that runs on Kubernetes that embeds operational knowledge around your containerised applications. An operator allows you to automate and re-use that knowledge to manage your application on an ongoing basis. Tasks typically undertaken by operators include:
- when and how to upgrade your application as new underlying components become available
- application and data backup management
- application patching 
- application scaling - horizontal and vertical
- configuration aspects - e.g. whether or use SSL, number of replicas etc.

An Operator is like an extension your engineering team that watches over the Kubernetes environment and uses its current state to make decisions in milliseconds.

The vision for Operators is to allow Independent Software Vendors (ISVs) to bundle operators with their software - in order to make them as maintainable, updatable and essentially as robust as possible, This vision is fast becoming a reality. 

Where operators will really become powerful is with the future release of the Operator Lifecycle Manager (OLM). Using the OLM, cluster administrators will be able to centrally manage and configure ISV provided operators - controlling everything an operator has been configured to do across the whole cluster and which namespaces get access.
Developers will then be able to provision or consume *operated services* that the administrator has made available.
See this [demo of OLM](https://www.youtube.com/watch?v=nGM2s4-Qr74).
Today's discussion focuses on the *building* of operators, as will be undertaken by ISVs. We'll document and demonstrate operator *usage* through the OLM in a future article.

Currently there are 3 Operator implementation options of varying levels of maturity and complexity. They are
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

# Helm based NGINX operator

This discussion and demo is based on an existing [Helm demo](https://github.com/operator-framework/operator-sdk/blob/master/doc/helm/user-guide.md).
As shown in the diagrams above, the Helm operator is the simplest of the 3 Operator types supported by Openshift. That said, it's a great way to run Helm charts in Openshift without Tiller, which as mentioned, requires root access and as such can be considered a security risk.   
The following shows the steps we will follow:
![](https://github.com/tnscorcoran/openshift-operators/blob/master/images/4-nginx-helm-operator-summary.png)

### Instructions

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

Build your local operator definition. Note the *api-version* and *kind* used to build the operator. This configures the operator to watch for and act on custom resources of this *api-version* and *kind*. 
 
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

> 2 things to note. In quay.io, under your new nginx-operator image:
	1) Under Tags, check the 'v0.0.1' checkbox
	2) Under Settings, make the repo public 


You will build your operator in Openshift using a Kubernetes Deployment object that references your newly pushed image in Quay. Modify your deployment object to reflect your image:

```
sed -i 's|REPLACE_IMAGE|quay.io/tnscorcoran/nginx-operator:v0.0.1|g' deploy/operator.yaml
```

As described in the Demo flow diagram above, we need to create a *Service Account* and a *Role* - meaning a non-human user and rights to perform actions, respectively. These 2 are joined and applied to our Operator in our *Role Binding*

> Note. This raw Kubernetes based example requires a small modification to run on Openshift. The Nginx application which the operator manages, requires root access. Openshift will not allow this by default - a prudent security feature of Openshift. As this is just a demo, we'll lift the restriction, running the command below: 'oc adm policy....'. In production, you should source images not requiring root access, like those in the Red Hat Container Catalog. 

Run the following commands to setup your objects and customer resource our operator will watch.



```
kubectl create -f deploy/service_account.yaml
oc adm policy add-scc-to-user anyuid -z default
kubectl create -f deploy/role.yaml
kubectl create -f deploy/role_binding.yaml
kubectl create -f deploy/operator.yaml
kubectl apply -f deploy/crds/example_v1alpha1_nginx_cr.yaml
```
Examine our Deployments, Pods and Service: 
```
kubectl get deployment
kubectl get pods
kubectl get service
```
- We can see our operator running as well as our Nginx application. Our Desired and Current replicas are in sync.
- 3 pods are running as expected
- Our Service is exposed on port 8080 as we configured in our Custom Resource instance that our operator is watching 

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
kubectl get pods
```

Wait until all pods are deleted then delete the project.

```
oc delete project nginx-operator
```

That's it - you've completed your simple Helm operator demo!


# Ansible based GOGS operator

Next we're going to discuss and run a more powerful operator - the Ansible operator. This is particularly useful for Ops folk who want to harness the power of Ansible in their operators. 
This discussion and demo is based on an existing [Ansible operator demo](http://www.opentlc.com/operators/04_01_Writing_an_Operator_Lab.html).

The following shows the steps we will follow:
![](https://github.com/tnscorcoran/openshift-operators/blob/master/images/6-gogs-ansible-operator-summary.png)

### Instructions

SSH into your RHEL box. If starting from scratch, set up Go Environment (version 1.11.2). As we're not, we don't need to do this.


Cleanup the work we did in the previous lab

```
cd $HOME
sudo rm -r $GOPATH/src/github.com/operator-framework
```


Clone and re-build the operator-sdk executable - as we use the v0.2.x branch for this demo

```
mkdir -p $GOPATH/src/github.com/operator-framework
cd $GOPATH/src/github.com/operator-framework
git clone https://github.com/operator-framework/operator-sdk
cd operator-sdk
git checkout v0.2.x
make dep
make install
```

There is a repository that someone else built that includes Ansible roles to set up a PostgreSQL database and a Gogs Server on top of OpenShift. The repository also contains an Ansible playbook that calls these roles to create a Gogs Server backed by a PostgreSQL database in a project on OpenShift. We'll now use the Ansible Operator to convert these roles into an Operator that can create Gogs installations by simple requesting a Custom Gogs Resource.
Clone and checkout desired branch:

```
cd $HOME
git clone https://github.com/redhat-gpte-devopsautomation/ansible-operator-roles
cd ansible-operator-roles
git checkout 0.2.0
cd $HOME
```

The Ansible Operator uses this playbook that sets up 2 tasks:
1) a GOGS Git server 
2) a Postgres database used by GOGS. 
Both tasks reference an Ansible role where the desired state and configuration of the underlying Pods is defined. 
Examine the playbook that calls the roles. This playbook will be periodically run to ensure actual state matches desired state:  

```
cat $HOME/ansible-operator-roles/playbooks/gogs.yaml

```

We can see the actual tasks Ansible executes by viewing the referenced tasks files: 

```
cat $HOME/ansible-operator-roles/roles/postgresql-ocp/tasks/main.yml
cat $HOME/ansible-operator-roles/roles/gogs-ocp/tasks/main.yml
```

As administrator, create your operator definition

```
oc login -u system:admin
operator-sdk new gogs-operator --api-version=gpte.opentlc.com/v1alpha1 --kind=Gogs --type=ansible --generate-playbook --skip-git-init
```


As we specified the type as *ansible*, the operator-sdk creates a skeleton playbook and a skeleton roles directory. But we already have a playbook and roles from the repository we cloned earlier.
Switch into the directory and replace the playbook and roles directory with the ones we downloaded.

```
cd $HOME/gogs-operator
rm -rf roles playbook.yaml
mkdir roles
cp -R $HOME/ansible-operator-roles/roles/postgresql-ocp ./roles
cp -R $HOME/ansible-operator-roles/roles/gogs-ocp ./roles
cp $HOME/ansible-operator-roles/playbooks/gogs.yaml ./playbook.yaml
```

Once we have our operator and its configuration in place via the playbooks above, the Ansible Operator's configuration is controlled by a simple file, *watches.yaml*. The link to the configuration we applied previously is through the *group*, *version* and *kind*
Take a look at this simple file: 

```
cat ./watches.yaml
```

Take a look at the Dockerfile used to build your image - it specifies the base Ansible Operator image and copies your Ansible files. 

```
cat build/Dockerfile
```

Let's change the base image to a more up to date one:

```
sed -i 's|quay.io/water-hole/ansible-operator|quay.io/wkulhanek/ansible-operator:v0.2.0|g' build/Dockerfile
```

If starting from scratch, Install Docker as described above (we don't need to).
Now it's time to build our own version of the Ansible Operator image and deploy to our own repo. (we use quay.io)
If needed, create a free account at https://quay.io. Mine is tnscorcoran. Login:

```
docker login -u tnscorcoran quay.io
```

Build and push to registry

```
cd $HOME/gogs-operator
operator-sdk build quay.io/tnscorcoran/gogs-operator:v0.0.1
docker push quay.io/tnscorcoran/gogs-operator:v0.0.1
```

Now we modify the operator deployment definition (deploy/operator.yaml) to use the image that we just pushed 

```
sed -i 's|REPLACE_IMAGE|quay.io/tnscorcoran/gogs-operator:v0.0.1|g' deploy/operator.yaml
cat deploy/operator.yaml
```

> Similar to above on the Helm demo, there are 2 things to note at this point. In quay.io, under your new gogs-operator image:
	1) Under Tags, check the 'v0.0.1' checkbox
	2) Under Settings, make the repo public 

Now it's almost time to deploy the Gogs Operator. First examine the Custom Resource Definition that got created by the Operator SDK. Instances of these objects, once created, will be managed by the operator we just created.

```
cd $HOME/gogs-operator
cat deploy/crds/gpte_v1alpha1_gogs_crd.yaml
```

As a non-privileged, developer user and create a project 

```
oc login -u andrew
oc new-project gogs-operator
```

Now, as an administrator, create the Custom Resource Definition (or blueprint for the objects operator will act on)

```
oc login -u system:admin
oc create -f deploy/crds/gpte_v1alpha1_gogs_crd.yaml
```

Now we create a *Cluster Role* detailing the actions this operator can execute (there is also a namespace scoped equivalent *Role*)
Create this Cluster Role:

```
echo '---
apiVersion: authorization.openshift.io/v1
kind: ClusterRole
metadata:
  labels:
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
  name: gogs-admin-rules
rules:
- apiGroups:
  - gpte.opentlc.com
  resources:
  - gogs
  verbs:
  - create
  - update
  - delete
  - get
  - list
  - watch
  - patch' | oc create -f -
```

We also need a non-human Openshift user to execute the operator tasks

```
oc create -f ./deploy/service_account.yaml
```
Replace the existing Role with one having the correct permissions

```
sudo rm $HOME/gogs-operator/deploy/role.yaml

echo 'apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gogs-operator
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - services
  - endpoints
  - persistentvolumeclaims
  - configmaps
  - secrets
  - routes
  verbs:
  - create
  - update
  - delete
  - get
  - list
  - watch
  - patch
- apiGroups:
  - gpte.opentlc.com
  resources:
  - gogs
  verbs:
  - create
  - update
  - delete
  - get
  - list
  - watch
  - patch' > $HOME/gogs-operator/deploy/role.yaml
  
oc create -f $HOME/gogs-operator/deploy/role.yaml
```

The Role Binding connects this Role to the Operator's Service account:

```
oc create -f $HOME/gogs-operator/deploy/role_binding.yaml
```

Now create the Operator and wait for it to come up 

```
oc create -f ./deploy/operator.yaml
watch oc get pod
```

Use the Operator to create a Gogs Server and its dependant Postgres DB and watch them come up:

```
echo "apiVersion: gpte.opentlc.com/v1alpha1
kind: Gogs
metadata:
  name: gogs-server
spec:
  postgresqlVolumeSize: 4Gi
  gogsVolumeSize: 4Gi
  gogsSsl: True" > $HOME/gogs-operator/gogs-server.yaml

oc create -f $HOME/gogs-operator/gogs-server.yaml

watch oc get pod
```

See your GOGS custom resource and describe the Gogs server

```
oc get gogs
oc describe gogs gogs-server
```

View your Route

```
oc get route
```

Notice if you paste in the the route hostname without the protocol, you'll be redirected to the https route - as we specified gogsSsl as true in our Custom Resource


Now deploy a second gogs server - with this Custom Resource instance - configured with *gogsSsl:false*. Wait for it to come up: 

```
echo "apiVersion: gpte.opentlc.com/v1alpha1
kind: Gogs
metadata:
  name: another-gogs
spec:
  postgresqlVolumeSize: 4Gi
  gogsVolumeSize: 4Gi
  gogsSsl: False" > $HOME/gogs-operator/gogs-server-2.yaml

oc create -f $HOME/gogs-operator/gogs-server-2.yaml
watch oc get pod
```


When it's ready, use the route as previously

```
oc get route
```

It's http based, so it doesn't redirect as we specified *gogsSsl: False*

Now it's almost time to clean up the Gogs Servers. One final thing - notice our Postgresql DB Pod definition has an *ownerReferences* section specifying *gogs-server* when we view its yaml:  

```
oc get pod postgresql-gogs-gogs-server -o yaml
```

By contrast, the top level *gogs-server* - does not have an *ownerReferences* section:  

```
oc get gogs gogs-server -o yaml
```

This *parent referencing* allows us to delete the entire dependency tree just by deleting the top level entity - a useful automation feature.
Let's delete our 2 gogs servers and watch their dependant Postgres DBs being deleted:

```
oc delete gogs gogs-server
oc delete gogs another-gogs
oc get pods -w
```

Now complete our cleanup and we're done.

```
oc delete deployment gogs-operator
oc delete rolebinding gogs-operator
oc delete role gogs-operator
oc delete sa gogs-operator
```

Wait until everything is gone

```
oc get pods
```

Now delete the project and cluster scoped objects

```
oc delete project gogs-operator
oc delete clusterrole gogs-admin-rules
oc delete crd gogs.gpte.opentlc.com
```

That's it! Thanks for allowing me to demo you 2 Kubernetes and Openshift Operators - the Helm Operator and the Ansible Operator. A video of this demonstration is available at https://youtu.be/kkJWHqiuxlQ

