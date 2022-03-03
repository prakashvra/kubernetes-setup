In this session we will see how to setup a k8s cluster on AWS EC2 hosts using kubeadm.

We will see in a series of steps 
- Where to start?
- What applications to install?
- How do we install these applications?

## Where and what applications?

Let us consider a basic cluster setup with 1 master node and 2 worker nodes.

The master node runs the Control Plane and the worker nodes run the actual workloads

Install these 2 apps on both master and worker nodes

- Container runtime
- Kubelet

These 2 apps run as regular linux processes. They can be installed from the package repository like any other linux packages

They need to be installed on all the nodes.

### Master Node

The master node runs the Control Plane along with the other necessary apps

Control plane consists of the following apps which run as Pods.

- API server
- Controller Manager
- Scheduler 
- Etcd

Along with the above we need to install Kubeproxy which also runs as a Pod.

### Worker Node

Each of the worker nodes also should run Kubeproxy.

Now, what are the main things required to deploy these Pods. 

- Kubernetes manifest files for each of the apps
- Security setup to prevent unauthorized access to the apps and securing the communication between the apps

How do we setup the apps in the master node? Because we said all the control plane apps run as Pods. Without the control plane in place how are we going to deploy and run these Pods? 

To solve this problem, we have something called Static Pods.

### Static Pods

These are pods managed directly by the kubelet daemon. So, the request for deploying a pod does not go through the API server in the control plane which is the case for any regular pod.

How does kubelet daemon do this?

Kubelet looks for any manifests available in a specific location on the Node where it is running.

`/etc/kubernetes/manifests
`
If the kubelet finds a Pod manifest in that location, it schedules a Pod directly on the node without having to go through the API server on the control plane running on the master node.

Kubelet also restarts the static pods in case if the pods crash.

### Creating static Pod manifests

All static pod manifests have to be created and dropped into this location `/etc/kubernetes/manifests` on their respective nodes.

### Securing the pods

Securing the communication between the apps within the cluster is one of the key aspects.

K8s uses TLS certificate based security for this purpose.

In order to decide which certificate will be used in each case we need to understand the communication flow between various components.

First, most of all the components talk to the API server as it is the centre of all the communications. So, every other app that wants to talk to API server needs to send a TLS certificate. 

For learning purposes we can generate self-signed CA certificates for all the different apps. 

As API server is the one which receives requests, we need to setup server certificate. For all other apps which connect to API server, client certificates are to be setup.

In cases where the API server connects to other apps, client certificate is required for API server too.

- Server and client certificate for API server
- Client certificate for Controller manager and Scheduler
- Server and client certificate for Kubelet
- Server certificate for Etcd

The certificates have to be stored on the respective nodes in this location /`etc/kubernetes/pki`

We also need to setup a certificate for the K8s admin users who will access the K8s cluster through API or CLI. This will be a client certificate.

We need to generate a self-signed certificate which will be the root. Then generate certs for all the apps and sign them with the root.

So, in order to get the k8s cluster up and running, we need to setup all these stuff.

- All mandatory process for master and worker nodes
- Manifest files for all pods
- Certificates for securing the apps

We know by experience that these can take a lot of time if we do it manually. Luckily there is a CLI tool called Kubeadm. 

### Kubeadm

[Kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) is a tool created and maintained by Kubernetes. A very detailed documentation is available for this tool.

This CLI tool helps in bootstrapping the k8s cluster by creating the necessary artifacts and performing the necessary actions.

### Prerequisites

For this session, we are going to setup our k8s cluster on AWS infrastructure. It is fine if you have a preference to use your own hardware/VM or any other cloud hosted infra.

You can refer to kubeadm document [here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin)

The following are the infrastructure requirements

- Any Linux based hosts. We will be using the latest Ubuntu VM available on AWS EC2.

- Capacity of the servers - 2 CPUs and 2 GB or greater. We will choose t2.medium type as it provides 2 vCPU and 4 GiB memory.We will be creating 1 master node and 2 worker nodes and all 3 nodes will use the t2.medium type instances.

- All instances should be part of the same network. In this case all the EC2 instances will be part of a single VPC.

- Should have unique host name and MAC address. All EC2 instances will have unique hostnames.

- Required ports have to be opened. Please use the details [here ](https://kubernetes.io/docs/reference/ports-and-protocols/)to configure open the required ports.
 
- Memory swapping should be disabled. Kubelets are designed to run the pods within the memory available on the host. Keeping swap enabled may destabilise the k8s cluster.

Now the we understand the pre-requisites, we can start the installation!

## How do we install?

### _Step-1_ : Setup the hosts

We need 3 hosts for creating the k8s cluster. We will be using AWS EC2 instances. You can sign up for [AWS Free tier](https://aws.amazon.com/free/). Please read through the free tier details. Once you have setup the k8s cluster and finished practicing, be sure to delete the EC2 instances so that you don't incur any unexpected cost.

#### Creating hosts

1. Sign up for a new [AWS Free tier](https://aws.amazon.com/free/) account if you don't have one.
2. Create 3 t2.medium EC2 instances
3. Name 1 instance as master and the other 2 instances as worker1 and worker2.
4. Launch the instances. Don't forget to download the key-pair pem file as you will need this to ssh into the instances . Refer AWS documentation [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/launching-instance.html#step-7-review-instance-launch)

Note:- You can use the same keypair to access all the instances.

Once you launch a Public IP will be assigned to each of those instances. Make sure you are able to SSH to the instances using the public IP. Use the following command from your laptop terminal to login.

```
ssh -i k8s-cluster.pem ubuntu@54.179.167.247

```
#### Setting up pre-requisites

_Disable memory swapping_

SSH to each of the instances and execute this command.

```
sudo swapoff -a
```

_Open required ports_

By default AWS opens only port 22 for incoming SSH connections. So in order for all the k8s components running on 3 instances to communicate to each other, other ports need to be opened as per this [kubeadm reference](https://kubernetes.io/docs/reference/ports-and-protocols/) 

Use the AWS security groups to configure the inbound ports. Refer to [AWS EC2 Security Groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/working-with-security-groups.html)

So far, we have provisioned the infrastructure and configured the necessary pre-requisites.

### _Step-2_ : Setup container runtime

Container runtime has to be installed on master node and all the worker nodes.

We will be using [containerd](https://containerd.io/) as the container runtime as this is the standard recognised and used by all the managed k8s cluster providers [EKS](https://docs.aws.amazon.com/eks/index.html), [AKS](https://azure.microsoft.com/en-in/services/kubernetes-service/) and [GKE](https://cloud.google.com/kubernetes-engine)

Please refer to [Container Runtime Interface(CRI)](https://kubernetes.io/docs/concepts/architecture/cri/) if you want to refresh the concepts around the CRI.

Ok. Let us install the container runtime.

We will be following the steps given [here](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd).

The container installation steps are captured in a script file that can be found [here](https://github.com/prakashvra/kubernetes-setup/blob/3c47dd70ecc5d301a14034cfcbcbdf8c2a43da69/cri-install.sh)

Run the install script on all the nodes.

### _Step-3_ : Install kubeadm, kubelet and kubectl

Now we need to install kubeadm, kubelet and kubectl on all the nodes. 

As mentioned earlier kubeadm will be used to initialise the cluster. It will create the necessary components for the control plane.

Kubelet is required for starting pods and containers

Kubectl is required for communicating with the cluster

We will be following the steps given [here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl).

The installation steps are captured in a script file that can be found [here](https://github.com/prakashvra/kubernetes-setup/blob/0487602e14857bfed3c35bf5bcdde4e505be941f/kubeadm-install.sh)

Run the above script on all the nodes.

After completing the installations, check the status of each of the components by executing these commands

- kubeadm version
- kubectl version
- service kubelet status

### _Step-3_ : Initialise the cluster

We will be using the kubeadm command to initialise the cluster. 

On the master node, execute the following command.

```
sudo kubeadm init
```

Once the initialisation completes successfully, you should see some output like the following.

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.31.17.66:6443 --token i1yboi.66y4mulp2gtblrvv \
	--discovery-token-ca-cert-hash sha256:4ddfa86a2cf2df7aaf21858c685e98ee2d9687252e2a2ef46df0c96223671504
```

In the output you will notice the different steps that were executed to setup the control plane and other processes.

First executes `preflight` step to check and pull the necessary images.
```
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
```

Creates the certificates for all the k8s components.

```
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master] and IPs [10.96.0.1 172.31.17.66]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost master] and IPs [172.31.17.66 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost master] and IPs [172.31.17.66 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
```

Creates the necessary kube config files that connect to the API server.
```
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
```

Configures the kubelet config and starts the kubelet service
```
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
```

Generates the manifest files for static pods and starts the pods.
```
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
```

Sets up the manifest for etcd
```
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
```

Creates a ConfigMap called kubeadm-config in the kube-system namespace. This configuration is later used during `kubeadm join`, `kubeadm-reset` and `kubeadm-upgrade`
```
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
```

A kubelet config is created in kube-system namespace for all the kubelets in the cluster
```
[kubelet] Creating a ConfigMap "kubelet-config-1.23" in namespace kube-system with the configuration for the kubelets in the cluster
```

Adds a taint to prevent any workload pods from being scheduled on the master node.
```
[mark-control-plane] Marking the node master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
```

Then, creates bootstrap tokens to be used along with RBAC roles to enable authentication in the API server and used when a new cluster is created or joining new nodes to the cluster.
```
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
```

Sets up CoreDNS as the DNS server and kube-proxy which is a network proxy that contains network rules on nodes and allows the traffic to the pods from inside and outside of cluster.
```
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy
```

With this, we have completed the cluster initialisation step!!!

### _Step-4_ : Connecting to the cluster

We know that kubectl is the CLI tool to talk to the cluster. So, how does kubectl know to connect to the cluster.

Let us do a quick check. Execute this command on the master node.
```
ubuntu@master:~$ kubectl get nodes
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```
kubectl is unable to connect as the kubeconfig is not setup yet.

To make it work, we need to copy the kubeconfig to a default location on the node from where kubectl can use it.

Execute the following commands.
```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Now, if you check the nodes, you should see an output like this.
```
ubuntu@master:~$ kubectl get nodes
NAME     STATUS     ROLES                  AGE   VERSION
master   NotReady   control-plane,master   11h   v1.23.4
```

The status of the master node is showing as 'NotReady' because the pod networking is not setup. So let us setup the pod network in the next step.

### _Step-5_ : Setup pod network

You can refer to [k8s networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model) to understand the fundamental concepts.

There are a number of networking solutions available. For this session we are going to use [calico](https://projectcalico.docs.tigera.io/about/about-calico).

Let us follow these steps to install calico on k8s. This installation has to be done on the master node.

#### _Pre-requisites_

Check the calico [network requirements](https://projectcalico.docs.tigera.io/getting-started/kubernetes/requirements#network-requirements) 

In this session, we are using the default BGP bidirectional networking. For all the nodes, configure the inbound rules in security groups to allow TCP port 179. Only allow the VPC IP range for eg `172.31.0.0/16`. 

#### _Installation_

Download the calico manifest file.

```
curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O

```
By default calico is configured to use 192.168.0.0/16 for the pod network. This should not overlap with the node IP range. If you are using this IP range already, make the necessary changes to the calico manifest.

Uncomment these 2 lines in calico.yaml file if you want to use the default CIDR range.

`- name: CALICO_IPV4POOL_CIDR
  value: "192.168.0.0/16"`

Now, apply the manifest.

```
kubectl apply -f calico.yaml

```
After the installation completes, you can verify by checking the mode status and the calico pods created and running.

```
kubectl get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   3d    v1.23.4

```

Checking the pods.

```
kubectl get pod -n kube-system
calico-kube-controllers-566dc76669-dcnbq   1/1     Running   0          56s
calico-node-nkhj9                          1/1     Running   0          56s
coredns-64897985d-28rdb                    1/1     Running   0          3d
coredns-64897985d-82m9d                    1/1     Running   0          3d
etcd-master                                1/1     Running   0          3d
kube-apiserver-master                      1/1     Running   0          3d
kube-controller-manager-master             1/1     Running   0          3d
kube-proxy-d25qg                           1/1     Running   0          3d
kube-scheduler-master                      1/1     Running   0          3d
```

Notice the calico pods running and coredns pods in running state.

We have successfully created a pod network for a single node cluster. We still need to bring the worker nodes into the pod network. To do this we need to join the worker nodes to the cluster.

### _Step-6_ : Add worker nodes to cluster

Use the `kubadm join` command shown in the result of Step-3 to join the worker nodes to cluster.

```
kubeadm join 172.31.17.66:6443 --token l87fxe.30820pct4o7k80ax --discovery-token-ca-cert-hash sha256:4ddfa86a2cf2df7aaf21858c685e98ee2d9687252e2a2ef46df0c96223671504 

```

After execution of the command, you can verify if the nodes are joined in the cluster by executing the following in master node.
```
ubuntu@master:~$ kubectl get nodes
NAME      STATUS     ROLES                  AGE     VERSION
master    Ready      control-plane,master   3d1h    v1.23.4
worker1   Ready      <none>                 3m53s   v1.23.4
worker2   NotReady   <none>                 18s     v1.23.4
```
### _Step-7_ : Checking calico networking is setup for all nodes

You can see 3 calico-node pods are running.

```
ubuntu@master:~$ kubectl get pod -A  -o wide
NAMESPACE         NAME                                       READY   STATUS    RESTARTS   AGE    IP               NODE      NOMINATED NODE   READINESS GATES
kube-system       calico-kube-controllers-566dc76669-fss4z   1/1     Running   0          30s    172.16.189.65    worker2   <none>           <none>
kube-system       calico-node-b7mwj                          1/1     Running   0          31s    172.31.28.225    worker2   <none>           <none>
kube-system       calico-node-ln7g2                          1/1     Running   0          31s    172.31.17.66     master    <none>           <none>
kube-system       calico-node-qkg8c                          1/1     Running   0          31s    172.31.22.174    worker1   <none>           <none>
kube-system       coredns-64897985d-28rdb                    1/1     Running   0          4d9h   192.168.219.65   master    <none>           <none>
kube-system       coredns-64897985d-82m9d                    1/1     Running   0          4d9h   192.168.219.66   master    <none>           <none>
kube-system       etcd-master                                1/1     Running   0          4d9h   172.31.17.66     master    <none>           <none>
kube-system       kube-apiserver-master                      1/1     Running   0          4d9h   172.31.17.66     master    <none>           <none>
kube-system       kube-controller-manager-master             1/1     Running   0          4d9h   172.31.17.66     master    <none>           <none>
kube-system       kube-proxy-d25qg                           1/1     Running   0          4d9h   172.31.17.66     master    <none>           <none>
kube-system       kube-proxy-nqwkm                           1/1     Running   0          32h    172.31.28.225    worker2   <none>           <none>
kube-system       kube-proxy-nvn4d                           1/1     Running   0          32h    172.31.22.174    worker1   <none>           <none>
kube-system       kube-scheduler-master                      1/1     Running   0          4d9h   172.31.17.66     master    <none>           <none>
```

Let us create a nginx pod to check if the IP allocation works as expected.

Create a nginx Pod manfiest.

```
apiVersion: v1
kind: Pod
metadata:
   name: nginx
   namespace: default
spec:
   containers:
   - name: nginx
     image: nginx
     ports:
       - containerPort: 80
```

Create the nginx pod

```
kubectl create -f nginx.yaml
```

View the pod details
```
kubectl get pods -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP               NODE      NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          13s   192.168.189.65   worker2   <none>           <none>
```

You can see the newly created nginx pod is assigned an IP from the CIDR block 192.168.0.0/16

Let us create another pod using another simpler method.

```
kubectl run nginx2 --image=nginx
```

View the pod details now
```
kubectl get pods -o wide
NAME     READY   STATUS    RESTARTS   AGE   IP                NODE      NOMINATED NODE   READINESS GATES
nginx    1/1     Running   0          57m   192.168.189.65    worker2   <none>           <none>
nginx2   1/1     Running   0          7s    192.168.235.130   worker1   <none>           <none>
```

You can see the new pod 'nginx2' is scheduled on worker1 and the IP is within the CIDR range 192.168.0.0/16

With this we have completed a basic k8s cluster setup and tested it.

In the next article we will deploy a sample "go lang" application.
