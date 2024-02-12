---
title: "Simple Tensorflow Serving Operator"
date: 2019-11-02
draft: false
categories: ["Kubernetes"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

Kubernetes is a powerful and highly extensible system for managing containerized workloads and services. It has two main components, the Master Components and the Node Componenets, and extensions.  

### The Master Components:

* **Kube-apiserver**, the API server is the front end for the Kubernetes control plane.
* **etcd**, is a highly-available key value store used as Kubernetes’ backing store for all cluster data.
* **kube-scheduler**, is the component of the master that watches newly created pods that have no node assigned, and selects a node for them to run on.
* **kube-control-manager**, is single binary which holds all the built in controlers like Node Controller, Replication Controller, Endpoints Controller, Service Account & Token Controllers.

### The Node Components:

* **kubelet**, an agent that runs on each node in the cluster, which makes sure that containers are running in a pod based on the Specs.
* **kube-proxy**, is a network proxy that runs on each node, which maintains network rules.
* **Container Runtime**, is the software that is responsible for running containers, like docker.  

### CRDs, Controllers and Operators

With Kubernetes, it is relatively easy to manage and scale web apps, mobile backends, and API services right out of the box, because these applications are generally stateless, so the basic Kubernetes APIs, like Deployments, can scale and recover from failures without additional knowledge.

In a Core Os [Blog Post](https://coreos.com/blog/introducing-operators.html) from 2016 the Operator Concept was introduced. An Operator is an application-specific controller that extends the Kubernetes API to create, configure, and manage instances of complex stateful applications on behalf of a Kubernetes user. It builds upon the basic Kubernetes resource and controller concepts but includes domain or application-specific knowledge to automate common tasks.
Operators are clients of the Kubernetes API that act as controllers for a Custom Resource.

But first let's look at what Controllers does. Based on Kubernetes Glossary reference, controllers are control loops that watch the state of your cluster, then make or request changes where needed. Each controller tries to move the current cluster state closer to the desired state. So basically, a controller tracks at least one Kubernetes resource type. These objects have a spec field that represents the desired state. The controller for that resource are responsible for making the current state come closer to that desired state.

The Custom Resource is an endpoint in the Kubernetes API that stores a collection of API objects of a certain kind. For example, the built-in pods resource contains a collection of Pod objects. A custom resource is an extension of the Kubernetes API that is not necessarily available in a default Kubernetes installation. Many core Kubernetes functions are now built using custom resources, making Kubernetes more modular.

A Custom Resource Definition (CRD) file defines your own object kinds and lets the API Server handle the entire lifecycle. Deploying a CRD into the cluster causes the Kubernetes API server to begin serving the specified custom resource. When you create a new custom resource definition (CRD), the Kubernetes API Server reacts by creating a new RESTful resource path, that can be accessed by an entire cluster or a single project (namespace).

* most of the above definitions are from https://kubernetes.io/docs/concepts/

### Extending Kubernetes

Now that we covred the basic building blocks for extending Kubernetes, there are a number of options to put this in practice. Several tools were built to facilitate the creation of custom controllers.

* [**client-go**](https://github.com/kubernetes/client-go), is the offical API client library, providing access to Kubernetes restful API interface served by the Kubernetes API server. The library contains several important packages and utilities which can be used for accessing the API resources or facilitate a custom controller. 

* [**Sample Controller**](https://github.com/kubernetes/sample-controller), uses the client-go library directly and the [code-generator](https://github.com/kubernetes/code-generator) to generate a typed client, informers, listers, and deep-copy functions. Whenever the API types change in your custom controller—for example, adding a new field in the custom resource — you have to use the update-codegen.sh script to regenerate the aforementioned source files. Check out [this Link](https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md) if you are interested to find how it works under the hood.

* [**Kubebulder**](https://github.com/kubernetes-sigs/kubebuilder) owned and maintained by the Kubernetes Special Interest Group (SIG) API Machinery, is a tool and set of libraries enabling you to build operators in an easy and efficient manner. This is what I'm going to use to build the Simple Tensorflow Serving Operator.

* [**Operator Framework**](https://github.com/operator-framework) is an open source project that provides developer and runtime Kubernetes tools, enabling you to accelerate the development of an Operator. 
It is somehow similar with Kubebuilder, but if you are interested what provide one vs the other, [here](https://github.com/operator-framework/operator-sdk/issues/1758) you can find some hints.
The Operator Framework includes:
  * Operator SDK which enables developers to build Operators based on their expertise without requiring knowledge of Kubernetes API complexities.
  * Operator Lifecycle Management: Oversees installation, updates, and management of the lifecycle of all of the Operators (and their associated services) running across a Kubernetes cluster.
  * Operator Metering (joining in the coming months): Enables usage reporting for Operators that provide specialized services.  

### Kubebuilder

Building Kubernetes tools and APIs involves making a lot of decisions and writing a lot of boilerplate. Kubebuilder attempts to facilitate the following developer workflow for building APIs

* Create a new project directory
* Create one or more resource APIs as CRDs and then add fields to the resources
* Implement reconcile loops in controllers and watch additional resources
* Test by running against a cluster (self-installs CRDs and starts controllers automatically)
* Update bootstrapped integration tests to test new fields and business logic
* Build and publish a container from the provided Dockerfile

Install Kubebuilder:

{{< code language="bash" isCollapsed="false" >}}
os=$(go env GOOS)
arch=$(go env GOARCH)

# download kubebuilder and extract it to tmp
curl -sL https://go.kubebuilder.io/dl/2.0.1/${os}/${arch} | tar -xz -C /tmp/

# move to a long-term location and put it on your path
sudo mv /tmp/kubebuilder_2.0.1_${os}_${arch} /usr/local/kubebuilder
export PATH=$PATH:/usr/local/kubebuilder/bin
{{< /code >}}

## A simple Tensorflow Serving Operator

what is [**Tensorflow Serving**](https://www.tensorflow.org/tfx/guide/serving) ? Based on the official definition, is a flexible, high-performance serving system for machine learning models, designed for production environments.  It deals with the inference aspect of machine learning, taking models after training and managing their lifetimes.

Before diving into the nitty-gritty of building the operator, let's talk a little bit of what I am trying to achieve. My Operator role would be to deploy and serve a Tensorflow model over Kubernetes and automate a number of steps like: creating the deployment and the service yaml, pull the model from block storage and serve at scale. 

You can find a number of articles on the internet on how to run **Tensorflow Serving** within a docker container or even how to deploy a serving cluster with Kubernetes. That's cool, but the expectation is that you are going to manage all the steps mentioned above. If you are a machine learning expert, working with Tensorflow, you would like to have these steps automated for you so you can focus on doing experiments and to place your models into production as fast as possible.

There are a number of tools that facilitate these things, maybe most known and used framework over Kubernetes is [**Kubeflow**](https://www.kubeflow.org/#overview), which I would highly recommend to everyone who is searching for a production ready Machine Learning workflow on Kubernetes.  
My intention is not to compete with these tools, as this is just a toy project but to showcase how to automate some of these steps using a custom made Kubernetes Operator. 

### Scaffolding Out the Project

In this first step, we are doing any codding, but you'll see that at the end of the step we'll have a functional Operator with the custom CRDs installed. This is the value proposition that Kubebuilder offers to us.

To initialize a new project, run following command:

{{< code language="bash" isCollapsed="false" >}}
$ kubebuilder init --domain dev-state.com --license apache2 --owner "Your Name"
{{< /code >}}

It generates a Kubebuilder project template with a Makefile, a Dockerfile a basic manager and some default yaml files.  
Now we can create an API, but before that it is important to get a bit of understanding of the components of the API. We will define **groups**, **versions** and **kind**.  
An **API Group** in Kubernetes is simply a collection of related functionality. Each group has **one or more versions**, which allow us to change how an API works over time. Each API group-version contains one or more API types, called **Kinds**. A **resource** is simply a use of a Kind in the API. Often, there’s a one-to-one mapping between Kinds and resources. For instance, the **pods** resource corresponds to the **Pod Kind**.
Resources and Kinds are always part of an API group and a version, collectively referred to as GroupVersionResource (GVR) or GroupVersionKind(GVK).  

{{< code language="bash" isCollapsed="false" >}}
kubebuilder create api --group servapi --version v1alpha1 --kind Tfserv
Create Resource [y/n]
y
Create Controller [y/n]
y
Writing scaffold for you to edit...
api/v1alpha1/tfserv_types.go
controllers/tfserv_controller.go
Running make...
{{< /code >}}

On completion of this command, Kubebuilder has scaffolded the operator,generating a bunch of files, from the custom controller to a sample CRD. It creates an api/v1alpha1 directory, corresponding to servapi.dev-state.com/v1alpha1 group-version. has also added a file **tfserv_types.go** which contain the define kind **Tfserv**.  
The directory structure should look something similar to this:

{{< code language="bash" isCollapsed="false" >}}
.
├── api
│   └── v1alpha1
│       ├── groupversion_info.go
│       ├── tfserv_types.go
│       └── zz_generated.deepcopy.go
├── bin
│   └── manager
├── config
│   ├── certmanager
│   │   ├── certificate.yaml
│   │   ├── kustomization.yaml
│   │   └── kustomizeconfig.yaml
│   ├── crd
│   │   ├── kustomization.yaml
│   │   ├── kustomizeconfig.yaml
│   │   └── patches
│   │       ├── cainjection_in_tfservs.yaml
│   │       └── webhook_in_tfservs.yaml
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   ├── manager_prometheus_metrics_patch.yaml
│   │   ├── manager_webhook_patch.yaml
│   │   └── webhookcainjection_patch.yaml
│   ├── manager
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── rbac
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   ├── leader_election_role.yaml
│   │   └── role_binding.yaml
│   ├── samples
│   │   └── servapi_v1alpha1_tfserv.yaml
│   └── webhook
│       ├── kustomization.yaml
│       ├── kustomizeconfig.yaml
│       └── service.yaml
├── controllers
│   ├── suite_test.go
│   └── tfserv_controller.go
├── Dockerfile
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── main.go
├── Makefile
├── PROJECT
└── README.md

14 directories, 39 files
{{< /code >}}

Most of the business logic will go within these files api/v1alpha1/tfserv_types.go and controllers/tfserv_controller.go. The rest of files help us to install the CRDs and build and run the controller in the Kubernetes cluster.  
We want to see what it offers out-of-the-box , so we are not doing yet any changes, we use the defaults. Install the CRDS into the cluster:

{{< code language="bash" isCollapsed="false" >}}
$ make install
controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases

kustomize build config/crd | kubectl apply -f -
customresourcedefinition.apiextensions.k8s.io/tfservs.servapi.dev-state.com created

$ kubectl get crds
NAME                            CREATED AT
tfservs.servapi.dev-state.com   2019-10-21T14:02:57Z
{{< /code >}}

Run your controller locally (this will run in the foreground, so switch to a new terminal if you want to leave it running):

{{< code language="bash" isCollapsed="false" >}}
$ make run
controller-gen object:headerFile=./hack/boilerplate.go.txt paths="./..."
go fmt ./...
go vet ./...

controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases

go run ./main.go

2019-10-21T17:08:34.537+0300	INFO	controller-runtime.metrics	metrics server is starting to listen	{"addr": ":8080"}
2019-10-21T17:08:34.537+0300	INFO	controller-runtime.controller	Starting EventSource	{"controller": "tfserv", "source": "kind source: /, Kind="}
2019-10-21T17:08:34.538+0300	INFO	setup	starting manager
2019-10-21T17:08:34.538+0300	INFO	controller-runtime.manager	starting metrics server	{"path": "/metrics"}
2019-10-21T17:08:34.638+0300	INFO	controller-runtime.controller	Starting Controller	{"controller": "tfserv"}
2019-10-21T17:08:34.739+0300	INFO	controller-runtime.controller	Starting workers	{"controller": "tfserv", "worker count": 1}
{{< /code >}}

In a new terminal session create the sample custom resource like this:

{{< code language="bash" isCollapsed="false" >}}
$ kubectl apply -f config/samples/
tfserv.servapi.dev-state.com/tfserv-sample created

kubectl get tfserv
NAME            AGE
tfserv-sample   2m12s
{{< /code >}}

If you are looking to the output of the session where controller is running you should see something similar with this:

{{< code language="bash" isCollapsed="false" >}}
2019-10-21T17:13:36.735+0300	DEBUG	controller-runtime.controller	Successfully Reconciled	{"controller": "tfserv", "request": "default/tfserv-sample"}
{{< /code >}}

This tells us that the overall setup was successful so we can start to implement the business logic.

### Designing the API

In terms of business logic there are two parts to implement in the operator. We have to modify the the Spec and Status of the defined Kind. Kubernetes functions by reconciling desired state **Spec** with actual cluster state and then recording what it observed in **Status**.

Tfserv is our root type, and describes the Tfserv kind. Like all Kubernetes objects, it contains TypeMeta (which describes API version and Kind), and also contains ObjectMeta, which holds things like name, namespace,and labels.

{{< code language="go" isCollapsed="false" >}}
// Tfserv is the Schema for the tfservs API
type Tfserv struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   TfservSpec   `json:"spec,omitempty"`
	Status TfservStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true
{{< /code >}}

**+kubebuilder:object:root** comment is called a marker, tells the object generator that this type represents a Kind, so it can generate an implementation of the **runtime.Object** interface for us, which is the standard interface that all types representing Kinds must implement. 

{{< code language="go" isCollapsed="false" >}}
type Object interface {
    GetObjectKind() schema.ObjectKind
    DeepCopyObject() Object
}
{{< /code >}}

A Kubernetes object in Go is a data structure that can return and set the GroupVersionKind and make a deep copy, which is a clone of the data structure,so it does not share any memory with the original object.

Now we get into the specific of our implementation. What is required for user to provide in order to create the Tensorflow Serving Infrastructure?

Tensorflow Serving support GRPC APIS but also RESTful APIs, so you can specify the port number you would like to listen on.
* GrpcPort -- GRPC port number
* RestPort -- REST port number
* Replicas -- number of Tensorflow serving instances
* ConfigMap -- it holds the configuration of the Tensorflow serving service
* ConfigFileName -- is the name of the config file
* ConfigFileLocation -- is the path to config file
* SecretFileName -- is the name of the secret file, which is required if the model is hosted on Google Storage or AWS S3
* SecretFileLocation -- is the path to the Secret file

{{< code language="go" isCollapsed="false" >}}
// TfservSpec defines the desired state of Tfserv
type TfservSpec struct {
    GrpcPort int32 `json:"grpcPort,omitempty"`
	RestPort int32 `json:"restPort,omitempty"`
	Replicas int32 `json:"replicas,omitempty"`
	ConfigMap string `json:"configMap,omitempty"`
	ConfigFileName string `json:"configFileName,omitempty"`
	ConfigFileLocation string `json:"configFileLocation,omitempty"`
	SecretFileName string `json:"secretFileName,omitempty"`
	SecretFileLocation string `json:"secretFileLocation,omitempty"`
}
{{< /code >}}

The status is what is recorded by the Controller in the reconciling process.

* Active -- record a list of pointers to currently running jobs.

{{< code language="go" isCollapsed="false" >}}
// TfservStatus defines the observed state of Tfserv
type TfservStatus struct {
	Active []corev1.ObjectReference `json:"active,omitempty"`
}
{{< /code >}}

In the end, we add the Go types to the API group. This allows us to add the types in this API group to any Scheme.

{{< code language="go" isCollapsed="false" >}}
func init() {
	SchemeBuilder.Register(&Tfserv{}, &TfservList{})
}
{{< /code >}}

Scheme defines methods for serializing and deserializing API objects, a type registry for converting group, version, and kind information to and from Go schemas, and mappings between Go schemas of different versions. The main feature of a scheme is the mapping of Golang types to possible GVKs.

### Implementing the Controller

Controllers are the core component of Kubernetes, their role is to ensure that the actual state of the cluster matches the desired state. Reconciler implements a Kubernetes API for a specific Resource by Creating, Updating or Deleting Kubernetes objects, or by making changes to systems external to the cluster (e.g. cloudproviders, github, etc). Reconcile implementations compare the state specified in an object by a user against the actual cluster state, and then perform operations to make the actual cluster state reflect the state specified by the user.  Another role of the Reconcile function is to update the Status part of the custom resource.

There is a Reconcilier struct, that define the dependencies of the reconcile method. 

{{< code language="go" isCollapsed="false" >}}
type TfservReconciler struct {
	client.Client
	Log    logr.Logger
	Scheme *runtime.Scheme
}
{{< /code >}}

It is instantiated in the main package, but I'll get there soon. What it important to know is that **client.Client** contains functionality for interacting with Kubernetes API servers and it knows how to perform CRUD operations on Kubernetes objects, like Get, Create, Delete, List, Update which are used heavily within reconciler method. A logger and the Scheme should include the schemes of all objects the controller will work with.

In the reconcile method, we'll fetch the Tfserv custom resource by calling Get method, which takes in a ctx, the Namespace and the object requested.
If the resource is not present, ignore it, if there is an error while retrieving the resource then requeue it.

{{< code language="go" isCollapsed="false" >}}
var tfs servapiv1alpha1.Tfserv
if err := r.Get(ctx, req.NamespacedName, &tfs); err != nil {
	log.Error(err, "unable to fetch tfs")
	// we'll ignore not-found errors, since they can't be fixed by an immediate
	// requeue (we'll need to wait for a new notification), and we can get them
	// on deleted requests.
	return ctrl.Result{}, ignoreNotFound(err)
}

func ignoreNotFound(err error) error {
    if apierrs.IsNotFound(err) {
        return nil
    }
    return err
}
{{< /code >}}

As the tensorflow serving requires a model-config-file to be defined. We are injecting configuration data into Pods using ConfiMaps. The data stored in a ConfigMap object can be referenced in a volume of type configMap and then consumed by containerized applications running in a Pod.  
As this is critical for the application we can't move forward untill this condition is satisfied and we are able to find the ConfigMap.

{{< code language="go" isCollapsed="false" >}}
	var configmap *corev1.ConfigMap
	err := r.Get(ctx, types.NamespacedName{Name: tfs.Spec.ConfigMap, Namespace: tfs.Namespace}, configmap)
	if err != nil && errors.IsNotFound(err) {
		log.V(1).Info("ConfigMap not found, wont continue untill config map is found", "tfs.Spec.ConfigMap", tfs.Spec.ConfigMap)
		return ctrl.Result{}, err
	}
{{< /code >}}

We need a deployment and a service infront of the cluster to serve the tensorflow serving containers.  We mandate the controller to create the resources and to compare those with the ones that are already running in the cluster.  
The desired deployment is created in the memmory. Then, it tries to find the deployment in the Kubernetes cluster, by namespace and name. If it could not be found then it creates one, else it compare the Specs of the found deployment with the desired deployment. If there are not the same then it updates the found deployment. If errors occurs while creating or updating the deployment,it is requed to try again later.

{{< code language="go" isCollapsed="false" >}}
labels := map[string]string{
		"tfsName": tfs.Name,
	}

	// Define the desired Deployment object
	var deployment *appsv1.Deployment
	deployment, err = r.createDeployment(&tfs, labels)
	if err != nil {
		return ctrl.Result{}, err
	}

	// Check if the Deployment already exists
	foundDeployment := &appsv1.Deployment{}
	err = r.Get(ctx, types.NamespacedName{Name: deployment.Name, Namespace: deployment.Namespace}, foundDeployment)
	//If the deployment does not exist, create it
	if err != nil && errors.IsNotFound(err) {
		log.V(1).Info("Creating Deployment", "Deployment.Namespace", deployment.Namespace, "Deployment.Name", deployment.Name)
		err = r.Create(ctx, deployment)
		if err != nil {
			return ctrl.Result{}, err
		}
		//	r.recorder.Eventf(tfs, "Normal", "DeploymentCreated", "The Deployment %s has been created", deployment.Name)
		log.V(1).Info("The Deployment has been created", "Deployment.Name", deployment.Name)
		return ctrl.Result{}, nil
	} else if err != nil {
		return ctrl.Result{}, err
	}

	// Update the found deployment and write the result back if there are any changes
	if !reflect.DeepEqual(deployment.Spec, foundDeployment.Spec) {
		foundDeployment.Spec = deployment.Spec
		log.V(1).Info("Updating Deployment", "Deployment.Namespace", deployment.Namespace, "Deployment.Name", deployment.Name)
		err = r.Update(ctx, foundDeployment)
		if err != nil {
			return reconcile.Result{}, err
		}
		//	return reconcile.Result{}, nil
	}
{{< /code >}}

The function to create the deployment, it looks like below. It's worthwhile to notice that the **user defined Specs** are used all over the place and two volume are mounted. One is for **Secret key** as the model is hosted in the cloud and in order to access cloud resources a service account key is required. The othe one is the ConfigMap volume type, which is used to access the configuration file from the ConfigMap.  
**SetControllerReference** sets owner as a Controller OwnerReference on owned. This is used for garbage collection of the owned object and for
reconciling the owner object on changes to owned (with a Watch + EnqueueRequestForOwner).

{{< code language="go" isCollapsed="false" >}}
func (r *TfservReconciler) createDeployment(tfs *servapiv1alpha1.Tfserv, labels map[string]string) (*appsv1.Deployment, error) {

	deployment := &appsv1.Deployment{
		ObjectMeta: metav1.ObjectMeta{
			Name:      tfs.Name,
			Namespace: tfs.Namespace,
			Labels:    labels,
		},
		Spec: appsv1.DeploymentSpec{
			Replicas: &tfs.Spec.Replicas,
			Selector: &metav1.LabelSelector{
				MatchLabels: labels,
			},
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Name:   tfs.Name,
					Labels: labels,
				},
				Spec: corev1.PodSpec{
					Containers: []corev1.Container{
						{
							Name:    "tfs-main",
							Image:   "tensorflow/serving:latest",
							Command: []string{"/usr/bin/tensorflow_model_server"},
							Args: []string{
								fmt.Sprintf("--port=%d", tfs.Spec.GrpcPort),
								fmt.Sprintf("--rest_api_port=%d", tfs.Spec.RestPort),
								fmt.Sprintf("--model_config_file=%s%s", tfs.Spec.ConfigFileLocation, tfs.Spec.ConfigFileName),
							},
							Env: []corev1.EnvVar{
								{
									Name:  "GOOGLE_APPLICATION_CREDENTIALS",
									Value: tfs.Spec.SecretFileLocation + tfs.Spec.SecretFileName,
								},
							},
							Ports: []corev1.ContainerPort{
								{
									ContainerPort: tfs.Spec.GrpcPort,
									Protocol:      corev1.ProtocolTCP,
								},
								{
									ContainerPort: tfs.Spec.RestPort,
									Protocol:      corev1.ProtocolTCP,
								},
							},
							VolumeMounts: []corev1.VolumeMount{
								{
									Name:      "tfs-config-volume",
									MountPath: tfs.Spec.ConfigFileLocation,
								},
								{
									Name:      "tfs-secret-volume",
									MountPath: tfs.Spec.SecretFileLocation,
								},
							},
						},
					},
					Volumes: []corev1.Volume{
						{
							Name: "tfs-config-volume",
							VolumeSource: corev1.VolumeSource{
								ConfigMap: &corev1.ConfigMapVolumeSource{
									LocalObjectReference: corev1.LocalObjectReference{
										Name: "tf-serving-models-config",
									},
								},
							},
						},
						{
							Name: "tfs-secret-volume",
							VolumeSource: corev1.VolumeSource{
								Secret: &corev1.SecretVolumeSource{
									SecretName: "tfs-secret",
								},
							},
						},
					},
				},
			},
		},
	}


	if err := controllerutil.SetControllerReference(tfs, deployment, r.Scheme); err != nil {
		return nil, err
	}

	return deployment, nil
}
{{< /code >}}

Lastly we need a Service, which is an abstraction that defines a logical set of Pods and a policy by which to access them. The set of Pods targeted by a Service is usually determined by a selector. The same mechanism, as we saw before for deployment applys here as well.

{{< code language="go" isCollapsed="false" >}}
//Define the desired Service object
	var service *corev1.Service
	service, err = r.createService(&tfs, labels)
	if err != nil {
		return ctrl.Result{}, err
	}

	//check if the service already exists
	foundService := &corev1.Service{}
	err = r.Get(ctx, types.NamespacedName{Name: service.Name, Namespace: service.Namespace}, foundService)
	//If the service does not exist, create it
	if err != nil && errors.IsNotFound(err) {
		log.V(1).Info("Creating a new Service", "Service.Namespace", service.Name, "Service.Name", service.Name)
		err = r.Create(ctx, service)
		if err != nil {
			return reconcile.Result{}, err
		}
		log.V(1).Info("The Service has been created", "Service.Name", service.Name)
		return ctrl.Result{}, nil
	} else if err != nil {
		return reconcile.Result{}, err
	}

{{< /code >}}

And the create service function:

{{< code language="go" isCollapsed="false" >}}
func (r *TfservReconciler) createService(tfs *servapiv1alpha1.Tfserv, labels map[string]string) (*corev1.Service, error) {

	service := &corev1.Service{
		ObjectMeta: metav1.ObjectMeta{
			Name:      tfs.Name + "-service",
			Namespace: tfs.Namespace,
			Labels:    labels,
		},
		Spec: corev1.ServiceSpec{
			Ports: []corev1.ServicePort{
				{
					Name:     "rest",
					Port:     tfs.Spec.RestPort,
					Protocol: corev1.ProtocolTCP,
				},
				{
					Name:     "grpc",
					Port:     tfs.Spec.GrpcPort,
					Protocol: corev1.ProtocolTCP,
				},
			},
			//Selector: map[string]string{"app": tfs.Name},
			Selector: labels,
			Type:     corev1.ServiceTypeNodePort,
		},
	}

	if err := controllerutil.SetControllerReference(tfs, service, r.Scheme); err != nil {
		return nil, err
	}

	return service, nil
}
{{< /code >}}

### Putting all together in the main package

In order to allow our reconciler to quickly look up Deployments and Services by their owner, we’ll need an index. We declare an index key that we can later use with the client as a pseudo-field name, and then describe how to extract the indexed value from the objects. This inform the manager that this controller owns some resources, so that it will automatically call Reconcile on the underlying resources changes.

{{< code language="go" isCollapsed="false" >}}
//SetupWithManager setup the controler with manager
func (r *TfservReconciler) SetupWithManager(mgr ctrl.Manager) error {
	if err := mgr.GetFieldIndexer().IndexField(&appsv1.Deployment{}, ownerKey, func(rawObj runtime.Object) []string {
		// grab the job object, extract the owner...
		deployment := rawObj.(*appsv1.Deployment)
		owner := metav1.GetControllerOf(deployment)
		if owner == nil {
			return nil
		}
		// ...make sure it's a Deployment...
		if owner.APIVersion != apiGVStr || owner.Kind != "Tfserv" {
			return nil
		}

		// ...and if so, return it
		return []string{owner.Name}
	}); err != nil {
		return err
	}

	if err := mgr.GetFieldIndexer().IndexField(&corev1.Service{}, ownerKey, func(rawObj runtime.Object) []string {
		// grab the job object, extract the owner...
		service := rawObj.(*corev1.Service)
		owner := metav1.GetControllerOf(service)
		if owner == nil {
			return nil
		}
		// ...make sure it's a Service...
		if owner.APIVersion != apiGVStr || owner.Kind != "Tfserv" {
			return nil
		}

		// ...and if so, return it
		return []string{owner.Name}
	}); err != nil {
		return err
	}

	return ctrl.NewControllerManagedBy(mgr).
		For(&servapiv1alpha1.Tfserv{}).
		Owns(&appsv1.Deployment{}).
		Owns(&corev1.Service{}).
		Complete(r)
}
{{< /code >}}

In the main.go file, we have to ensure that we reference and add 
all the resources schemes.

{{< code language="go" isCollapsed="false" >}}
func init() {
	_ = clientgoscheme.AddToScheme(scheme)

	_ = servapiv1alpha1.AddToScheme(scheme)
	_ = corev1.AddToScheme(scheme)
	_ = appsv1.AddToScheme(scheme)
	// +kubebuilder:scaffold:scheme
}
{{< /code >}}

Then we use **ctrl.NewManager** to create new manager, which is used for creating Controllers. **SetupWithManager** registers this reconciler with the controller manager and starts watching Tfserv, Deployment and Service resources.

{{< code language="go" isCollapsed="false" >}}
if err = (&controllers.TfservReconciler{
		Client: mgr.GetClient(),
		Log:    ctrl.Log.WithName("controllers").WithName("Tfserv"),
		Scheme: mgr.GetScheme(),
	}).SetupWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to create controller", "controller", "Tfserv")
		os.Exit(1)
	}
{{< /code >}}

The manager is started with **mgr.Start** command.

### Running the Operator

You need a **service account key** to access the model. You can find some details [here](https://cloud.google.com/iam/docs/service-accounts), on what is a Service Account and how to create keys to autheticate applications.  

{{< code language="bash" isCollapsed="false" >}}
$ kubectl create secret generic tfs-secret --from-file=./test-apps-model-viewer.json
secret/tfs-secret created
{{< /code >}}

Install rbac and CRDs:

{{< code language="bash" isCollapsed="false" >}}
$ make install
controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
kustomize build config/crd | kubectl apply -f -
customresourcedefinition.apiextensions.k8s.io/tfservs.servapi.dev-state.com created
{{< /code >}}

Verify if the CRDs were installed:

{{< code language="bash" isCollapsed="false" >}}
$ kubectl get crds
NAME                            CREATED AT
tfservs.servapi.dev-state.com   2019-10-28T16:20:04Z
{{< /code >}}

Create ConfigMap where we define the configuration of the tensorflow service. 

{{< code language="bash" isCollapsed="false" >}}
$ kubectl apply -f servapi_configmap_resnet.yaml 
configmap/tf-serving-models-config created
{{< /code >}}

Now apply the configuration of the resource for the Tfserv CRD.

{{< code language="bash" isCollapsed="false" >}}
$ kubectl apply -f servapi_v1alpha1_tfserv.yaml 
tfserv.servapi.dev-state.com/tfserv-sample created
{{< /code >}}

There should be no pods either Services available, as the controller is not running yet.

{{< code language="bash" isCollapsed="false" >}}
kubectl get pods
No resources found.
$ kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   10m
{{< /code >}}

Run the controller locally:

{{< code language="bash" isCollapsed="false" >}}
$ make run
/home/rdan/go/bin/controller-gen object:headerFile=./hack/boilerplate.go.txt paths="./..."
go fmt ./...
go vet ./...
/home/rdan/go/bin/controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
go run ./main.go
2019-10-28T18:29:01.335+0200	INFO	controller-runtime.metrics	metrics server is starting to listen	{"addr": ":8080"}
2019-10-28T18:29:01.336+0200	INFO	controller-runtime.controller	Starting EventSource	{"controller": "tfserv", "source": "kind source: /, Kind="}
2019-10-28T18:29:01.336+0200	INFO	controller-runtime.controller	Starting EventSource	{"controller": "tfserv", "source": "kind source: /, Kind="}
2019-10-28T18:29:01.336+0200	INFO	controller-runtime.controller	Starting EventSource	{"controller": "tfserv", "source": "kind source: /, Kind="}
2019-10-28T18:29:01.336+0200	INFO	setup	starting manager
2019-10-28T18:29:01.336+0200	INFO	controller-runtime.manager	starting metrics server	{"path": "/metrics"}
2019-10-28T18:29:01.436+0200	INFO	controller-runtime.controller	Starting Controller	{"controller": "tfserv"}
2019-10-28T18:29:01.537+0200	INFO	controller-runtime.controller	Starting workers	{"controller": "tfserv", "worker count": 1}
2019-10-28T18:29:01.537+0200	DEBUG	controllers.Tfserv	Creating Deployment	{"tfserv": "default/tfserv-sample", "Deployment.Namespace": "default", "Deployment.Name": "tfserv-sample"}
2019-10-28T18:29:01.570+0200	DEBUG	controllers.Tfserv	The Deployment has been created	{"tfserv": "default/tfserv-sample", "Deployment.Name": "tfserv-sample"}
2019-10-28T18:29:01.570+0200	DEBUG	controller-runtime.controller	Successfully Reconciled	{"controller": "tfserv", "request": "default/tfserv-sample"}
2019-10-28T18:29:01.579+0200	DEBUG	controllers.Tfserv	Updating Deployment	{"tfserv": "default/tfserv-sample", "Deployment.Namespace": "default", "Deployment.Name": "tfserv-sample"}
2019-10-28T18:29:01.592+0200	ERROR	controller-runtime.controller	Reconciler error	{"controller": "tfserv", "request": "default/tfserv-sample", "error": "Operation cannot be fulfilled on deployments.apps \"tfserv-sample\": the object has been modified; please apply your changes to the latest version and try again"}
2019-10-28T18:29:02.593+0200	DEBUG	controllers.Tfserv	Updating Deployment	{"tfserv": "default/tfserv-sample", "Deployment.Namespace": "default", "Deployment.Name": "tfserv-sample"}
2019-10-28T18:29:02.604+0200	DEBUG	controllers.Tfserv	Creating a new Service	{"tfserv": "default/tfserv-sample", "Service.Namespace": "tfserv-sample-service", "Service.Name": "tfserv-sample-service"}
2019-10-28T18:29:02.632+0200	DEBUG	controllers.Tfserv	The Service has been created	{"tfserv": "default/tfserv-sample", "Service.Name": "tfserv-sample-service"}
2019-10-28T18:29:02.632+0200	DEBUG	controller-runtime.controller	Successfully Reconciled	{"controller": "tfserv", "request": "default/tfserv-sample"}
{{< /code >}}

Verify again the pods and services and if everithing worked fine you should see something similar with this:

{{< code language="bash" isCollapsed="false" >}}
$ kubectl get deployments
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
tfserv-sample   3/3     3            3           42m

$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
tfserv-sample-555c4fc776-7kh7c   1/1     Running   0          41m
tfserv-sample-555c4fc776-r74rm   1/1     Running   0          41m
tfserv-sample-555c4fc776-tjhgw   1/1     Running   0          41m

$ kubectl get services
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
kubernetes              ClusterIP   10.96.0.1        <none>        443/TCP                         46m
tfserv-sample-service   NodePort    10.103.219.173   <none>        8500:32108/TCP,9000:31235/TCP   40m
{{< /code >}}

Verify if the service suceffuly identified the Endpoints:

{{< code language="bash" isCollapsed="false" >}}
$ kubectl describe services tfserv-sample-service
Name:                     tfserv-sample-service
Namespace:                default
Labels:                   tfsName=tfserv-sample
Annotations:              <none>
Selector:                 tfsName=tfserv-sample
Type:                     NodePort
IP:                       10.103.219.173
Port:                     rest  8500/TCP
TargetPort:               8500/TCP
NodePort:                 rest  32108/TCP
Endpoints:                172.17.0.5:8500,172.17.0.6:8500,172.17.0.7:8500
Port:                     grpc  9000/TCP
TargetPort:               9000/TCP
NodePort:                 grpc  31235/TCP
Endpoints:                172.17.0.5:9000,172.17.0.6:9000,172.17.0.7:9000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
{{< /code >}}

Check out the deployment:

{{< code language="bash" isCollapsed="false" >}}
$ kubectl describe deployments tfserv-sample
Name:                   tfserv-sample
Namespace:              default
CreationTimestamp:      Mon, 28 Oct 2019 18:29:01 +0200
Labels:                 tfsName=tfserv-sample
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               tfsName=tfserv-sample
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  tfsName=tfserv-sample
  Containers:
   tfs-main:
    Image:       tensorflow/serving:latest
    Ports:       9000/TCP, 8500/TCP
    Host Ports:  0/TCP, 0/TCP
    Command:
      /usr/bin/tensorflow_model_server
    Args:
      --port=9000
      --rest_api_port=8500
      --model_config_file=/var/config/models.conf
    Environment:
      GOOGLE_APPLICATION_CREDENTIALS:  /secret/gcp-credentials/test-apps-model-viewer.json
    Mounts:
      /secret/gcp-credentials/ from tfs-secret-volume (rw)
      /var/config/ from tfs-config-volume (rw)
  Volumes:
   tfs-config-volume:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      tf-serving-models-config
    Optional:  false
   tfs-secret-volume:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  tfs-secret
    Optional:    false
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  tfserv-sample-5dc67f948b (3/3 replicas created)
NewReplicaSet:   <none>
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  7m12s  deployment-controller  Scaled up replica set tfserv-sample-5dc67f948b to 3
{{< /code >}}

The pod logs show to us that the tensorflow serving is running and serving the defined model and version and also it is listening for GRPC and REST connections. 

{{< code language="bash" isCollapsed="false" >}}
$ kubectl logs tfserv-sample-5dc67f948b-588ft
2019-10-28 16:29:10.774683: I tensorflow_serving/model_servers/server_core.cc:462] Adding/updating models.
2019-10-28 16:29:10.774725: I tensorflow_serving/model_servers/server_core.cc:573]  (Re-)adding model: resnet
2019-10-28 16:29:15.495255: I tensorflow_serving/core/basic_manager.cc:739] Successfully reserved resources to load servable {name: resnet version: 1538687457}
2019-10-28 16:29:15.496407: I tensorflow_serving/core/loader_harness.cc:66] Approving load for servable version {name: resnet version: 1538687457}
2019-10-28 16:29:15.496556: I tensorflow_serving/core/loader_harness.cc:74] Loading servable version {name: resnet version: 1538687457}
2019-10-28 16:29:15.496605: I external/org_tensorflow/tensorflow/cc/saved_model/reader.cc:31] Reading SavedModel from: gs://tfmodel-store/resnet/1538687457
2019-10-28 16:29:16.299485: I external/org_tensorflow/tensorflow/cc/saved_model/reader.cc:54] Reading meta graph with tags { serve }
2019-10-28 16:29:19.957398: I external/org_tensorflow/tensorflow/cc/saved_model/loader.cc:151] Running initialization op on SavedModel bundle at path: gs://tfmodel-store/resnet/1538687457
2019-10-28 16:29:20.015567: I external/org_tensorflow/tensorflow/cc/saved_model/loader.cc:311] SavedModel load for tags { serve }; Status: success. Took 4518958 microseconds.
2019-10-28 16:29:23.157294: I tensorflow_serving/core/loader_harness.cc:87] Successfully loaded servable version {name: resnet version: 1538687457}
2019-10-28 16:29:23.165306: I tensorflow_serving/model_servers/server.cc:353] Running gRPC ModelServer at 0.0.0.0:9000 ...
[warn] getaddrinfo: address family for nodename not supported
[evhttp_server.cc : 238] NET_LOG: Entering the event loop ...
2019-10-28 16:29:23.168552: I tensorflow_serving/model_servers/server.cc:373] Exporting HTTP/REST API at:localhost:8500 ...
{{< /code >}}

Check if the model is reachable and it's status. Retrieve first the minikube IP and the NodePort.

{{< code language="bash" isCollapsed="false" >}}
$ minikube ip
192.168.99.112

$ kubectl get services tfserv-sample-service
NAME                    TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
tfserv-sample-service   NodePort   10.103.219.173   <none>        8500:32108/TCP,9000:31235/TCP   36m

$ curl http://192.168.99.112:32108/v1/models/resnet
{
 "model_version_status": [
  {
   "version": "1538687457",
   "state": "AVAILABLE",
   "status": {
    "error_code": "OK",
    "error_message": ""
   }
  }
 ]
}
{{< /code >}}

Runnig the example, which is comming with Tensorflow Serving, we are getting **Prediction class: 286**, which coresponds to a **Cat**.
So the model is serving requests over http and grpc.

{{< code language="bash" isCollapsed="false" >}}
$ tools/run_in_docker.sh python tensorflow_serving/example/resnet_client.py 
== Pulling docker image: tensorflow/serving:nightly-devel
nightly-devel: Pulling from tensorflow/serving
Digest: sha256:a847b417e53a093319dc7d04cecf4f2671dbb6e5f99525f72a0ff6aa06ca7234
Status: Image is up to date for tensorflow/serving:nightly-devel
== Running cmd: sh -c 'cd /home/rdan/Downloads/serving; python tensorflow_serving/example/resnet_client.py'
Prediction class: 286, avg latency: 119.1608 ms
{{< /code >}}

## Conclusion
You can find [the complete code](https://github.com/danrusei/tfServing_simple_operator) on Github. I may come back to it and complete the TODOs, maybe extend further on. But the scope is not to create a production ready Operator, it is just for me to get used with the Kubebuilder workflow. I like that the tool generate most of the stuff which allows you to focus more on the business model.
