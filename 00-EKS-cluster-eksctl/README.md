# STEP-00: PRE-REQUISITES: INSTALL CLI'S

- AWS account 
- Install and configure AWS CLI
- Install kubectl
- Install eksctl
- Install helm cli

# STEP-01: CREATE EKS CLUSTER

-- Create Cluster

It will take 15 to 20 minutes to create the Cluster Control Plane 

    eksctl create cluster --name=jms-cluster \
                          --region=eu-west-2 \
                          --zones=eu-west-2a,eu-west-2b \
                          --without-nodegroup 
    #It will create a VPC with 2 AZ's and without nodegroup as we will specify our rndoegroup requirements later on

-- Get list of nodes

    kubectl get node

-- Get list of clusters

    eksctl get cluster 

-- Replace with region & cluster name

      eksctl utils associate-iam-oidc-provider \
          --region eu-west-2 \
          --cluster jms-cluster \
          --approve

# STEP-02: CREATE NODE GROUP

- Public Node Group

-- We create a key pair so whenever we create the node group it create worker node with ec2 instance if we want to access the instance we need to create a key pair and associate it


    eksctl create nodegroup --cluster=jms-cluster \
                            --region=eu-west-2 \
                            --name=jms-cluster-ng-public3 \
                            --node-type=t2.medium\
                            --nodes=2 \
                            --nodes-min=2 \
                            --nodes-max=4 \
                            --node-volume-size=20 \
                            --ssh-access \
                            --ssh-public-key=eks-key\
                            --managed \
                            --asg-access \
                            --external-dns-access \
                            --full-ecr-access \
                            --appmesh-access \
                            --alb-ingress-access 

- Private Node Group

-- If we don't need to run any workload in the public subnet nodes we can delete the public node group using instructions in Step-03. The kubelet will communicate with the control plane via the NAT Gateway. 

We then add the following option to the command --node-private-networking to create our public node group

    eksctl create nodegroup --cluster=jms-cluster \
                            --region=eu-west-2 \
                            --name=jms-cluster-ng-private1 \
                            --node-type=t2.medium\
                            --nodes=2 \
                            --nodes-min=2 \
                            --nodes-max=4 \
                            --node-volume-size=20 \
                            --ssh-access \
                            --ssh-public-key=eks-key\
                            --managed \
                            --asg-access \
                            --external-dns-access \
                            --full-ecr-access \
                            --appmesh-access \
                            --alb-ingress-access \
                            --node-private-networking


-- To verify EKS Node groups

     ekstcl get node groupe --cluster=<clustername>

-- To verify that your node group has been created in the private subnet run the below command and look for EXTERNAL-IP : <none>

     kubectl get nodes -o wide

-- To verify if any IAM Service Accounts present in EKS Cluster

    eksctl get iamserviceaccount --cluster=jms-cluster

    Observation: 
    1. No iamserviceaccounts found

When we create an EKS cluster automatically a kube config will be created located in 

    $ cat $HOME/.kube/config


# STEP-03: Create EBS CSI DRIVER 

In our project we will use MySQL db with persistent storage. We create a storage class to dynamically provision a volume. We need to create an IAM policy and attach it to the IAM role worker node and deploy Amazon EBS CSI driver in our cluster

1) Create IAM policy 

- Go to services -> IAM -> Create a Policy:

Select JSON tab and copy paste the below JSON

    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "ec2:AttachVolume",
            "ec2:CreateSnapshot",
            "ec2:CreateTags",
            "ec2:CreateVolume",
            "ec2:DeleteSnapshot",
            "ec2:DeleteTags",
            "ec2:DeleteVolume",
            "ec2:DescribeInstances",
            "ec2:DescribeSnapshots",
            "ec2:DescribeTags",
            "ec2:DescribeVolumes",
            "ec2:DetachVolume"
          ],
          "Resource": "*"
        }
      ]
    }

- Review the same in Visual Editor
- Click on Review Policy
- Name: Amazon_EBS_CSI_Driver
- Description: Policy for EC2 instances to access Elastic Block Store
- Click on Create Policy

2) Get the IAM role Worker Nodes using and Associate this policy to that role

-- Get Worker node IAM Role ARN

    kubectl -n kube-system describe configmap aws-auth

-- from output check rolearn

rolearn: arn:aws:iam::180789647333:role/eksctl-eksdemo1-nodegroup-eksdemo-NodeInstanceRole-IJN07ZKXAWNN

- Go to Services -> IAM -> Roles
- Search for role with name eksctl-clustername-nodegroup and open it
- Click on Permissions tab
- Click on Attach Policies
- Search for Amazon_EBS_CSI_Driver and click on Attach Policy

3) Deploy Amazon EBS CSI Driver

-- Verify kubectl version, it should be 1.14 or later

    kubectl version --client --short

-- Deploy EBS CSI Driver

    kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"

-- Verify ebs-csi pods running

    kubectl get pods -n kube-system


#  STEP-04: Create an External dns policy

-- Create IAM Policy
-- Name: AllowExternalDNSUpdates
-- Description: Allow access to Route53 Resources for ExternalDNS
-- Make a note of the policy ARN
        {
        "Version": "2012-10-17",
        "Statement": [
            {
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets"
            ],
            "Resource": [
                "arn:aws:route53:::hostedzone/*"
            ]
            },
            {
            "Effect": "Allow",
            "Action": [
                "route53:ListHostedZones",
                "route53:ListResourceRecordSets"
            ],
            "Resource": [
                "*"
            ]
            }
        ]
        }


# STEP-05: Create IAM, k8s Service Account and Associate IAM policy

-- This step create a k8s service account and AWS IAM role and associate them by annotating the policy ARN in Service Account

        eksctl create iamserviceaccount \
            --name ext-dns \
            --namespace wfa \
            --cluster jms-cluster \
            --attach-policy-arn arn:aws:iam::142540658952:policy/AllowExternalDNSUpdates \
            --approve \
            --override-existing-serviceaccounts

-- Get the IAM Role ARN with the below command 

     eksctl get iamserviceaccount --cluster jms-cluster

-- Deploy external DNS
    kubectl apply -f kube-manifests/

-- List All resources from default Namespace
    kubectl get all

-- List pods (external-dns pod should be in running state)
    kubectl get pods

-- Verify Deployment by checking logs
    kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')


# STEP-06: DELETE NODE GROUP

-- List EKS Clusters

    eksctl get clusters

-- Capture Node Group name

    eksctl get nodegroup --cluster=<clusterName>

-- Delete Node Group

    eksctl delete nodegroup --cluster=<clusterName> --name=<nodegroupName>

    eksctl delete nodegroup --cluster=<clusterName> --name=<nodegroupName> --drain=false

# STEP-07: DELETE CLUSTER

-- Delete Cluster

    eksctl delete cluster <clusterName>

