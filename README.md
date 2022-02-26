# Kubernetes cluster setup

Where to start?

What applications to install?

How do we install these applications?

Let us consider a basic cluster setup with 1 master node and 2 worker nodes.

The master node runs the Control Plane and the worker nodes run the actual workloads

Install these 2 apps on both master and worker nodes

- Container runtime
- Kubelet

These 2 apps run as regular linux processes. They can be installed from the package repository like any other linux packages

They need to be installed on all the nodes.

## Master Node

The master node runs the Control Plane along with the other necessary apps

Control plane consists of the following apps which run as Pods.

- API server
- Controller Manager
- Scheduler 
- Etcd

Along with the above we need to install Kubeproxy which also runs as a Pod.

## Worker Node

Each of the worker nodes also should run Kubeproxy.

Now, what are the main things required to deploy these Pods. 

- Kubernetes manifest files for each of the apps
- Security setup to prevent unauthorized access to the apps and securing the communication between the apps

How do we setup the apps in the master node? Because we said all the control plane apps run as Pods. Without the control plane in place how are we going to deploy and run these Pods? 

To solve this problem, we have something called Static Pods.

## Static Pods

These are pods managed directly by the kubelet daemon. So, the request for deploying a pod does not go through the API server in the control plane which is the case for any regular pod.

How does kubelet daemon do this?

Kubelet looks for any manifests available in a specific location on the Node where it is running.

`/etc/kubernetes/manifests
`
If the kubelet finds a Pod manifest in that location, it schedules a Pod directly on the node without having to go through the API server on the control plane running on the master node.

Kubelet also restarts the static pods in case if the pods crash.

## Creating static Pod manifests

All static pod manifests have to be created and dropped into this location `/etc/kubernetes/manifests` on their respective nodes.

## Securing the pods

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

## Kubeadm

[Kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) is a tool created and maintained by Kubernetes. A very detailed documentation is available for this tool.

This CLI tool helps in bootstrapping the k8s cluster by creating the necessary artifacts and performing the necessary actions.

## Prerequisites

For this session, we are going to setup our k8s cluster on AWS infrastructure. It is fine if you have a preference to use your own hardware/VM or any other cloud hosted infra.

You can refer to kubeadm document [here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin)

The following are the infrastructure requirements

- Any Linux based hosts. We will be using the latest Ubuntu VM available on AWS EC2.

- Capacity of the servers - 2 CPUs and 2 GB or greater. We will choose t2.medium type as it provides 2 vCPU and 4 GiB memory.We will be creating 1 master node and 2 worker nodes and all 3 nodes will use the t2.medium type instances.

- All instances should be part of the same network. In this case all the EC2 instances will be part of a single VPC.

- Should have unique host name and MAC address. All EC2 instances will have unique hostnames.

- Required ports have to be opened. Please use the details [here ](https://kubernetes.io/docs/reference/ports-and-protocols/)to configure open the required ports.
 
- Memory swapping should be disabled. Kubelets are designed to run the pods within the memory available on the host. Keeping swap enabled may destabilise the k8s cluster.

## Let us start!

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

We will be using [containerd](https://containerd.io/) as the container runtime as this is the standard recognised and used by all the managed k8s cluster providers [EKS](https://docs.aws.amazon.com/eks/index.html), [AKS](https://azure.microsoft.com/en-in/services/kubernetes-service/) and [GKE](https://cloud.google.com/kubernetes-engine)

Please refer to [Container Runtime Interface(CRI)](https://kubernetes.io/docs/concepts/architecture/cri/) if you want to refresh the concepts around the CRI.

Ok. Let us install the container runtime.

We will be following the steps given [here](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd).

