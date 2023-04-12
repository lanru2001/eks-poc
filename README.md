# Synopsis

The purpose of this document is to describe the proof of concept (POC) created for the DLFrame migration to AWS EKS. The POC was created as a guide in setting up and migrating DLFrame services from AWS EC2 hosting to AWS EKS.

This POC deploys a containerized Python application (Quote of the day) into an EKS cluster. If deployed successfully the quote of the day should appear in https://<hostname>/qod. Deployment instructions (cloud and local) and code can be found in the repo below. 

Repository

 

Cloud architecture - AWS EKS

![image](https://user-images.githubusercontent.com/59709429/231320828-63445dc0-b38e-4c4f-9cc4-917fcfc1d851.png)




Networking

The cloud setup uses one VPC and subnets in multiple availability zones for fault tolerance.

Private subnets (x2) - internal network used by the EC2 based nodes. This network cannot be accessed directly from the outside world.

Public subnets (x2) - used to allow ingress traffic from the internet into the network and Kubernetes cluster via application load balancer / ingress resource.

NAT gateway (x1) - used to give resources in the private subnet access to the internet for updates. This is used by the pods running in the EC2 instances to pull data as required from the internet.

Application load balancer - allows traffic from the internet into the Kubernetes cluster 

EKS 

The EKS service controls the is used to run Kubernetes on AWS without needing to install, operate, and maintain a Kubernetes control plane or nodes.

EKS nodes

The EKS cluster comprises of two or more EC2 based nodes. The nodes house the pods which house the containers. 

The nodes are part of a cluster autoscaler / autoscaling group, which increases the size of the nodes based on the number of desired pods.

ECR

AWS elastic container registry (ECR) is used to host custom Docker images (with application code) used in running docker containers in the Kubernetes pods.

Kubernetes provisioned cloud resources

The resources are provisioned and deployed into the Kubernetes cluster via Terraform using the Helm provider. 

Cluster auto scaler

Cluster Autoscaler is a tool that automatically adjusts the size of the Kubernetes cluster when one of the following conditions is true:

there are pods that failed to run in the cluster due to insufficient resources.

there are nodes in the cluster that have been underutilized for an extended period of time and their pods can be placed on other existing nodes.

Horizontal pod scaler

HorizontalPodAutoscaler automatically updates a workload resource (such as a Deployment or StatefulSet), with the aim of automatically scaling the workload to match demand. Horizontal scaling means that the response to increased load is to deploy more Pods. This also works in conjunction with the cluster auto scaler ie as pods are scaled more node will be provided by the cluster auto scaler to meed the resource requirements.

AWS load balancer controller

The AWS Load Balancer Controller (ingress controller) manages AWS Elastic Load Balancers for a Kubernetes cluster. The controller provisions the following resources.

An AWS Application Load Balancer (ALB) when you create a Kubernetes Ingress.

An AWS Network Load Balancer (NLB) when you create a Kubernetes service of type LoadBalancer. In the past, the Kubernetes network load balancer was used for instance targets, but the AWS Load balancer Controller was used for IP targets. With the AWS Load Balancer Controller version 2.3.0 or later, you can create NLBs using either target type. For more information about NLB target types, see Target type in the User Guide for Network Load Balancers.

External DNS

ExternalDNS makes Kubernetes resources discoverable via public DNS servers. It retrieves a list of resources (Services, Ingresses, etc.) from the Kubernetes API to determine a desired list of DNS records. It updates the CNAME entry for the hostname to point to the DNS address of the load balancer provisioned by the ingress resource. 

In a broader sense, ExternalDNS allows you to control DNS records dynamically via Kubernetes resources in a DNS provider-agnostic way.

Prometheus and Grafana

Monitors Kubernetes cluster using Prometheus. Shows overall cluster CPU / Memory / Filesystem usage as well as individual pod, containers, systemd services statistics. 

Kubernetes cluster

The following are resources created from Kubernetes manifest files. These resources are deployed using four Kubernetes manifest files:

app-manifest.yml - service & deployment for app

redis-manifest.yml - service & deployment for redis

config-map.yml - configuration for app pods

ingress.yml - ingress resources for app-service

Resource types in the POC cluster

Pods & containers

A Pod is a group of one or more containers, with shared storage and network resources, and a specification for how to run the containers. 

The POC cluster uses two different types of pods. One each for the App and Redis containers. The container ports used are:

App - 8080

Redis - 6379

Deployment

A Kubernetes Deployment is used to tell Kubernetes how to create or modify instances of the pods that hold a containerized application. Deployments can scale the number of replica pods, enable rollout of updated code in a controlled manner, or roll back to an earlier deployment version if necessary.

There are two deployments in the POC cluster. 

app-deployment - running x1 replica of the app pod

redis-deployment - running x1 replica of the redis pod

The deployment uses meta data of the replica sets and pods to select and control the pods.

Config map

A ConfigMap is an API object used to store non-confidential data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume.

There is one config map in the POC cluster - app-config This config map is used to store the configuration data for the app pods.

Services

A service is responsible for enabling network access to a set of pods. It creates a host name and load balancer to allow traffic to the respective pods selected using their meta data (labels). 

There are two services in the POC cluster:

app-service (NodePort type) - allows external traffic from the ports of the nodes into the app pod(s). NodPort type services are used for services that would be exposed to traffic from the internet e.g. a front end application

redis service (Cluster IP type) - allows traffic into the redis pod from the app pod(s). Cluster IP type services are usually internal services e.g. a backend application

Ingress

The ingress resource does the following:

Provisions an application load balancer, listener (port 80 for app-service) and target group (consisting of EC2 nodes in port 30005) in AWS to allow traffic into the app-service. This also done using the AWS load balancer controller (ingress controller).

Creates a DNS hostname and entry for the load balancer DNS name in Route53 to allow external users (outside the AWS network) access the application. This uses the External DNS.

Routes external traffic from the internet into the app-service using the load balancer and target group.



Deployment

Deployment to the cloud or local is performed using the tools below. Deployment instructions can be found in the repo for the POC.

Cloud

Terraform

Used to provision and deploy AWS cloud resources and Kubernetes provision cloud resources (via Helm charts)

Docker

Used to build and push custom Docker images to AWS ECR

Kubernetes

Kubernetes CLI used to connect to cluster and provision Kubernetes resources using manifest yaml files

Local

Docker-compose

Docker Compose is used for running multiple containers as a single service. Each of the containers here run in isolation but can interact with each other when required. Docker Compose files are very easy to write in a scripting language called YAML, which is an XML-based language that stands for Yet Another Markup Language. Another great thing about Docker Compose is that users can activate all the services (containers) using a single command.
 

 

