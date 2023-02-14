
Everything I have learned about Kubernetes.

## Getting started

**[Hướng dẫn Kubernetes cho người mới bắt đầu [KHÓA HỌC ĐẦY ĐỦ trong 4 giờ]](https://www.youtube.com/watch?v=X48VuDVv0do)**


## Introduction
Kubernetes introduces a whole new dictionary of terms; this guide contains some of the basic terms and their definitions as a reference.

### Calico
Calico is an implementation of Kubernetes’ networking model, and it served as the original reference for the Kubernetes NetworkPolicy API during its development. Calico also provides advanced policy enforcement capabilities that extend beyond Kubernetes’ NetworkPolicy API, and these can be used by administrators alongside that API. Calico uses BGP to distribute routes for Kubernetes Pods, without the need for overlays.

### Cluster
A group of servers containing at least one master node and one or more worker nodes.

### Container
Similar to virtual machines, containers are isolated runtimes that you can run your applications and services inside. Containers consume fewer resources than virtual machines, as they do not attempt to emulate a full operating system running on dedicated hardware. Instead, containers only bundle the files, environment variables, and libraries needed by the applications they run, and they share the other resources of the operating system that hosts them.

### Containerization
Containerization is a software architecture practice that organizes applications and their dependencies in containers. Containerizing an application requires a base image that can be used to create an instance of a container. Once an application’s image exists, you can push it to a centralized container registry. Docker, Kubernetes, and other orchestration tools can download images from a registry to deploy container instances.

### Container Storage Interface
The [Container Storage Interface (CSI)](https://github.com/container-storage-interface/spec/blob/master/spec.md) specification provides a common storage interface for container orchestrators like Kubernetes (and others, like Mesos). The interface is used by an orchestrator to attach storage volumes to containers and to manage the lifecycle of those volumes.

The objective of this specification is to allow cloud computing platforms to develop a single storage plugin that works with any container orchestrator. Linode has authored a CSI driver for Linode’s Block Storage service, which makes Block Storage Volumes available to your containers.

### Controller
A Kubernetes Controller is a control loop that continuously watches the Kubernetes API and tries to manage the desired state of certain aspects of the cluster. Examples of different Controllers include:

- ReplicaSets, which manage the number of running instances of a particular Pod.
- Deployments, which manage the number of running instances of a particular Pod and can perform upgrades of Pods to new versions.
- Jobs, which manage Pods that perform one-off tasks.

### Control Plane
kube-apiserver, kube-controller-manager, kube-scheduler, and etcd form what is known as the Control Plane of a Kubernetes cluster. The Control Plane is responsible for keeping a record of the state of a cluster, making decisions about the cluster, and pushing the cluster towards new desired states.

### Deployment
A Deployment can manage a ReplicaSet, so it shares the ability to keep a defined number of replica Pods up and running. A Deployment can also update those Pods to resemble the desired state by means of rolling updates. For example, if you wanted to update a container image to a newer version, you would create a Deployment, and the Controller would update the container images one by one until the desired state is achieved. This ensures that there is no downtime when updating or altering your Pods.

### Docker
Docker is a tool that allows quick deployment of apps in containers using operating system level virtualization. While Kubernetes supports several container runtimes, Docker is a very popular option.

### Dockerfile
A Dockerfile contains all commands, in their required order of execution, needed to build a given Docker image. For example, a Dockerfile might contain instructions for:

- Installing a specific operating system referencing another image,
- Installing an application’s dependencies, and
- Executing configuration commands in the running container.

### Docker Hub
Docker Hub is a centralized container image registry that can host your images and make them available for sharing and deployment. You can also find and use official Docker images and vendor specific images. When combined with a remote version control service, like GitHub, Docker Hub can automate the build process for images and can trigger actions for further automation with other services and tooling.

### etcd
etcd is a highly available key-value store that provides the backend database for Kubernetes.

### Flannel
Flannel is a networking overlay that meets the functionality of the Kubernetes. Flannel supplies a layer 3 network fabric and is relatively easy to set up.

### Helm
Helm is a tool that assists with installing and managing applications on Kubernetes clusters. It is often referred to as “the package manager for Kubernetes,” and it provides functions that are similar to a package manager for an operating system.

### Helm Charts
The software packaging format for Helm. A Helm chart specifies a file and directory structure for packaging your Kubernetes manifests.

### Helm Client
The Helm client software issues commands to your cluster that can install new applications, upgrade them, and delete them. You run the client software on your computer, in your CI/CD environment, or anywhere else you’d like.

### Helm Tiller
A server component that runs on your cluster and receives commands from the Helm client software before its deprecation in Helm 3. Tiller has been removed in Helm 3. Tiller is responsible for directly interacting with the Kubernetes API (which the client software does not do). Tiller maintains the state for your Helm releases.

### Job
A Job is a Controller which manages Pods created for a single or a set of tasks. This is handy if you need to create a Pod that performs a single function, or calculates a value. The deletion of the Job will delete the Pod.

### kube-apiserver
The kube-apiserver is the front end for the Kubernetes API server. Validates and configures data for Kubernetes’ API objects including Pods, Services, Deployments, and more.

### kube-controller-manager
The kube-controller-manager is a daemon that manages the Kubernetes control loop. It watches the shared state of the cluster through the Kubernetes API server.

### kube-proxy
The kube-proxy is a networking proxy that proxies the UDP, TCP, and SCTP networking of each node, and provides load balancing. This is only used to connect to Services.

### kube-scheduler
The kube-scheduler is a function that looks for newly created Pods that have no nodes. kube-scheduler assigns Pods to a nodes based on a host of requirements.

### kubeadm
kubeadm is a cloud provider-agnostic tool that automates many of the tasks required to get a cluster up and running. Users of kubeadm can run a few simple commands on individual servers to turn them into a Kubernetes cluster consisting of a master node and worker nodes.

### kubectl
kubectl is a command line tool used to interact with the Kubernetes cluster. It offers a host of features, including:

- Creating, stopping, and deleting resources
- Describing active resources
- Auto scaling resources.

### kubelet
kubelet is an agent that receives descriptions of the desired state of a Pod from the API server and ensures the Pod is healthy and running on its node.

### Kubernetes
Kubernetes, often referred to as “k8s”, is an open source container orchestration system that helps deploy and manage containerized applications. Developed by Google starting in 2014 and written in the Go language, Kubernetes is quickly becoming the standard way to architect horizontally-scalable applications.

### Kubernetes Manifests
Files, often written in YAML, used to create, modify, and delete Kubernetes resources such as Pods, Deployments, and Services.

### Linode NodeBalancer
NodeBalancers are highly-available, managed, cloud based “load balancers as a service”. They intelligently route incoming requests to backend Linodes to help your application cope with load, and to increase your application’s availability.

### Master Server
The Kubernetes Master is normally a separate server in a Kubernetes cluster responsible for maintaining the desired state of the cluster. It does this by telling the nodes how many instances of your application it should run and where.

### Namespaces
Namespaces are virtual clusters that exist within the Kubernetes cluster that help to group and organize Kubernetes API objects. Every cluster has at least three Namespaces: default, kube-system, and kube-public. When interacting with the cluster it is important to know which Namespace the object you are looking for is in, as many commands will default to only showing you what exists in the default Namespace. Resources created without an explicit Namespace will be added to the default Namespace.

### Orchestration
Orchestration is the automated configuration, coordination, and management of computer systems, software, middleware, and services. It takes advantage of automated tasks to execute processes. The subject of orchestration is often discussed in reference to lifecycle management for containers, a practice known as container orchestration.

### Pod
A Pod is the smallest deployable unit of computing in the Kubernetes architecture. A Pod is a group of one or more containers with shared resources and a specification for how to run these containers. Each Pod has its own IP address in the cluster. Pods are “mortal,” which means that they are created and destroyed depending on the needs of the application

### Rancher
Rancher is a web application that provides a GUI interface for cluster creation and for the management of clusters. Rancher also provides easy interfaces for deploying and scaling apps on your clusters, and it has a built-in catalog of curated apps to choose from.

### ReplicaSet
A ReplicaSet is one of the Controllers responsible for keeping a given number of replica Pods running. If one Pod goes down in a ReplicaSet, another will be created to replace it. In this way, Kubernetes is self-healing. However, for most use cases it is recommended to use a Deployment instead of a ReplicaSet.

### REST
REST stands for REpresentational State Transfer. It is an architectural style for network based software that requires stateless, cacheable, client-server communication via a uniform interface between components. The HTTP protocol is most often used in RESTful applications.

### Services
Services group identical Pods together to provide a consistent means of accessing them. Each service is given an IP address and a corresponding DNS entry. Services exist across nodes. There are four types of Services:

- ClusterIP: exposes the Service internally to the cluster; this is the default type of Service.

- NodePort: exposes the Service to the internet from the IP address of the node at the specified port number, which is in the range 30000-32767.

- LoadBalancer: creates a load balancer assigned to a fixed IP address in the cloud if the cloud provider supports it. For clusters deployed on Linode, this is the responsibility of the Linode’s Cloud Controller Manager (CCM), which will create NodeBalancers for each of your LoadBalancer services. This is the best way to expose your cluster to the internet.

- ExternalName: maps the Service to a DNS name by returning a CNAME record redirect. ExternalName is good for directing traffic to outside resources, such as a database hosted on another cloud.

### Terraform
Terraform by HashiCorp is a software tool that allows you to represent your Linode instances and other resources with declarative code inside configuration files, instead of manually creating those resources via the Linode Manager or API. This practice is referred to as Infrastructure as Code, and Terraform is a popular example of this methodology.

### Volumes
A Volume in Kubernetes is a way to share file storage between containers in a Pod. Kubernetes Volumes differ from Docker volumes because they exist inside the Pod rather than inside the container.

### Worker Nodes
Worker nodes in a Kubernetes cluster are servers that run your applications’ Pods. The number of nodes in your cluster is determined by the cluster administrator.

### More Information
You may wish to consult the following resources for additional information on this topic. While these are provided in the hope that they will be useful, please note that we cannot vouch for the accuracy or timeliness of externally hosted materials.

- [Kubernetes API Documentation](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.17/)
- [Kubernetes Concepts Documentation](https://kubernetes.io/docs/concepts/)
