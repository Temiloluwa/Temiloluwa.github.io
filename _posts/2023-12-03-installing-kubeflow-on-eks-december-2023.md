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

The goal of this post is to share my experience with installing kubeflow on Amazon EKS in December 2023.
EKS has to be first setup either using the AWS cli or the AWS console before installing Kubeflow

There are two approaches Kubeflow installation approaches I explored: 
  1. Using Juju by Canonical
  2. Using the Official guide by Amazon at [link](https://awslabs.github.io/kubeflow-manifests/)


## Setup of EKS Cluster

First or all, we have to create an EKS cluster. This can be achieved programmatically using the aws cli or with the was console.
Kubeflow requires a minimum of 12gb of memory and 4 Cpus per the official [documentation](https://v0-6.kubeflow.org/docs/started/k8s/overview/).
Below are the configuration options for my nodes. 
Kubeflow would create a minimum of 70 pods on the nodes. The t2.xlarge can support up to 44 pods therefore, a minimum of 2 nodes are needed.

- kubernetes version: 1.25
- minimum number of nodes: 2
- instance type: t2.xlarge (16gb RAM, 4 CPUs)
- AMI: Amazon Linux 2
- EBS: 100gb gp3

### Things to Note
1. Creating an EKS Cluster using the AWS console is a two-step process: first you create the cluster which takes about 20 minutes to complete, then you add a node group to the cluster.

2. You need a (Cluster Service Role)[https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html#create-service-role] to create a cluster and a (Node IAM role)[https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html] to create a node group.
### Installation EBS Addon

### Install eksctl, kubectl and K9s

## Installation of Kubeflow with Juju by Canonical

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
According to the official (documentation)[https://juju.is/docs/juju/model], a model is a user-defined collection of applications and all components
that support its functionality like storage, networking. 

We name our model here kubeflow but you could use any name you choose.

``` bash
juju add-k8s kubeflow   
```
### Step 5: Deploy the Charm

```bash
juju deploy kubeflow --channel 1.7/stable  --trust

```

### Configuring Dex and OIDC to Login to Kubeflow Dashboard

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
