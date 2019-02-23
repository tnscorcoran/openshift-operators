# Openshift Operators

TODO - 
	With Ansible Operator - Draw a diagram showing the elements - and what each element determines
	(equivalent of the following etc-go-operator)
	
	->		ROLE						
	->		ROLE-BINDING		
	->		CREATE-ROLE.sh (creates ROLE and ROLE-BINDING, uses example/rbac yaml templates in the case of etcd go 									operator)	
	->		CRD	(determines plurals etc
				-  with Ansible Operator - how does it pull in what the plurals etc are
					- the plurals etc that show up when you run: 
					oc describe crd etcdclusters.etcd.database.coreos.com 
				)	
	->		INSTANCE-OF-TYPE: CRD (determines replicas and version)		
	-> 		OPERATOR ( 	- of type: Deployment 
						watches for CRD instance creation and makes current state 
						as is in INSTANCE-OF-TYPE: CRD
						 - what is it this Deployment type that causes it to watch for CRD instance creation?	
						 	)
						 	
========================================================================================================

Ansible Operator Notes
======================

Make a reference to the fact this is based on [1] 

[1 http://www.opentlc.com/operators/04_01_Writing_an_Operator_Lab.html]

=========

FROM: first sentence of: http://www.opentlc.com/operators/04_01_Writing_an_Operator_Lab.html
ADD: These to intro
	- "The Operator then watches for custom resource objects to be created - and when it sees a custom resource 	
	being created it creates the application based on the parameters in the custom resource object."
	- "Red Hat has provided a Software Development Kit that significantly simplifies the creation of Operators. 
	The SDK handles all the core logic while the developer only needs to fill in the actual business logic."
	- "In this lab you are using the Ansible Operator SDK to create an Operator using two Ansible Roles and a 
	Playbook that sets up a PostgreSQL Database and a Gogs Server that uses that database"

=========

"There is a repository that someone else built that includes Ansible roles to set up a PostgreSQL database and a Gogs Server on top of OpenShift. The repository also contains an Ansible playbook that calls these roles to create a Gogs Server backed by a PostgreSQL database in a project on OpenShift.
You will now use the Ansible Operator to convert these roles into an Operator that can create Gogs installations by simple requesting a Custom Gogs Resource."

=========

- take my notes from DAY_3_OPERATORS_2_WRITE_GOGSOPERATOR.txt - with # and use them in the repo 

=========

https://www.lucidchart.com
	tcorcora@redhat.com
	
	https://www.lucidchart.com/documents/edit/6851404a-f3e7-420d-a723-7f2f773801a5/0

=========
						 	
						 	