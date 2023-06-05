# AWS EKS Fargate Implementation Steps
# This repository contains the implementation steps for the AWS EKS Fargate

Prerequisites:

1) Kubernetes client - Kubectl must be installed. If not already installed, Go to https://kubernetes.io/docs/tasks/tools/ and install it.
2) Amazon Web Service account
3) AWS CLI configured - Configure from here (https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) 
4) An account with the required permissions to create EKS cluster


Steps to create the EKS cluster, AWS Load Balancer Controller and Kubernetes Dashboard:

1)	Create the AWS user having required permissions to deploy EKS cluster. AWS Document reference: https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html

2)	Configure AWS cli with the credentials created in the first step.
 
3)	Creation of the EKS Fargate cluster
There are two ways of creating the EKS Fargate cluster which are mentioned below.
a)	Using eksctl command:

                eksctl create cluster \
           --name cluster_name\
           --region aws_region \
           --fargate 
            This command creates a Fargate cluster with three flags:
--name:  the name used to define the name of the cluster
--region: which region to deploy the Kubernetes cluster
--fargate: used to deploy a cluster using the Fargate deployment
            Ex: eksctl create cluster --name Test --region us-west-2 --fargate 
                  Test   Name of the cluster
                  us-west-2  AWS Region
            This will create the cluster named “Test” in the us-west-2 region with the default                  fargate profile created for the default and kube-system namespaces.

b)	Using the YAML configuration file:

The following config file declares an EKS cluster with two Fargate profiles. All pods defined in the default and kube-system namespaces will run on Fargate. All pods in the dev namespace that also have the label env=dev will also run on Fargate.
 


4)	Configure AWS load balancer controller for AWS fargate with the below steps:

a)	Allow the cluster to use AWS Identity and Access Management (IAM) for service accounts by running the following command: 

eksctl utils associate-iam-oidc-provider --region <aws-region> --cluster <EKS cluster name> --approve
                                           Ex: aws-region => us-west-2
                                                  EKS cluster => test
                    
b)	To create a service account named aws-load-balancer-controller in the kube-system namespace for the AWS Load Balancer Controller, run the following command: 
      eksctl create iamserviceaccount \
      --cluster=YOUR_CLUSTER_NAME \
      --namespace=kube-system \
      --name=aws-load-balancer-controller \
      --attach-policy-    arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
      --override-existing-serviceaccounts --approve                   
c)	To download an IAM policy that allows the AWS Load Balancer Controller to make calls to AWS APIs on your behalf, run the following command:
                            curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.4/docs/install/iam_policy.json

d)	To create an IAM policy using the policy that you downloaded in step c, run the following command:
                              aws iam create-policy \
                             --policy-name AWSLoadBalancerControllerIAMPolicy \
                            --policy-document file://iam_policy.json
               
e)	Install the AWS Load Balancer Controller using Helm
            To add the Amazon EKS chart repo to Helm, run the following command:
                  helm repo add eks https://aws.github.io/eks-charts

            To install the TargetGroupBinding custom resource definitions (CRDs), run the following command:
                      kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"

             To install the Helm chart, run the following command:
                    helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
                   --set clusterName=YOUR_CLUSTER_NAME \
                   --set serviceAccount.create=false \
                   --set region=YOUR_REGION_CODE \
                   --set vpcId=<VPC_ID> \
                   --set serviceAccount.name=aws-load-balancer-controller \
                   -n kube-system
                      

5)	Kubernetes dashboard configuration

Apply the dashboard manifest to your cluster using the command for the version of your cluster.

kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml 

Above command performs the following operations:
1)	It will create the namespace kubernetes-dashboard
2)	It will create the service account kubernetes-dashboard
3)	It will create the service kubernetes-dashboard
4)	It will create the secret required for dashboard
5)	It will create the configmaps for the dashboard
6)	It will create the deployment kubernetes-dashboard


Create the cluster role binding with cluster-admin role for the above service account created.

kubectl create clusterrolebinding kubernetes-dashboard-admin-sa --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:kubernetes-dashboard

