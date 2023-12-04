---
title: Installing kubeflow on Amazon EKS (December 2023)
date: 2023-12-03 13:28:43 +0100
categories: [MLOPS]
tags: [deployment]
img_path: /assets/img/blog-headers
image:
  path: kubeflow-eks.png
  alt: Installing kubeflow on Amazon EKS (December 2023)
comments: true
---

## Introduction

The primary objective of this post is to recount my experience installing Kubeflow on Amazon EKS in December 2023. Before diving into the Kubeflow installation process, it is imperative to first set up EKS either through the AWS CLI or the AWS console. 

There are two Kubeflow installation approaches I explored: 
  1. Using Juju by Canonical
  2. Using the Official guide by Amazon at [link](https://awslabs.github.io/kubeflow-manifests/)

## Setting Up Your EKS Cluster

According to the official [documentation](https://v0-6.kubeflow.org/docs/started/k8s/overview/), Kubeflow mandates a minimum of 12GB of memory and 4 CPUs. Here are the configuration options for the nodes:

- Kubernetes version: 1.25
- Minimum number of nodes: 2
- Instance type: t2.xlarge (16GB RAM, 4 CPUs)
- AMI: Amazon Linux 2
- EBS: 100GB gp3

Kubeflow, upon deployment, will generate a minimum of 70 pods on these nodes. Considering that the t2.xlarge instance type can support up to 44 pods, a minimum of 2 nodes becomes a requirement.

#### Important EKS Configuration Details
1. **EKS Cluster Creation**:
Creating an EKS Cluster using the AWS console involves a two-step process. First, you create the cluster, which takes approximately 20 minutes to complete. Subsequently, you add a node group to the cluster.

2. **Roles Requirement**:
To accomplish these tasks, you'll need [Cluster Service Role](https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html#create-service-role) for cluster creation and a [Node IAM role](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html) for node group creation. Ensure these roles are configured correctly to facilitate the setup process

#### Installation Amazon EBS CSI driver Addon

EKS comes with four [Addons](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html) installed by default but Amazon EBS CSI driver has to be installed for [dynammic provisioning of persistent volumes](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/). 
Kubeflow has components like a Mysql database that would require persistent volumes to be provisioned. 
To install the Amazon EBS CSI driver:

1. Create an IAM role according to the [guide](https://docs.aws.amazon.com/eks/latest/userguide/csi-iam-role.html)
2. Install the Addon on the Nodegroup using the [AWS Console or AWSCLI](https://docs.aws.amazon.com/eks/latest/userguide/managing-ebs-csi.html#adding-ebs-csi-eks-add-on)

#### Installation of Eksctl, Kubectl and K9s
Eksctl, Kubectl and K9s are command line tools for interacting with your EKS cluster.
[Eksctl](https://eksctl.io/) is the offical cli tool by AWS but I personally prefer using Kubectl and K9s.
K9s provides a GUI-like interface in the command line.
To permit Kubectl and K9s to detect the presence of your EKS cluster, the kubeconfig file must be updated with the cluster configuration.
Run the following awscli command to achieve this:

``` bash
# update kubeconfig file with cluster configuration
aws eks update-kubeconfig --region <region-code> --name <my-cluster>
```

## Installation of Kubeflow with Juju by Canonical
**An EKS Cluster must have been setup before you proceed** <br>
**Dependencies for this method**: Juju

If you've worked with IAC tools like Terraform or Cloudformation, then understanding how Juju works is straightforward.
Juju is an open source orchestration engine for deploying and configuring applications on on-premise and Cloud environments.
Juju charms are artifacts that encapsulate the deployment details of applications for example, wordpress, mysql, and for our use case, Kubeflow.
Visit the [Charm Hub](https://charmhub.io/) to see the comprehensive list of available charms.

To install the Kubeflow charm, we need a Controller. A controller could be installed on seperated EC2 instance or a pod in our target Kubernetes cluster.
The controller as the name implies, oversees the deployment of the charm.

### Step 1: Install juju on your local machine

These are the commands to install juju on either mac or linux machine.

``` bash
  # mac
  brew install juju
  # linux
  sudo snap install juju --classic

```

### Step 2: Add your AWS Programmatic credentials 
If you don't have programmtic credentials to your aws account, run `aws configure`.
Afterward, add this credentials to your juju local setup by running the command below and following the interactive prompt.

``` bash
juju add-credential aws   
```

### Step 3: Create your Juju controller
We are creating a controller in the Kubernetes controller as pod in our cluster.
You could opt for an EC2 controller but that would create a "M type" instance by default resulting in unnecessary costs.

``` bash
# ec2 controller
juju bootstrap aws aws-controller

# eks pod controller
juju bootstrap  kubeflow kubeflow-controller 
```

### Step 4: Create a Model on the Controller
According to the official [documentation](https://juju.is/docs/juju/model), a model is a user-defined collection of applications and all components
that support its functionality like storage, networking. 

The model is named **kubeflow** but you could use any name you choose.

``` bash
juju add-k8s kubeflow   
```
### Step 5: Deploy the Charm
The kubeflow version chosen to be deployed is **1.7/stable**.
Other versions can be found on the [Charm hub](https://charmhub.io/kubeflow)

```bash
juju deploy kubeflow --channel 1.7/stable  --trust
```

### Configuring Dex and OIDC to Login to Kubeflow Dashboard
The installation progress can be monitored in the terminal while the status of the created pods is best visualized using **K9s**.
Once all the pods are ready, **Dex** and **OIDC** can be configured with the commands below to enable access to the Kubeflow Dashboard.

Juju creates two LoadBalancers upon Kubeflow deployment, one for ingress with **Istio** the other for the **Kubernetes Controller**.
Login to the dashboard with the **Istio Ingress Gateway*** pod service url or its Load Balancer url retrieved from the Amazon web console.
The default user email address is user@example.com and the default password is 12341234.
``` bash
  # set variables
  load_balancer_url=<istio-ingressgateway url>
  password=<new password>
  email=<new email>

  # configure dex
  juju config dex-auth public-url=<load_balancer_url>
  juju config oidc-gatekeeper public-url=<load_balancer_url> 
  juju config dex-auth static-password=password
  juju config dex-auth static-username=email
```
  
## Installation of Kubeflow using the Amazon Official Guide

**An EKS Cluster must have been setup before you proceed** <br>
**Dependencies for this method**: Make, Python3.8 and Kustomize


The prequisites section of the AWS Kubeflow offical documentation provides three options for creating an ubuntu environment in order to deploy Kubeflow.
Any option would work fine, but I decided to use my local machine and perform the following:

1. Clone the official [AWS labs Kubeflow manifest repository](https://awslabs.github.io/kubeflow-manifests/docs/deployment/prerequisites/)
2. Install a **python 3.8** environment using [Miniconda](https://docs.conda.io/projects/miniconda/en/latest/). The choice of python 3.8 is very crucial. Miniconda offers the ability to select a desired python version for an environment. 
3. Install [Kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/)


## Installation with Kubeflow Manifests
The cloned repository contains a **Makefile** at the root directory. Thefore, **Make** must be installed on your system in order to run its commands. I chose not execute the **make install-tools** commands as stated in the documentation as it installs some of the dependencies like python and kubectl which I had previously installed.

The documentation offers two Kubeflow deployment methods with either **Terraform** or **Manifests**. Terraform would have been installed with the **make install-tools** command but I opted for using Manifests. The manifests method invovles executing a python script that executes systematic shell commands using **kustomize** deploy Kubeflow. The script can be found at **tests/e2e/utils/kubeflow_installation.py**. 

``` yaml
# conda environment yaml file
name: kubeflow-aws
channels:
  - conda-forge
  - defaults
dependencies:
  - boto3=1.33.6
  - numpy=1.23.5
  - pandas=1.5.3
  - sagemaker=2.198.0
  - pip:
    - black==23.11.0
    - kfp==2.4.0
    - kubernetes==26.1.0
    - mysql-connector-python==8.2.0
    - pytest==7.4.3
    - pyyaml==6.0.1
    - requests==2.31.0
    - retrying==1.3.4
```

In preparation for the deployment, I installed kustomize and the python libraries in the yaml above into my python environment. 
Next, I exported my desired deployment and installation options as environmental variables and ran **make deploy-kubeflow***  

``` bash
  # set environmental variables for vanilla deployment
  export DEPLOYMENT_OPTION=vanilla 
  export INSTALLATION_OPTION=kustomize 

  # install kubeflow using the python script at tests/e2e/utils/kubeflow_installation.py
  make deploy-kubeflow
```
The installation progress can be monitored in the terminal and with K9s.
Once all the pods are ready, port forward the Istio ingress gateway pod on your local port 8085 to access the Kubeflow dashboard.
The default user email address is user@example.com and the default password is 12341234.

``` bash
kubectl port-forward svc/istio-ingressgateway -n istio-system 8085:80

```


## Comparision of both Kubeflow Deployment Methods.