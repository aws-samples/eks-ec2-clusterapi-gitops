# eks-ec2-clusterapi-gitops

Cluster API Provider for aws (https://github.com/kubernetes-sigs/cluster-api-provider-aws) (CAPA), maintained by an opensource community, (https://github.com/kubernetes-sigs/cluster-api-provider-aws) is a SIG Cluster Lifecycle project, with which you can deploy and upgrade Kubernetes clusters by handling infrastructure provisioning in AWS. CAPA enables provisioning of EC2 and EKS based Kubernetes clusters. In this project, we will cover both to simulate a multi-cluster scenario where customer needs to operate Kubernetes cluster either running on a self-managed cluster backed by EC2 or the other on Amazon EKS.

# Architecture
![Diagram](/Architecture_Diagram.jpg)

# Prerequisits
* An understanding of Amazon EKS, Argo CD and Kubernetes
* Complete installation of kubectl, kind, clusterctl, clusterawsadm, argocd cli, aws cli, jq, docker
* An AWS account and a profile with administrator permissions should be configured

You may refer to the following links for installing the necessary CLI toolings: 

* kubectl - https://kubernetes.io/docs/tasks/tools/
* Kind - https://kind.sigs.k8s.io/docs/user/quick-start/
* clusterctl - https://cluster-api.sigs.k8s.io/user/quick-start.html
* clusterawsadm - https://cluster-api.sigs.k8s.io/user/quick-start.html
* argocd cli - https://argo-cd.readthedocs.io/en/stable/cli_installation/ 
* aws cli - https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
* jq - https://stedolan.github.io/jq/download/
* docker - https://docs.docker.com/engine/install/ubuntu/

# Fork the Git repositry and clone it to your local workstation 
## Step 1: Fork the git repo

Fork this Git repository to your own Github account, clone to your local workstation, then change to the following directory 

```
cd ./eks-ec2-clusterapi-gitops
```

## Step 2: Initialize the management cluster
Kind is used for creating a local Kubernetes cluster for the creation of a temporary bootstrap cluster used to provision a target management cluster on the Cluster API provider for AWS. 

Now, let’s create the kind cluster and verify the success of the creation:
```
kind create cluster
kubectl cluster-info --context kind-kind
```
You should be able to see the following when the Kind cluster is created successfully.
```
CoreDNS is running at https://127.0.0.1:60624/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Kubernetes control plane is running at https://127.0.0.1:60624 (https://127.0.0.1:60624/)
```

The Kind Kubernetes cluster will be transformed into a management cluster by installing the Cluster API provider components, it’s recommended to be separated from any application workload. 

Cluster API Provider for AWS ships with clusterawsadm, a utility to help you manage IAM objects for this project. The clusterawsadm binary uses your environment variables and encodes them in a value to be stored in a Kubernetes Secret of the Kind cluster in order to fetch necessary permissions to create the workload clusters. 
```
clusterawsadm bootstrap iam create-cloudformation-stack --region us-east-1
```

Now, let predefine all necessary environment parameters:
```
export AWS_B64ENCODED_CREDENTIALS=$(clusterawsadm bootstrap credentials encode-as-profile)
export AWS_REGION=us-east-1
```

Since we will use Cluster API to help us launch an EKS cluster too, we need to enable the following necessary feature gates: 
```
export EKS=true
export EXP_MACHINE_POOL=true (For using managed node group, this should be enabled)
export CAPA_EKS_IAM=true
```

To install the Cluster API components for AWS, the kubeadm boostrap provider, and the kubeadm control-plane provider, and transform the Kind cluster into a management cluster, we use the clusterctl init command. 
```
clusterctl init --infrastructure aws
```

The output is similar to this and if you see this, it means that you have successfully initialized the Kind cluster into a management cluster.
```
Your management cluster has been initialized successfully!
Fetching providers
Installing cert-manager Version="v1.9.1"
Installing Provider="cluster-api" Version="v1.2.2" TargetNamespace="capi-system"
...
Installing Provider="infrastructure-aws" Version="v1.5.0" TargetNamespace="capa-system"
...
```

## Step 3: Generate the Kubernetes Cluster Manifests for EKS and EC2 respectively
In this step, we are going to provision one Kubernetes cluster on EC2 and one Amazon EKS cluster, in us-east-1 and ap-southeast-2 region respectively. As we need to provision the workload clusters via Argo CD, the first step is to prepare the manifest files for each cluster respectively. One quick way to do this is to use clusterctl generate cluster and the command line will capture the environment variables you defined to formulate the manifests. 

Below is a list of environment variables you should define: 
```
export AWS_CONTROL_PLANE_MACHINE_TYPE=t3.large
export AWS_NODE_MACHINE_TYPE=t3.large
export AWS_SSH_KEY_NAME=capi-ec2 (Given we are deploying the EC2 cluster and EKS cluster in different regions, this should be adjusted accordingly )
export AWS_REGION=us-east-1 (This should be adjusted based on where you would like to create your clusters)
```

Since we haven’t created the SSH key for remote accessing the worker nodes of the clusters, let’s create them respectively: 
```
aws ec2 create-key-pair --key-name capi-eks --region ap-southeast-2 --query 'KeyMaterial' --output text > capi-eks.pem
aws ec2 create-key-pair --key-name capi-ec2 --region us-east-1 --query 'KeyMaterial' --output text > capi-ec2.pem
```

Now, we can create our workload cluster manifests by running the following command: (If you cloned the Git repository, the following manifests should exist under the capi-cluster folder.


For K8S cluster running on EC2, if you would like to generate a new cluster template instead of using the one provided, run the following command:* 
```
clusterctl generate cluster capi-ec2 --kubernetes-version v1.24.0 --control-plane-machine-count=3 --worker-machine-count=3 > ./capi-cluster/aws-ec2/aws-ec2.yaml
```

For EKS, if you would like to generate a new cluster template instead of using the one provided, run the following command: (the eks-managedmachinepool dictates that managed node group should be used when creating the work nodes for EKS cluster):
```
clusterctl generate cluster capi-eks --flavor eks-managedmachinepool --kubernetes-version v1.22.6 --worker-machine-count=2 > ./capi-cluster/aws-eks/capi-eks.yaml
export AWS_SSH_KEY_NAME=capi-eks
export AWS_REGION=ap-southeast-2
```

## Step 4: Install Argo CD on the management Kubernetes cluster
According to Cluster API documentation, one way to deploy a new Amazon EKS/K8S cluster is to run “kubectl apply” to apply the generated manifests into the ‘Kind’ management cluster. However, we will use Argo CD to streamline the workload cluster deployment based on the generated manifest stored in Git repository. 

First, let’s install Argo CD in the Kind management cluster: 
```
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait till Argo CD’s server is up, then get the login password Argo CD Web UI (For production environment, we suggest using Single-Sign-On integrating with your existing OIDC identity provider: https://argo-cd.readthedocs.io/en/stable/operator-manual/security/): 
```
kubectl get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Now, let’s login Argo CD’s web portal by using port-forwarding:
```
kubectl port-forward svc/argocd-server 8080:80
```

## Step 5: Create Application in Argo CD to deploy a new EKS cluster and K8S cluster that runs on EC2
Argo CD provides an Application CRD that maps the configuration code of a given git repository to a Kubernetes namespace. Before we deploy the Argo CD application in the management cluster, we first need to ensure that Argo CD can be authenticated to Git. To do this, log in Argo CD’s web UI, navigate to the “Setting” gear on menu bar on the left hand side, then choose “CONNECT REPO USING HTTPS”. Fill in the proper settings to connect to your git repo.

For Argo CD to be authorized to manage Cluster API objects, we need to create a ClusterRoleBinding in order to associate necessary permissions to the service account assumed by Argo CD. For simplicity, let’s add the cluster-admin role to the argocd-application-controller ServiceAccount used by Argo CD.

```
kubectl apply -f ./management/argo-cluster-role-binding.yaml
```

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-argocd-contoller
subjects:
  - kind: ServiceAccount
    name: argocd-application-controller
    namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

To create a new EKS cluster or a K8S cluster that runs on EC2, we can first define an application in a declarative approach, as a representation of a collection of Kubernetes manifests that makes up all the pieces to deploy the new cluster. In this guide, all of the configuration of the Argo CD application to be added is stored in “Management” folder of the cloned repository. For example, below is the application manifest file for creating a new K8S cluster that runs on EC2: 
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ec2-cluster
spec:
  destination:
    name: ''
    namespace: 'default'
    server: 'https://kubernetes.default.svc'
  source:
    path: capi-cluster/aws-ec2
    repoURL: '[update the Git Repo accordingly]' #Indicate which source repo for fetching the cluster configuration
    targetRevision: HEAD
  project: default #You can give a project name here
  syncPolicy:
    automated:
      prune: true
      allowEmpty: true
```

After modifying the Argo CD application yaml file as above, commit and push the updates to the source repo: 
```
git push .
git add .
git commit -m “updates Argo CD app manifest file” 
```

Run the following command to create a new Application to create the K8S cluster that runs on EC2: 
```
kubectl apply -f ./management/argocd-ec2-app.yaml
```

Similarly, we will leverage Argo CD application to help us spin up the EKS cluster. An application shall be created  in advance and in this example. Remember to modify the corresponding yaml file to ensure “repoURL” points to your own Git repo, then commit and push the update to the source repo. 

Apply the Argo CD application to spin up the EKS cluster: 
```
kubectl apply -f ./management/argocd-eks-app.yaml
```
As you deploy the two applications in Argo CD, you should be able to see them created in Argo CD’s web UI

Now, we could list all the clusters being provsioned: 
```
$ kubectl get clusters
NAME         PHASE          AGE    VERSION
capi-ec2     Provisioning  70s 
capi-eks     Provisioning  39s
```

Check status of the clusters respectively: 
```
clusterctl describe cluster capi-eks
clusterctl describe cluster capi-ec2
```

```
└─*3 Machines...* True 6m14
NAME                                           READY  SEVERITY REASON  SINCE  
Cluster/*capi-ec2*                                True                   2m53s 
├─ClusterInfrastructure - AWSCluster/*capi-ec2*   True                   9m20s 
├─ControlPlane - KubeadmControlPlane/*capi-ec2-control-plane* True       2m53s 
│ └─*3 Machines...*                               True                   6m2s   
└─Workers  
 └─MachineDeployment/*capi-ec2-md-0*              False  Warning          12m  
```

For the cluster running on EC2, you should notice that “READY” status of worker nodes remain “False”, since there’s no CNI in place in the cluster, the worker nodes haven’t joined to the cluster successfully. While you could automate the CNI deployment process via Argo CD , now let’s deploy the Calico CNI into the EC2 cluster for demonstration purpose.

Fetch the kubeconfig file for the newly created cluster on EC2:*
```
clusterctl get kubeconfig capi-ec2 > capikubeconfig-ec2
```

Deploy Calico CNI
```
kubectl --kubeconfig=./capikubeconfig-ec2 apply -f https://docs.projectcalico.org/v3.21/manifests/calico.yaml
```

As Calico CNI is successfully deployed, check the cluster status again
```
└─*3 Machines...*                                True                   16m
NAME                                          READY SEVERITY REASON SINCE  
Cluster/*capi-ec2*                              True                   13m 
├─ClusterInfrastructure - AWSCluster/*capi-ec2* True                   20m 
├─ControlPlane - KubeadmControlPlane/*capi-ec2-control-plane* True     13m 
│ └─*3 Machines...*                             True                   16m  
└─Workers 
 └─MachineDeployment/*capi-ec2-md-0*             True                   1s 
```

Navigate to the EC2 service in AWS Management Console and select “us-east-1” region; you can see that all the control plane and worker nodes have spun up running.

Let’s also verify the EKS cluster has been deployed successfully: 
```
└─ControlPlane-AWSManagedControlPlane/*capi-eks-control-plane* True                  3m22s  
clusterctl describe cluster capi-eks
NAME                                                        READY SEVERITY REASON SINCE MESSAGE
Cluster/*capi-eks*                                             True                  3m22s 
```

When creating an EKS cluster there are kubeconfigs generated and stored as secrets in the management cluster. This is different to when you create a non-managed cluster using the AWS provider. The name of the secret that contains the kubeconfig will be [cluster-name]-user-kubeconfig where you need to replace *[cluster-name]* with the name of your cluster. 

To get the user kubeconfig for a cluster named ‘capi-eks’ you can run a command similar to:
```
kubectl --namespace=default get secret capi-eks-user-kubeconfig -o jsonpath={.data.value} | base64 --decode > capikubeconfig-eks
```

List all available worker nodes of the newly created EKS cluster: 
```
ip-10-0-247-161.ap-southeast-2.compute.internal  Ready <none>  42m  v1.22.6-eks-7d68063
kubectl --kubeconfig=./capikubeconfig-eks get no
NAME                                            STATUS  ROLES AGE  VERSION
ip-10-0-191-240.ap-southeast-2.compute.internal  Ready <none>  42m  v1.22.6-eks-7d68063
```

## Step6: Deploy a sample application into the newly created clusters
For us to be able to deploy a sample application into the workload clusters via Argo CD, we need to add the EC2/EKS cluster to Argo CD as a managed cluster. For demonstration purpose, I will only show the steps for deploying a sample Nginx application onto the EKS cluster; the steps for deploying to EC2 cluster should be quite similar. In order to do so, we first need to enrol the EKS cluster as a managed cluster in Argo CD.

Login ArgoCD instance (with port forwarding enabled), we are going to use argocd CLI to authenticate against Argo CD server:
argocd login localhost:8080 (when asked if to proceed with insecure connection, say “yes)

Get the context name of the newly created EKS cluster

```
kubectl config get-contexts --kubeconfig=./capikubeconfig-eks

CURRENT    NAME       CLUSTER        AUTHINFO 
*        default_capi-eks-control-plane-user@default_capi-eks-control-plane  default_capi-eks-control-plane  default_capi-eks-control-plane-user 
```

Run the following command to add the newly created EKS cluster as a managed cluster in Argo CD
```
argocd cluster add default_capi-eks-control-plane-user@default_capi-eks-control-plane --server localhost:8080 --insecure --kubeconfig capikubeconfig-eks
```

Say ‘y’ with the following warning:
```
WARNING: This will create a service account `argocd-manager` on the cluster referenced by context `default_capi-eks-control-plane-user@default_capi-eks-control-plane` with full cluster level admin privileges. Do you want to continue [y/N]?
```

Now, let’s create a new Argo CD application to deploy the sample application to the EKS cluster. We need to substitute the EKS cluster’s API endpoint in the following Argo CD application yaml file. To discover the URL of the API endpoint, run the following command: 
```
aws eks describe-cluster --name default_capi-eks-control-plane --region ap-southeast-2 | jq '.cluster.endpoint'
```

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: eks-nginx
spec:
  destination:
    name: ''
    namespace: ''
    server: '[Replace with your EKS API Endpoint URL]' #Replace the API endpint URL of your EKS cluster
  source:
    path: workload/aws-eks
    repoURL: '[Replace with your own Git Repo URL]' #Indicate which source repo for fetching the cluster configuration
    targetRevision: HEAD
  project: default #You can give a project name here
  syncPolicy:
    automated:
      prune: true
      allowEmpty: true
```

Next, we should create a new Application in Argo CD to allow it to streamline the sample application deployment to the EKS cluster. Before applying the following command, modify the manifest file above to ensure “repoURL” points to your Git repo, then commit and push the updates. 

Apply the following command to deploy the sample nginx application:
```
kubectl apply -f ./app/eks-app/eks-nginx-deploy.yaml 
```

In Argo CD’s Web portal, you can see that the sample nginx app has been deployed into the EKS cluster successfully.


# Cleanup
To make sure there’s no unwanted cloud cost, please clean up the environment. 
- Delete the Argo CD applications: 
```
kubectl delete -f ./management/argocd-ec2-app.yaml 
kubectl delete -f ./management/argocd-eks-app.yaml
kubectl delete -f ./app/eks-app/eks-nginx-deploy.yaml
```

- Delete the created clusters: 
```
kubectl delete cluster capi-eks
kubectl delete cluster capi-ec2
```

- Delete the created EC2 SSH key in each region:
```
aws ec2 delete-key-pair --key-name capi-eks --region ap-southeast-2
aws ec2 delete-key-pair --key-name capi-ec2 --region us-east-1
```

- Delete the CloudFormation Stack when running “clusterawsadm boostrap” command to create all necessary IAM resources
```
aws cloudformation delete-stack --stack-name cluster-api-provider-aws-sigs-k8s-io --region us-east-1
```

- Delete management cluster
```
kind delete cluster - 
```

# Security 
See [CONTRIBUTING](https://gitlab.aws.dev/bkkwong/k8s-clusterapi-argocd/-/blob/main/CONTRIBUTING.md) for more information.

# License
This library is licensed under the MIT-0 License. See the LICENSE file.
