> # Kubernetes Reference Guide: 
> [Official Kubernetes documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
> [Linode - Using kubeadm to Deploy a Kubernetes Cluster](https://www.linode.com/docs/guides/getting-started-with-kubernetes/)


# Before You Begin
Deploy three Linodes running Ubuntu 18.04 with a minimum of the following system requirements:

- One Linode to use as the master Node with 4GB RAM and 2 CPU cores.
- Two Linodes to use as the worker Nodes each with 2GB RAM and 1 CPU core.
- Follow the Getting Started and the Setting Up and Securing a Compute Instance guides for instructions on setting up your Linodes. The steps in this guide assume the use of a limited user account with sudo privileges.

> Note
When following the Getting Started guide, make sure that each Linode is using a different hostname. Not following this guideline leaves you unable to join some or all nodes to the cluster in a later step.
Disable swap memory on your Linodes. Kubernetes requires that you disable swap memory on any cluster nodes to prevent the kube-scheduler from assigning a Pod to a node that has run out of CPU/memory or reached its designated CPU/memory limit.


```
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

Verify that your swap has been disabled. You should expect to see a value of 0 returned.

```
cat /proc/meminfo | grep 'SwapTotal'
```

To learn more about managing compute resources for containers, see the official Kubernetes documentation.

Read the Beginners Guide to Kubernetes to familiarize yourself with the major components and concepts of Kubernetes. The current guide assumes a working knowledge of common Kubernetes concepts and terminology.

# Build a Kubernetes Cluster

A Kubernetes cluster consists of a master node and worker nodes. The master node hosts the control plane, which is the combination of all the components that provide it the ability to maintain the desired cluster state. This cluster state is defined by manifest files and the kubectl tool. While the control plane components can be run on any cluster node, it is a best practice to isolate the control plane on its own node and to run any application containers on a separate worker node. A cluster can have a single worker node or up to 5000. Each worker node must be able to maintain running containers in a Pod and be able to communicate with the master node’s control plane.

The following table provides a list of the Kubernetes tooling you need to install on your master and worker nodes in order to meet the minimum requirements for a functioning Kubernetes cluster as described above.

| Tool              |	Master Node |	Worker Nodes|
|-------------------|-------------|-------------|
| kubeadm	          | x          	| x           |
| Container Runtime	| x	          | x           |
| kubelet	          | x	          | x           |
| kubectl	          | x	          | x           |
| Control Plane	    | x           |             |


> Note
The control plane is a series of services that form Kubernetes master structure that allow it to control the cluster. The kubeadm tool allows the control plane services to run as containers on the master node. The control plane is created when you initialize kubeadm later in this guide.

# Install the Container Runtime

Remove any older installations of Docker that may be on your system:

```
sudo apt remove docker docker.io containerd runc
```

Make sure you have the necessary packages to allow the use of Docker’s repository:

```
sudo apt install apt-transport-https ca-certificates curl software-properties-common gnupg curl lsb-release
```

Add Docker’s GPG key:

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

Verify the fingerprint of the GPG key:

```
sudo apt-key fingerprint 0EBFCD88
```


You should see the following output:

```
pub   4096R/0EBFCD88 2017-02-22
        Key fingerprint = 9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid                  Docker Release (CE deb) <docker@docker.com>
sub   4096R/F273FCD8 2017-02-22
```

Add the stable Docker repository:

```
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

Update your package index and install Docker CE. For more information, see the Docker instructions within the Kubernetes setup guide.

```
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Alternatively, you can install specific versions of the software if you wish to prioritize stability. The following example installs specific versions, though you may wish to find the latest validated versions within Kubernetes dependencies file.

```
sudo apt-get update && sudo apt-get install -y \
containerd.io=1.2.13-2 \
docker-ce=5:19.03.11~3-0~ubuntu-$(lsb_release -cs) \
docker-ce-cli=5:19.03.11~3-0~ubuntu-$(lsb_release -cs)
```

Add your limited Linux user account to the docker group. Replace $USER with your username:

```
sudo usermod -aG docker $USER
```

>Note
After entering the usermod command, you need to close your SSH session and open a new one for this change to take effect.


Setup the Docker daemon to use systemd as the cgroup driver, instead of the default cgroupfs. This is a recommended step so that kubelet and Docker are both using the same cgroup manager. This makes it easier for Kubernetes to know which resources are available on your cluster’s nodes.

```
sudo bash -c 'cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF'
```

Create a systemd directory for Docker:

```
sudo mkdir -p /etc/systemd/system/docker.service.d
```

Restart Docker:

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

To ensure that Docker is using systemd as the cgroup driver, enter the following command:

``` 
sudo docker info | grep -i cgroup
```

You should see the following output:

```
Cgroup Driver: systemd
Cgroup Version: 1
```

# Install the cri-dockerd Service (Optional)
Although previously an unnecessary step when using Docker as a container runtime, as of Kubernetes v1.24, the Dockershim adapter service was officially removed from Kubernetes. In order to prepare for this change, Mirantis and Docker have worked together to create an adapter service called cri-dockerd to continue support for Docker as a container runtime. Installing the cri-dockerd service is a necessary step on all clusters using Kubernetes version 1.24 or later, and should be performed when following the steps in this guide:

Install the go programming language to support later commands performed during the installation process:

```
sudo apt install golang-go
```

Clone the cri-dockerd repository and change your working directory into the installation path:

```
cd && git clone https://github.com/Mirantis/cri-dockerd.git
```

Build the code:

```
sudo mkdir bin
cd cri-dockerd
mkdir bin
go build -o bin/cri-dockerd
mkdir -p /usr/local/bin
install -o root -g root -m 0755 bin/cri-dockerd /usr/local/bin/cri-dockerd
cp -a packaging/systemd/* /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
systemctl daemon-reload
systemctl enable cri-docker.service
systemctl enable --now cri-docker.socket
```

# Install kubeadm, kubelet, and kubectl
Complete the steps outlined in this section on all three Linodes.

Add the required GPG key to your apt-sources keyring to authenticate the Kubernetes related packages you install:

```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg  | sudo apt-key add -
```

Add Kubernetes to the package manager’s list of sources:

```
sudo bash -c "cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF"
```

Update apt, install kubeadm, kubelet, and kubectl, and hold the installed packages at their installed versions:

```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Verify that kubeadm, kubelet, and kubectl have installed by retrieving their version information. Each command should return version information about each package.

```
kubeadm version
kubelet --version
kubectl version
```

# Set up the Kubernetes Control Plane
After installing the Kubernetes related tooling on all your Linodes, you are ready to set up the Kubernetes control plane on the master node. The control plane is responsible for allocating resources to your cluster, maintaining the health of your cluster, and ensuring that it meets the minimum requirements you designate for the cluster.

The primary components of the control plane are the kube-apiserver, kube-controller-manager, kube-scheduler, and etcd. You can easily initialize the Kubernetes master node with all the necessary control plane components using kubeadm. For more information on each of control plane component see the Beginner’s Guide to Kubernetes.

In addition to the baseline control plane components, there are several addons, that can be installed on the master node to access additional cluster features. You need to install a networking and network policy provider addon that implements the Kubernetes’ network model on the cluster’s Pod network.

This guide uses Calico as the Pod network add on. Calico is a secure and open source L3 networking and network policy provider for containers. There are several other network and network policy providers to choose from. To view a full list of providers, refer to the official Kubernetes documentation.

> Note
kubeadm only supports Container Network Interface (CNI) based networks. CNI consists of a specification and libraries for writing plugins to configure network interfaces in Linux containers
Initialize kubeadm on the master node. This command runs checks against the node to ensure it contains all required Kubernetes dependencies. If the checks pass, kubeadm installs the control plane components.

When issuing this command, it is necessary to set the Pod network range that Calico uses to allow your Pods to communicate with each other. It is recommended to use the private IP address space, 10.2.0.0/16. Additionally, the CRI connection socket will need to be manually set, in this case to use the socket path to cri-dockerd.

>Note
The Pod network IP range should not overlap with the service IP network range. The default service IP address range is 10.96.0.0/12. You can provide an alternative service ip address range using the --service-cidr=10.97.0.0/12 option when initializing kubeadm. Replace 10.97.0.0/12 with the desired service IP range:

For a full list of available kubeadm initialization options, see the [Official Kubernetes documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/).

If use `cri-dockerd.sock`
```
sudo kubeadm init --pod-network-cidr=10.2.0.0/16 --cri-socket=unix:///var/run/cri-dockerd.sock
```

If use `containerd.sock`

```
sudo kubeadm init --pod-network-cidr=10.2.0.0/16 --cri-socket=unix:///var/run/containerd/containerd.sock --upload-certs
```

You should see a similar output:

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.0.2.0:6443 --token udb8fn.nih6n1f1aijmbnx5 \
    --discovery-token-ca-cert-hash sha256:b7c01e83d63808a4a14d2813d28c127d3a1c4e1b6fc6ba605fe4d2789d654f26 
```


The kubeadm join command will be used in the Join a Worker Node to the Cluster section of this guide to bootstrap the worker nodes to the Kubernetes cluster. This command should be kept handy for later use. Below is a description of the required options you will need to pass in with the kubeadm join command:

The master node’s IP address and the Kubernetes API server’s port number. In the example output, this is 192.0.2.0:6443. The Kubernetes API server’s port number is 6443 by default on all Kubernetes installations.
A bootstrap token. The bootstrap token has a 24-hour TTL (time to live). A new bootstrap token can be generated if your current token expires.
A CA key hash. This is used to verify the authenticity of the data retrieved from the Kubernetes API server during the bootstrap process.
Copy the admin.conf configuration file to your limited user account. This file allows you to communicate with your cluster via kubectl and provides superuser privileges over the cluster. It contains a description of the cluster, users, and contexts. Copying the admin.conf to your limited user account provides you with administrative privileges over your cluster.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

If you see this error:

`[ERROR CRI]: container runtime is not running: output: time="2023-03-06T03:16:20Z" level=fatal msg="validate service connection: CRI v1 runtime API is not implemented for endpoint \"unix:///var/run/containerd/containerd.sock\": rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService"`

=> The `containerd.service` is not work correctly. To resolve it:


- remove config 
```
rm /etc/containerd/config.toml
```

- Restart containerd: 
```
systemctl restart containerd
```

Install the necessary CNI(Container Network Interface) manifests to your master node and apply them using kubectl (In this case we use `Flannel` or `Weave`):

**Wave**

See installation from [Weave.works](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/)


**Flannel**

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

If the command above return Error 404, create  `k8s-network-plugin-control-plane.yml`

content: 

```
---
kind: Namespace
apiVersion: v1
metadata:
  name: kube-flannel
  labels:
    pod-security.kubernetes.io/enforce: privileged
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
- apiGroups:
  - "networking.k8s.io"
  resources:
  - clustercidrs
  verbs:
  - list
  - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-flannel
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-flannel
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni-plugin
        image: docker.io/flannel/flannel-cni-plugin:v1.1.2
       #image: docker.io/rancher/mirrored-flannelcni-flannel-cni-plugin:v1.1.2
        command:
        - cp
        args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        volumeMounts:
        - name: cni-plugin
          mountPath: /opt/cni/bin
      - name: install-cni
        image: docker.io/flannel/flannel:v0.20.2
       #image: docker.io/rancher/mirrored-flannelcni-flannel:v0.20.2
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: docker.io/flannel/flannel:v0.20.2
       #image: docker.io/rancher/mirrored-flannelcni-flannel:v0.20.2
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
        - name: xtables-lock
          mountPath: /run/xtables.lock
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni-plugin
        hostPath:
          path: /opt/cni/bin
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
```

Add Flannel config: 

create file `/run/flannel/subnet.env`

```
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```

Apply network plugin:

```
kubectl apply -f k8s-network-plugin-control-plane.yml
```



# Inspect the Master Node with Kubectl

After completing the previous section, your Kubernetes master node is ready with all the necessary components to manage a cluster. To gain a better understanding of all the parts that make up the master’s control plane, this section walks you through inspecting your master node. If you have not yet reviewed the Beginner’s Guide to Kubernetes, it is helpful to do so prior to continuing with this section as it relies on the understanding of basic Kubernetes concepts.

View the current state of all nodes in your cluster. At this stage, the only node you should expect to see is the master node, since worker nodes have yet to be bootstrapped. A STATUS of Ready indicates that the master node contains all necessary components, including the Pod network add-on, to start managing clusters.

```
kubectl get nodes
```
Yo
ur output should resemble the following:

```
NAME          STATUS     ROLES     AGE   VERSION
kube-master   Ready      master    1h    v1.14.1
```

# Inspect the available namespaces in your cluster.

```
kubectl get namespaces
```
Your output should resemble the following:

```
NAME              STATUS   AGE
default           Active   23h
kube-node-lease   Active   23h
kube-public       Active   23h
kube-system       Active   23h
```

Below is an overview of each namespace installed by default on the master node by kubeadm:

`default`: The default namespace contains objects with no other assigned namespace. By default, a Kubernetes cluster will instantiate a default namespace when provisioning the cluster to hold the default set of Pods, Services, and Deployments used by the cluster.
`kube-system`: The namespace for objects created by the Kubernetes system. This includes all resources used by the master node.
`kube-public`: This namespace is created automatically and is readable by all users. It contains information, like certificate authority data (CA), that helps kubeadm join and authenticate worker nodes.
`kube-node-lease`: The kube-node-lease namespace contains lease objects that are used by kubelet to determine node health. kubelet creates and periodically renews a Lease on a node. The node lifecycle controller treats this lease as a health signal. kube-node-lease was released to beta in Kubernetes 1.14.


View all resources available in the kube-system namespace. The kube-system namespace contains the widest range of resources, since it houses all control plane resources. Replace kube-system with another namespace to view its corresponding resources.

```
kubectl get all -n kube-system
```


# Join a Worker Node to the Cluster

Now that your Kubernetes master node is set up, you can join worker nodes to your cluster. In order for a worker node to join a cluster, it must trust the cluster’s control plane, and the control plane must trust the worker node. This trust is managed via a shared bootstrap token and a certificate authority (CA) key hash. kubeadm handles the exchange between the control plane and the worker node. At a high-level the worker node bootstrap process is the following:

kubeadm retrieves information about the cluster from the Kubernetes API server. The bootstrap token and CA key hash are used to ensure the information originates from a trusted source.

kubelet can take over and begin the bootstrap process, since it has the necessary cluster information retrieved in the previous step. The bootstrap token is used to gain access to the Kubernetes API server and submit a certificate signing request (CSR), which is then signed by the control plane.

The worker node’s kubelet is now able to connect to the Kubernetes API server using the node’s established identity.

Before continuing, you need to make sure that you know your Kubernetes API server’s IP address, that you have a bootstrap token, and a CA key hash. This information was provided when kubeadm was initialized on the master node in the Set up the Kubernetes Control Plane section of this guide. If you no longer have this information, you can regenerate the necessary information from the master node.


#### Regenerate a Bootstrap Token

When you lost the join command. Don't worry!
These commands should be issued from your master node.
Generate a new bootstrap token and display the kubeadm join command with the necessary options to join a worker node to the master node’s control plane:

```
kubeadm token create --print-join-command
```

Join command should look like this: 

```
kubeadm join 192.0.2.0:6443 --token udb8fn.nih6n1f1aijmbnx5 \
    --discovery-token-ca-cert-hash sha256:b7c01e83d63808a4a14d2813d28c127d3a1c4e1b6fc6ba605fe4d2789d654f26
``` 

Copy join command and specify node name for this node by add new parameter: `--node-name=worker-1`

When the bootstrap process has completed, you should see a similar output:

```
  This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

SSH into the master node and verify the worker nodes have joined the cluster:

```
kubectl get nodes
```

You should see a similar output.

```
NAME          STATUS   ROLES    AGE     VERSION
kube-master   Ready    master   1d22h   v1.14.1
kube-node-1   Ready    <none>   1d22h   v1.14.1
kube-node-2   Ready    <none>   1d22h   v1.14.1
```

# ENJOY!