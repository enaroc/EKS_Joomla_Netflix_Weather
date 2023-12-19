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


# Step 2: Deploy weather flask application

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

# Step 4: Additonal resources to be configured.


NOTE:
We are able to optimise our resources using a single external load balancer that can route the traffic to multiple services.
Additionally we need a domain name that will help route traffic to specific services based on defined rules as follow.
The ingress controller will be used as an entrypoint into the cluster and load balance the traffic to our applications via the service.

-- shop.application.com goes to joomla

-- weather.application.com goes to weather flask app

-- netflix.application.com goes to netflix clone

- Create a Domain name on AWS

On the AWS console use the Route 53 service and follow the instructions to  create a Domain name.

- Create Ingress 

In your EKS cluster create an ingress manifest file to deploy ingress pod


(Run this command)

    kubectl apply -f ingress.yaml

- Create pod disruption budget 
- Create horizontal pod autoscaler

# Step 5: Monitoring solution

We want to include a monitoring solution using prometheus helm chart stack to gain visibility in the cluster.

- deploy prometheus and grafana


