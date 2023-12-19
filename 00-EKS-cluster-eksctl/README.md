# STEP-00: PRE-REQUISITES: INSTALL CLI'S

- AWS account 
- Install and configure AWS CLI
- Install kubectl
- Install eksctl

# STEP-01: CREATE EKS CLUSTER

#Create Cluster

It will take 15 to 20 minutes to create the Cluster Control Plane 

    eksctl create cluster --name=jms-cluster \
                          --region=eu-west-2 \
                          --zones=eu-west-2a,eu-west-2b \
                          --without-nodegroup 

#Get list of nodes

  kubectl get node

#Get list of clusters

  eksctl get cluster 

#Replace with region & cluster name

    eksctl utils associate-iam-oidc-provider \
        --region eu-west-2 \
        --cluster jms-cluster \
        --approve

# STEP-02: CREATE PUBLIC NODE GROUP

#We create a key pair so whenever we create the node group it create worker node with ec2 instance if we want to access the instance we need to create a key pair and associate it

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

# STEP-03: DELETE NODE GROUP

-- List EKS Clusters

    eksctl get clusters

-- Capture Node Group name

    eksctl get nodegroup --cluster=<clusterName>

-- Delete Node Group

    eksctl delete nodegroup --cluster=<clusterName> --name=<nodegroupName>

# STEP-04: DELETE CLUSTER

-- Delete Cluster

    eksctl delete cluster <clusterName>


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

# STEP-05: Create EBS CSI DRIVER 

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

#Get Worker node IAM Role ARN

    kubectl -n kube-system describe configmap aws-auth

#from output check rolearn

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
