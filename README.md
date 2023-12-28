# Cloud - Kubernetes Project

For this project we will deploy 3 applications in a kubernetes cluster (Amazon EKS). These applications should be accessible from the external load balancer and we should have a monitoring solution to gain visibility in what is happening in the cluster.

We will be deploying the following:

- Joomla image
- Netflix clone
- Weather flask app

For this project we will be required to:

- Create custom Dockerfile to containerise our netflix and weather applications
- Create docker image and upload to container registry to use in our deployments
- Create a domain name in AWS 

00-EKS = Instruction to create EKS cluster 

01-Joomla = manifest for Joomla application

02-Weather = source code + manifests for Weather Flask application

03-Django_netflix = source code + manifests for netflix application


# What's Amazon EKS ?

Amazon Elastic Kubernetes Service is a fully managet kubernetes service for managing containerized workload.. Customers can run their most sensitive and mission critical applications because of it's security, reliability and scalability.

EKS is deeply integrated with services such as Amazon CloudWatch, Auto Scaling Groups, AWS Identity and Access Management (IAM), and Amazon Virtual Private Cloud (VPC), providing you a seamless experience to monitor, scale, and load-balance your applications.

Additionally, EKS provides a scalable and highly-available control plane that runs across multiple availability zones to eliminate a single point of failure.

Use instructions in 00-EKS-cluster-eksctl  to provision your EKS cluster, Node groups,  EBS CIS Driver, External DNS, K8s Service Account

# Step 1: Deploying the Joomla image and link MySQL pod

(This creates a storage class to dynamically provision a volume)

- Create a namespace named joomla and switch context form default namespace to joomla namespace

(Run the following commands)

    kubectl create ns joomla

    kubectl config set-context --current --namespace=joomla
- Create the storage class manifest sc_joomla_mysql and sc_joomla.yaml  #This creates a storage class to dynamically provision an ebs volume for our pods.

(Run this command)

    kubectl apply -f sc_joomla_mysql.yaml

    kubectl apply -f sc_joomla.yaml

- Create the MySQL deployment mysql_joomla.yaml

(Run this command)

    kubectl apply -f mysql_joomla.yaml

    kubectl get pod

- Create deploy_joomla.yaml (Deployment, Service, PVC)

(Run this command)

    kubectl apply -f deploy_joomla.yaml

    kubectl get pod 


# Step 2: Deploy Weather Flask Application

- Create a namespace named wfa

(Run this command)

    kubectl create ns wfa

    kubectl config set-context --current --namespace=wfa

- Create deploy_weather-flask-app.yaml (Deployment, Service)

(Run this command)

    kubectl apply -f deploy_weather-flask-app.yaml

    kubectl get pod


# Step 3: Deploy the Django Netflix Clone application and link to MySQL pod


- Dowload source code for https://freeprojectscodes.com/netflix-clone-with-django-python/ 

- Create a Dockerfile to build a Docker Image and push it to Amazon ECR private registry
- Create a Namespace named netflix

(Run this command)

    kubectl create ns netflix

    kubectl config set-context --current --namespace=netflix

- Create the storage class manifest sc_mysql_netflix.yaml

(Run this command)

    kubectl apply -f manifest sc_mysql_netflix.yaml

- Create MySQL DB StatefulSet for the Netflix application mysql_netflix.yaml

(Run this command)

    kubectl apply -f mysql_netflix.yaml

- Create deploy_netflix.yaml (Deployment, Service, PVC)

(Run this command)  

    kubectl apply -f deploy_netflix.yaml

for this deployment we are pulling image from private docker hub registry therefore we need to configure the following on the CLI
        kubectl create secret docker-registry <name>
        --docker-server=<your-registry-server> 
        --docker-username=<your-name> 
        --docker-password=<your-pword>
        --docker-email=<your-email>


# Step 4: Create DNS domain, External DNS


NOTE:
We are able to optimise our resources using a single external load balancer that can route the traffic to multiple services.
Additionally we need a domain name that will help route traffic to specific services based on defined rules as follow.
The ingress controller will be used as an entrypoint into the cluster and load balance the traffic to our applications via the service.

For this project we will configure ingress name based virtual host routing :

- shop.application.com goes to joomla related backend

- weather.application.com goes to weather flask app related backend

- netflix.application.com goes to netflix clone related backend


You can also configure path based routing for the request to go to the related backend When the request goes to:

- www.application.com/shop

- www.application.com/weather

- www.application.com/netflix


What is AWS ALB ?

ALB ingress controller triggers the creation of an ALB and the necessary AWS resources whener an ingress resource is created in the cluster with:


kubernetes.io/ingress.class:alb annotation.


The ALB ingress controller supports two traffic modes
-- Instance mode:
- registers nodes within the cluster as targets for the ALB
- traffic reaching ALB is routed to NodePort for your service and proxied to pods
- it's the default traffic mode

-- IP mode:
- it registers pods as the targets for the ALB
- traffoc reaching the ALB is direclty routed to pods for your service
- it must me specified to use this traffic mode.
- can be used with Fargate 

Install AWS Load balancer controller on AWS EKS cluster which is the core controller which helps to create the ingress services in kubernetes cluster.

AWS Load balancer controller resources are going to be created in kube system namespace

AWS Load Balancer Controller Service Account in kube-system namespace + IAM Poilicy and IAM role that will be annotated in load balacner controller service account
Which will

Once we deploy AWS Load Balancer Controller it will create a deployment which will create the pods related to LB controller and in addition to that :

- AWS Load Balancer Controller Service Account
- AWS Load Balancer Controller Deployment
- AWS Load Balancer Controller WebHook ClusterIP service
- AWS Load Balancer TLS

The assocaited service account to the AWS Load balancer Controller give it authorisation to create a load balancer in our AWS account from the EKS cluster

What's happens?

The kubernetes administrator deploye the Ingress k8s manifest and the AWS load balancer controller in our EKS cluster watch for Ingress Events on  the kubernetes API server and when it finds the ingress k8s manifest deployed. It takes the manifest and create the applciation load balancer for us 

1) Create IAM Policy and make note of Policy ARN
2) Create IAM Role and k8s Service Account and bound them together
3) Install AWS Load Balancer controller using HELM CLI
4) Understand IngressClass Concept and create default ingress class

1) Create IAM policy for the AWS Load Balancer Controller that allows it to make calls to AWS APIs on your behalf.

-- Dowload the latest IAM policy
(outline the permissions)
    curl -o iam_policy_latest.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

-- Verify latest
    ls -lrta 

-- Download specific version
    curl -o iam_policy_v2.3.1.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.3.1/docs/install/iam_policy.json

-- Create IAM Policy using policy downloaded and make note of the arn: as it will be reference in upcoming tasks

    aws iam create-policy \
        --policy-name AWSLoadBalancerControllerIAMPolicy \
        --policy-document file://iam_policy_latest.json

        {
    "Policy": {
        "PolicyName": "AWSLoadBalancerControllerIAMPolicy",
        "PolicyId": "ANPASCMAV3EEDAW3S4FFA",
        "Arn": "arn:aws:iam::142540658952:policy/AWSLoadBalancerControllerIAMPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2023-12-21T11:37:38+00:00",
        "UpdateDate": "2023-12-21T11:37:38+00:00"
    }

2) Create IAM Role for the AWS LoadBalancer Controller and attach the role to the Kubernetes service account

We can use ekstl with a single command :
 - Create an AWS IAM role with eksctl
 - Create Kubernetes Service Account in k8s cluster
 - Bound IAM role created and the kubernetes service account created

-- Verify if any existing service account
    kubectl get sa -n kube-system
    kubectl get sa aws-load-balancer-controller -n kube-system
Observation:
 Nothing with name "aws-load-balancer-controller" should exist

-- Template to create sa
    eksctl create iamserviceaccount \
      --cluster=jms-cluster \
      --namespace=kube-system \
      --name=aws-load-balancer-controller \
      --attach-policy-arn=arn:aws:iam::142540658952:policy/AWSLoadBalancerControllerIAMPolicy \
      --override-existing-serviceaccounts \
      --approve

 #Note:  K8S Service Account Name that need to be bound to newly created IAM Role

-- Replaced name, cluster and policy arn (Policy arn we took note in earlier)
    eksctl create iamserviceaccount \
      --cluster=eksdemo1 \
      --namespace=kube-system \
      --name=aws-load-balancer-controller \
      --attach-policy-arn=arn:aws:iam::142540658952:policy/AWSLoadBalancerControllerIAMPolicy \
      --override-existing-serviceaccounts \
      --approve

-- Verify  IAM service account using eksctl

    eksctl  get iamserviceaccount --cluster jms-cluster

-- Verify K8S service account using eksctl

    kubectl get sa -n kube-system
    kubectl get sa aws-load-balancer-controller -n kube-system
    Observation:
    1. We should see a new Service account created. 

-- Describe Service Account aws-load-balancer-controller
    kubectl describe sa aws-load-balancer-controller -n kube-system

3) Install the AWS Load balancer controler using HELM

-- Install HELM CLI

https://helm.sh/docs/intro/install/

-- Add the eks-charts repository.

    helm repo add eks https://aws.github.io/eks-charts

-- Update your local repo to make sure that you have the most recent charts.
    helm repo update


-- Install the AWS Load Balancer Controller.

(Template)

    helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
      -n kube-system \
      --set clusterName=<cluster-name> \
      --set serviceAccount.create=false \
      --set serviceAccount.name=aws-load-balancer-controller \
      --set region=<region-code> \
      --set vpcId=<vpc-xxxxxxxx> \
      --set image.repository=<account>.dkr.ecr.<region-code>.amazonaws.com/amazon/aws-load-balancer-controller
      # The account id for each regions can be found here https://docs.aws.amazon.com/eks/latest/userguide/add-ons-images.html
      # We set the service account create to false as we already created it and reference the name we gave it in service account name

(Replace it as below)

    helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
      -n kube-system \
      --set clusterName=jms-cluster \
      --set serviceAccount.create=false \
      --set serviceAccount.name=aws-load-balancer-controller \
      --set region=eu-west-2 \
      --set vpcId=vpc-0b7a7ac7c8e23755b \
      --set image.repository=602401143452.dkr.ecr.eu-west-2.amazonaws.com/amazon/aws-load-balancer-controller

-- Verify that the controller is installed.
    kubectl -n kube-system get deployment 
    kubectl -n kube-system get deployment aws-load-balancer-controller
    kubectl -n kube-system describe deployment aws-load-balancer-controller

-- Verify AWS Load Balancer Controller Webhook service created
    kubectl -n kube-system get svc 
    kubectl -n kube-system get svc aws-load-balancer-webhook-service
    kubectl -n kube-system describe svc aws-load-balancer-webhook-service

--  Verify Labels in Service and Selector Labels in Deployment
kubectl -n kube-system get svc aws-load-balancer-webhook-service -o yaml
kubectl -n kube-system get deployment aws-load-balancer-controller -o yaml
Observation:
1. Verify "spec.selector" label in "aws-load-balancer-webhook-service"
2. Compare it with "aws-load-balancer-controller" Deployment "spec.selector.matchLabels"
3. Both values should be same which traffic coming to "aws-load-balancer-webhook-service" on port 443 will be sent to port 9443 on "aws-load-balancer-controller" deployment related pods. 


- To Uninstall AWS Load balancer controller using Helm Command

-- This step should not be implemented at this stage it is to show how to uninstall aws load balancer from EKS cluster

    helm uninstall aws-load-balancer-controller -n kube-system


NOTES:
- Note-1: Rollback any Security Group Changes

When we create a EKS cluster using eksctl it creates the worker node security group with only port 22 access.
We will be creating many NodePort Services to access and test our applications via browser.

During this process, we need to add an additional rule to this automatically created security group, allowing access to  applications we have deployed.
So the point we need to understand here is when we are deleting the cluster using eksctl, its core components should be in same state which means roll back the change we have done to security group before deleting the cluster.

In this way, cluster will get deleted without any issues, else we might have issues and we need to refer cloudformation events and manually delete few things. In short, we need to go to many places for deletions.

- Note-2: Rollback any EC2 Worker Node Instance Role - Policy changes

When we are doing EBS Storage Section with EBS CSI Driver we will add a custom policy to worker node IAM role.

When you are deleting the cluster, first roll back that change and delete it.

This way we don't face any issues during cluster deletion.


4) Create a Domain name on AWS

On the AWS console use the Route 53 service and follow the instructions to  create a Domain name.

5) Create Ingress manifest and add an External DNS Annotation with 3 DNS names

-- Add the following annotation to your ingress manifest replace it with your choser records.
    
    external-dns.alpha.kubernetes.io/hostname: weather.<domainname>.com, netflix.<domainname>.com, shop.<domainname>.com

-- Configure Host based routing in ingress deployement as explained above

    kubectl apply -f ingress.yaml

-- Verify the 3 records heve been created in your AWS account. 


-- On the browser access your application on the browser as follow:

     weather.<domainname>.com goes to weather flask app related backend

     netflix.<domainname>.com goes to netflix clone related backend

     shop.<domainname>.com goes to joomla related backend

# STEP 6:  Horizontal Pod Autoscaling (HPA)
 

- What is HPA ?

Increasing and decreasing the number of Replicas (Pods).It automatically scales the number of pods in a deployment, replica set, stateful set based on the resource's CPU Utilization or other metrics.

It Helps our applcaition to scale out to meet increased demand or scale in when resources are not needed.

When we set a target CPU utilisation percentage, the HPA scales our application in or out to try and meet that target.

The HPA needs k8s metric server installed to verify the CPU of a pod

We do not need to deploy or install the HPA on our cluster to begin scaling our applciations, it's available as a default kubernetes APU resource.

- How does it work ?

In k8s cluster When you deploy application using deployment, ReplciaSet, Replication Controller , StatefulSet which is going to span pods to deploy our applcaition container.

We deploy a default k8s metrics server available for us and the pods metrics are sent to respective metric server. When we deploy HPA for our applciation it's going to query for metrics and once it gets it it will calculate the number of replicas the autoscaler need to increase or decrease and scale the applciation to the desired replicas which will increase or decrease the number of pods in our respective application.

The process of collecting the metrics, calculating the replies and trigger scaling request is called a control loop which is executed every 15 seconds.

- How is HPA configured ?

HPA requires:
- Scaling metric Ex: CPU utilization
- Target value Ex: CPU = 50% In the metric once the CPU has reached 50% you can scale in or out based on that
- Min Replicas = 2
- Max Replicas = 10


1) Install Metrics Server

-- Verify if Metrics Server is already isntalled

    kubectl -n kube-system get deployment/metrics-server    

-- Install Metrics Server

    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.6.4/components.yaml

NOTE: To get the latest metrics server releases 

    https://github.com/kubernetes-sigs/metrics-server/releases

-- Verify that it's been installed

    kubectl get deployment metrics-server -n kube-system    

-- Deploy HPA deployment

    kubectl apply -f deploy_hpa.yaml

-- Create HPA resource for our applications

2) Create HPA resource for each applications and generate load 

-- Create individual HPAs for each application enable granular monitoring and scaling 

-- Create shared HPA for application with similar resource requirements or scaling behaviours allow resource optimization

- Using an imperative approach   

Template to run a test 

    kubectl autoscale deployment <deployment> --metric=<targetValue> --min=<minimumValue> --max=<maximumValue>
    kubectl autoscale deployment hpa-demo-deployment --cpu-percent=50 --min=1 --max=10
    kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"

NOTE: This creates an autoscaler that targets 50 percent CPUutilization with a minimum of one pod and maximum of 10 pods.

When the average CPU load is below 50% the autoscaler tries to reduce the number of pods in teh deploymnet to a minimum of 1

When the load is greater than 50% the autoscaler increases the number of pods in the deploymnet up to a maximum of ten

Template for the weather app

    kubectl autoscale deployment weatherapp --cpu-percent=50 --min=1 --max=10

    kubectl run -i --tty load-generator --rm --image=httpd -- ab -n 500000 -c 1000 http://weather-svc.wfa.svc.cluster.local/ 

    kubectl get hpa netflix-app --watch

Template for the netflix app

    kubectl autoscale deployment netflix-app --cpu-percent=50 --min=1 --max=10

    kubectl run -i --tty load-generator --rm --image=httpd -- ab -n 500000 -c 1000 http://netflix-svc.wfa.svc.cluster.local/ 

    kubectl get hpa hpa-weatherapp --watch

Template for the joomla app

    kubectl autoscale deployment myapp-joomla --cpu-percent=50 --min=1 --max=10

    kubectl run -i --tty load-generator --rm --image=httpd -- ab -n 500000 -c 1000 http://joomla-svc.wfa.svc.cluster.local/ 

    kubectl get hpa hpa-joomla-app --watch
    

https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/


- Using an declarative approach  

Create a HPA k8s manifest 

3) verify how HPA is working   

In a separate terminal observe how the following changes

    # List all HPA
    kubectl get hpa

    # List specific HPA
    kubectl get hpa hpa-demo-deployment 

    # Describe HPA
    kubectl describe hpa/hpa-demo-deployment 

    # List Pods
    kubectl get pods

# STEP 7:  Additonal resources to be configured next

- Monitoring solution with prometheus and grafana 
- Create pod disruption budget
