---
title: Installing kubeflow on Amazon EKS (December 2023)
date: 2023-12-03 13:28:43 +0100
categories: [MLOPS]
tags: [deployment]
img_path: /assets/img/blog-headers
image:
  path: kubeflow-eks.jpg
  alt: Installing kubeflow on Amazon EKS (December 2023)
comments: true
---

# Introduction

The primary objective of this post is to share my experience installing Kubeflow on Amazon EKS in December 2023. Before diving into the Kubeflow installation process, it is imperative to set up EKS through the AWS CLI or the AWS console. 

There are two Kubeflow installation approaches I explored: 
  1. Using Juju by Canonical
  2. Use the Official guide by Amazon at [link](https://awslabs.github.io/kubeflow-manifests/)

# Setting Up Your EKS Cluster

According to the official [documentation](https://v0-6.kubeflow.org/docs/started/k8s/overview/), Kubeflow mandates a minimum of 12GB of memory and 4 CPUs. Here are the configuration options for the nodes:

- **Kubernetes version**: 1.25
- **Minimum number of nodes**: 2
- **Instance type**: t2.x-large (16GB RAM, 4 CPUs)
- **AMI**: Amazon Linux 2
- **EBS**: 100GB gp3

Kubeflow, upon deployment, will generate a minimum of 70 pods on these nodes. Considering that the t2.xlarge instance type can support up to 44 pods, a minimum of 2 nodes becomes required.

### Important EKS Configuration Details
1. **EKS Cluster Creation**:
Creating an EKS Cluster using the AWS console involves a two-step process. First, you create the cluster, which takes approximately 20 minutes to complete. Subsequently, you add a node group to the cluster.

2. **Roles Requirement**:
To accomplish these tasks, you'll need [Cluster Service Role](https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html#create-service-role) for cluster creation and a [Node IAM role](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html) for node group creation. Ensure these roles are configured correctly to facilitate the setup process

### Persistent Volumes with the  Amazon EBS CSI driver Addon

As soon as the cluster and nodes are ready, an Amazon EBS CSI driver addon has to be installed for the [dynamic provisioning of persistent volumes](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/). Kubeflow uses the addon to provision persistent volumes for pods like Mysql and Notebook servers. EKS comes with four [Addons](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html) by default and I wonder why this important addon is not one of them. 
. 
To install the Amazon EBS CSI driver:

1. Create an IAM role according to the [guide](https://docs.aws.amazon.com/eks/latest/userguide/csi-iam-role.html)
2. Install the Addon on the Nodegroup using the [AWS Console or AWSCLI](https://docs.aws.amazon.com/eks/latest/userguide/managing-ebs-csi.html#adding-ebs-csi-eks-add-on)

### Interacting with EKS with Eksctl, Kubectl, and K9s
These command line tools are recommended for interacting with your EKS cluster: Eksctl, Kubectl, and K9s.
[Eksctl](https://eksctl.io/) is the official cli tool by AWS but I prefer using Kubectl and K9s.
K9s is amazing! It provides a GUI-like interface in the command line.
To permit Kubectl and K9s to detect the presence of your EKS cluster, the kubeconfig file must be updated with the cluster configuration.
Run the following AWS CLI command to achieve this:

``` bash
# update kubeconfig file with cluster configuration
aws eks update-kubeconfig --region <region-code> --name <my-cluster>
```

# Installation of Kubeflow with Juju by Canonical
**An EKS Cluster must have been setup before you proceed** <br>
**Dependencies for this method**: Juju

Juju is an open-source orchestration engine for deploying infrastructure and configuring applications on on-premise and Cloud environments.
It is not only an IAC tool like Terraform, but also installs applications like Wordpress and Kubeflow, with a single command.
Juju charms are artifacts that encapsulate the deployment and configuration details of applications.
Visit the [Charm Hub](https://charmhub.io/) to see the comprehensive list of available charms.

To install the Kubeflow charm, we need a Controller. 
A controller in Juju oversees the deployment of a charm and it could be installed on a separate EC2 instance or a pod in our target Kubernetes cluster.
These are the steps for deploying Kubeflow with Juju


### Step 1: Install juju on your local machine

Install the Juju Cli on either a Mac or Linux machine.

``` bash
  # mac
  brew install juju
  # linux
  sudo snap install juju --classic

```

### Step 2: Add your AWS Programmatic credentials 
Add your AWS credentials to Juju. Configure programmatic access to your AWS account using `aws configure` if you lack access.

``` bash
juju add-credential aws   
```

### Step 3: Create your Juju controller
I chose to create a Kubernetes controller as a pod in the cluster.
If an EC2 controller is desired, a "M type" instance is created by default by Juju.

``` bash
# ec2 controller
juju bootstrap aws aws-controller

# eks pod controller
juju bootstrap  kubeflow kubeflow-controller 
```

### Step 4: Create a Model on the Controller
A Juju [model](https://juju.is/docs/juju/model) is a user-defined collection of applications and all components
that support its functionality like storage and networking. 

The model is named **kubeflow** but any arbitrary name can be used.

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
Once all the pods are ready, **Dex** and **OIDC** must be configured with the commands below to access the Kubeflow Dashboard.

Juju creates two LoadBalancers upon Kubeflow deployment, one for ingress with **Istio** and another for the **Kubernetes Controller**.
Login to the dashboard with the **Istio Ingress Gateway** pod service URL or its Load Balancer URL retrieved from the Amazon web console.
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
  
# Installation of Kubeflow using the Amazon Official Guide

**An EKS Cluster must have been setup before you proceed** <br>
**Dependencies for this method**: Make, Python3.8 and Kustomize

The prerequisites section of the [AWS Kubeflow official documentation](https://awslabs.github.io/kubeflow-manifests/docs/deployment/prerequisites/) offers three options for creating an Ubuntu environment to deploy Kubeflow.
Any option works fine, but I decided instead to execute the installation from my local machine (Mac or Linux).
These are the important prerequisites:

1. Clone the official [AWS labs Kubeflow manifest repository](https://awslabs.github.io/kubeflow-manifests/docs/deployment/prerequisites/)
2. Install a **Python 3.8** environment using [Miniconda](https://docs.conda.io/projects/miniconda/en/latest/). The choice of Python 3.8 is very crucial. Miniconda offers the ability to select a desired Python version for an environment. 
3. Install [Kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/)


## Installation with Kubeflow Manifests
The cloned repository contains a **Makefile** at the root directory. Therefore, **Make** must be installed on your system to run its commands. I chose not to execute the `make install-tools` command as stated in the documentation as it installs dependencies like Python and Kubectl which I had previously installed.

The documentation recommends two Kubeflow deployment methods as soon as the prerequisites are fulfilled: with either **Terraform** or **Manifests**. Terraform would have been installed with the `make install-tools` command but I opted for using Manifests. The manifests method runs a Python script that executes shell commands using Kustomize to deploy Kubeflow. The script can be found at `tests/e2e/utils/kubeflow_installation.py`. 

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

In preparation for the deployment, the Python libraries in the yaml above must be installed into Python environment. 
Finally, the deployment is implemented using `make deploy-kubeflow`

``` bash
  # set environmental variables for vanilla deployment
  export DEPLOYMENT_OPTION=vanilla 
  export INSTALLATION_OPTION=kustomize 

  # install kubeflow using the python script at tests/e2e/utils/kubeflow_installation.py
  make deploy-kubeflow
```
The installation progress can be monitored in the terminal.
Once all the pods are ready, port forward the Istio ingress gateway pod to your local port 8085 to access the Kubeflow dashboard.
The default user email address is user@example.com and the default password is 12341234.

``` bash
kubectl port-forward svc/istio-ingressgateway -n istio-system 8085:80

```

## Comparison of both Kubeflow Deployment Methods.
From a cost perspective, the official AWS installation method is recommended. There is no need for controllers like with Juju which creates an extra Load balancer or EC2 instance.
On the other hand, once Juju is set up, the installation and maintenance of Kubeflow are easier to implement.

# References

1. [Official Kubeflow on AWS Documentation](https://awslabs.github.io/kubeflow-manifests/docs/)
2. [Canonical Juju](https://juju.is/)