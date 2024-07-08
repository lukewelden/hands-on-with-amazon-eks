# Notes
I've created this page to document notes taken throughout the course. 

> Run all of the commands below in an AWS CloudShell for best results. I used ACG's Cloud Playgrounds :)  

## Introduction 
### The Bookstore Project 
The Bookstore project is an application that we'll be using throughout the course. It is made up of a  Fontend, Backend, and Data layer that allows users to manage clients, resources, and inventory of a make believe Bookstore. 
### The Bookstore Infrastructure 
The bookstore application will be deployed to EKS worker nodes (EC2 instances) over three AWS availability zones within a private subnet. There will also be a load balancer within the public subnets to direct traffic to a load balancer in the private subnets. The worker nodes will then communicate to Dynamo DB for Data persistence. 
### Creating the EKS Cluster
To create the EKS Cluster run  `eksctl create cluster -f Infrastructure/eksctl/01-initial-cluster/cluster.yaml`

> You can also run scripts-by-chapter/chapter-1.sh 

## Networking and EKS
### Private Access Recommendation 
It is best practice to keep your resources within an EKS cluster private. The following code snippet from [cluster.yaml](./Infrastructure/eksctl/01-initial-cluster/cluster.yaml) enables resources to be only private. 
```yaml
# cluster.yaml
privateCluster:
    enabled: true
```
You can setup access to the private resources by implementing a VPN connection. There's an interesting course on ACG that covers this named [Configuring OpenVPN on Amazon EC2](https://learn.acloud.guru/course/d1fa71b9-020d-44a8-affd-820a5bfa8112/overview) by Matthew Pearson. 
### Load Balancing Best Practices
Load balancers act as a front door to our applications, they help to distribute user traffic across the worker nodes. It does so by using algorithms to decide which node it will send the user request to. AWS have created an open source tool that aids load balancing within an EKS cluster named _AWS Load Balancer Controller_. 

To install the _AWS Load Balancer Controller_ into the EKS cluster follow the steps below;
1. Run the [create.sh](./Infrastructure/k8s-tooling/load-balancer-controller/create.sh). 
2. You can confirm installation by running `kubectl get pods` noticing the _aws-load-balancer-controller_ pods.
3. Browse to CloudFormation -> aws-load-balancer-iam-policy within the AWS console and copy the IAM policy name. 
4. Browse to the eksctl-eks-acg-nodegroup CloudFormation Stack and click on the `NodeInstanceRole` resource. 
5. In the new window that has been opened attach the IAM Policy to the Role. 

To test the installation; 
1. You can test that the aws controller has been configured by by running [run.sh](Infrastructure/k8s-tooling/load-balancer-controller/test/run.sh). 
2. Once the script has completed, browse to EC2 -> Load Balancers within the AWS Console and copy the DNS name. 
5. Browse to the DNS name within a browser and you should be directed to an NGINX web page. This confirms that your HTTPS request has been routed through the AWS controller and onto a pod running within the EKS cluster. 
### Secure Load Balancing for EKS 
By default the load balancers receive a name from AWS which is rather long and complex. To make it easier for our end users we can assign a readable DNS name via Route53 whilst we're doing this we can also assign an SSL certificate to the name via AWS Certificate Manager to encrypt HTTP traffic through the load balancer. To do this follow the steps below. 
1. Run [create.sh](./Infrastructure/cloudformation/ssl-certificate/create.sh). This script gets the first DNS Hosted Zone from Route53 and creates a wildcard SSL certificate for the domain using CloudFormation. 
2. To test an HTTPS connection to the application run [run-with-ssl.sh](./Infrastructure/k8s-tooling/load-balancer-controller/test/run-with-ssl.sh). This scripts grabs the domain name from Route53, updates the helm chart for the test application to allow HTTPS traffic by redirecting SSL traffic to port 443, and adds a host name of sample-app.{{base domain from Route53}}. 
3. Use Route53 to add an A record for `sample-app.{{base domain from Route53}}`, making sure to check the "Alias" button, and select the load balancer. 
Once the script has run you can browse to sample-app.{{base domain from Route53}} 
4. Finally, browse to sample-app.{{base domain from Route53}} using https!
### Automating DNS Management 
The Problem: DNS is managed manually! It is inefficient to manage DNS for your cluster manually. Each time you wish to add, remove, or update a record you'd need to go into the console and update it. Luckily for us there's a solution that automates this process [External DNS](https://github.com/kubernetes-sigs/external-dns). To setup External DNS follow the steps below. 
> Before starting the steps below be sure to delete the manually created DNS record from the last paragraph. 
1. Install External DNS onto cluster using [create.sh](./Infrastructure/k8s-tooling/external-dns/create.sh). This scripts runs a helm chart that installs External DNS onto the cluster. 
2. Check that the External DNS Pods are running on the cluster with `kubectl get pods` 
3. Update the `NodeInstanceRole` IAM Role with the `AmazonRoute53FullAccess` IAM Policy to allow the worker nodes access to Route53.
4. Delete the External DNS pods with `kubectl delete pod <podname>` to apply the changes. _I had to terminate my worker node EC2 instances for this to update. I found this out by using `kubectl logs <podname>` to identify a permission issue even after recreating the pod_
5. Once the External DNS pods are rebooted check Rout53 to see if the DNS records have automatically been created. 
### Installing the Bookstore Application
Now that the cluster is up and running and we have some networking configured its time to install the bookstore application onto the cluster. To do this follow the steps below. 
1. Deploy the DynamoDB tables for each microservice. In each of the microservices folders (`clients-api`,  `inventory-api`, `renting-api`, `resource-api`) there is a `infra/cloudformation` directory. Inside this directory run the [create-dynamodb-table.sh](./clients-api/infra/cloudformation/create-dynamodb-table.sh) with the argument `development` for example `./create-dynamodb-table.sh development`. _You need to be in the same dir as the script_. This script deploys the Cloudformation template [dynamodb-table.yaml](./clients-api/infra/cloudformation/dynamodb-table.yaml) with the namespace `development`. 
2. Update the `NodeInstanceRole` IAM Role with the `AmazonDynamoDBFullAccess` IAM Policy to allow the worker nodes access to DynamoDB. 
3. Each microservice has a helm chart to deploy the application. In each of the microservices folders (`clients-api`,  `inventory-api`, `renting-api`, `resource-api`, `front-end`) there's a [helm](./clients-api/infra/helm/) directory, within this directory is a [create.sh](./clients-api/infra/helm/create.sh) script that grabs the base domain from Route53 and installs the application to the cluster using the helm chart. Run this script for each of the microservices. 
4. Check that the pods have been deployed with `kubectl get pods -n development`
5. Check that the ingresses have been deployed with `kubectl get ingresses -n development`
6. You can also check the load balancer's listener rule to see each microservices rules. 
7. Check the Route53 DNS records that should have been automatically added via External DNS. 
8. Browse to bookstore.{{rootdomain}} from a browser to test the bookstore app! 
### EKS Integration with VPC Using the CNI Add-on  
Container Network Interface defines a standard that describe how the container orchestration manage container networking. The CNI addon is automatically installed on an EKS cluster. You can use CNI to customise networking for the cluster, for more info check out this [aws resource](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html#vpc-resize). In this course we'll customise the network by adding a secondary subnet to the cluster. To do this follow the steps below. 
1. Upgrade the CNI Addon by browsing the EKS page, clicking on Addons, Get more addons, Amazon CNI, select a version, in the optional settings select override, finally click create. 
2. Once the status of the addon is `active` run the [setup.sh](./Infrastructure/k8s-tooling/cni/setup.sh) script. This script associates a secondary cidr block with the VPC, gets the nat gateway, deploys a cloudformation template using [subnets.json](./Infrastructure/k8s-tooling/cni/subnets.json) which deploys the secondary subnets, and sets two environment variables in the cluster which allows for network customisation, and runs secondary [create.sh](./Infrastructure/k8s-tooling/cni/helm/create.sh) script which grabs the newly created subnets and the security groups associated with the cluster and then runs the [Helm Chart](./Infrastructure/k8s-tooling/cni/helm/Chart.yaml) which sets the ENI Config to use the new subnets. 
3. You can check the environment variables set using `kubectl describe node <node_name>` and scrolling to the `labels` block. 
4. You see the ENIConfig using `kubectl get ENIConfig`
5. Run `kubectl get pods -n development -o wide` and notice that the pods are still using the old subnets to move the pods to the new subnets simple destroy all of the node ec2 instances. EKS will spin up new worker nodes and as a result will spin up the pods into the new subnets. 
## Achieve the Principle of Least Privilege
### IAM Roles for Service Accounts (IRSA)
IRSA allows you to assign permission to applications running within a Kubernetes cluster. It integrates with the Kubernetes component Service Accounts and AWS IAM. It works by using the build in OpenID Connector that comes with EKS. You create an IAM Identity Prover using the OpenID Connect Issuer URL and then create IAM Roles with attached IAM policies which the Kubernetes Service Accounts assume to communicate with AWS Services. The first step to implementing IRSA is to create an Identity Provider in IAM and link it to the EKS cluster, to do this follow the steps below. 
1. Create the identity provider with the following command `eksctl utils associate-iam-oidc-provider --cluster=eks-acg --approve`. You can check that this command was successful by browsing to the IAM console and checking the Identity Providers tab. 
### Update the Bookstore Microservices to IRSA 
In a previous section we added Dynamo DB permissions to the whole worker node meaning that any application deployed to the worker nodes had access to all of DynamoDB. The next steps are going to implement _least privilege principles_ by creating application specific IAM policies. To implement this follow the steps below. 
1. Remove the DynamoDB permissions from the worker nodes. Browse to the NodeInstanceRole and remove the _AmazonDynamoDBFullAccess_. We do this manually because we manually added it in a previous step. 
2. Each microservice has it's own CloudFormation template for creating the IAM Policy that allows the microservice to access its own DynamoDB table. You can create it by running the [create-iam-policy.sh](./clients-api/infra/cloudformation/create-iam-policy.sh) in each of the microservices directory. Please note, you'll need to cd into the cloudformation directory before running the script. 
3. Create the Kubernetes Service Account that will use the above IAM policy `eksctl create iamserviceaccount --name <MICROSERVICE_NAME>-api-iam-service-account --namespace development --cluster eks-acg --attach-policy-arn <ARN_FOR_IAM_POLICY> --approve` this command will create both the service account in the k8s cluster and the IAM role in AWS. You can check the k8s service account with `kubectl get sa --namespace development` and you can check the IAM Role by checking the latest Cloudformation template that has been created. You can also find the arn for the IAM role by running the following command: `kubectl describe sa -n development <SERVICE_ACCOUNT_NAME>` in the output notice the _annotations_ field and see the value for _eks.amazonaws.com/role-arn_. 
4. The next step is to configure the application that is running on the k8s cluster to use the newly created service account. The code for this change is stored in each of the microservice's [helm-v2](./clients-api/infra/helm-v2/) directory. Notice on line 23 the new line for `serviceAccountName:`. To deploy the change run the [setup.sh](./clients-api/infra/helm-v2/create.sh) script. You can describe the newly create pod with `kubectl describe pods -n development <POD_NAME>` and you'll notice that there is now a `Service Account` in use and some new environment variables such as `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE`. 
5. Repeat steps 2-4 for the other microservices! 
### Update the K8s Tooling Applications to IRSA
Great, in the last section we updated the Bookstore Microservices to follow the least privilege. However, the worker nodes still have the IAM Policies to allow each of the K8s tooling applications to function. In this section we'll be updating this so that they also follow least privilege. 
1. The first step is to manually remove those IAM Policies. Locate the NodeInstanceRole and remove the `aws-load-balancer..`, `AmazonRoute53FullAccess`, and `AmazonEKS_CNI_Policy` IAM Policies from the role. 
2. First we'll update the load balancer. Run the following [create-irsa.sh](./Infrastructure/k8s-tooling/load-balancer-controller/create-irsa.sh). This script deploys a CloudFormation template that creates the IAM Policy for the service account to use. Then it creates the service account and IAM role using the `eksctl create iamserviceaccount` command. Before updating the helm chart to use the new service account. 
3. Then we'll update External DNS. Run the following [create-irsa.sh](./Infrastructure/k8s-tooling/external-dns/create-irsa.sh). This script differs from the last one as it doesn't need to create a new IAM Policy as it uses Amazon's _AmazonRoute52FullAccess_ IAM Policy. So, it creates the service account and IAM role using the `eksctl create iamserviceaccount` command. Before updating the helm chart to use the new service account. 
4. Finally, we'll update CNI. Run the following [create-irsa.sh](./Infrastructure/k8s-tooling/cni/setup-irsa.sh). This script differs from the last as it overrides the existing service account. If you've got a good eye you'll notice in the output of the previous commands it asked for an override flag. This script provides this flag, see line 10. Then it deletes the pods which will trigger a new pod to be created and the new changes will apply to the new pods. Notice that the script doesn't deploy a helm chart as CNI is already installed on the cluster by default. 
## More Power for Less Money
By default when we spin up and EKS cluster is uses On-Demand Instances which have some pros and cons. 
| Pros                                       | Cons                                                           |
|--------------------------------------------|----------------------------------------------------------------|
| Full control over the virtual machine      | Not that good for starting small                               |
| Autoscaling capabilities                   | Reserved Instances could work, but are you sure about capacity |
| Fairly quick stand-up process              | Long-running machines = more maintenance                       |
| Could take advantage of reserved instances | Could get expensive fast                                       |
In this section we'll look at some ways to optmise the cost of your EKS cluster. 
### Spot Instances
Spot instances are short running machines, they cost around 80% less than On-Demand, and they integrate with EKS. When using spot instances you set a price point at wish you wish to pay for the instances and amazon claims back those instances if the price increases past your threshold. To implement spot instances follow the steps below. 
1. First let's get some information about the existing nodegroup with the command `eksctl get nodegroups --cluster eks-acg`. Notice the TYPE is unmanaged. 
2. Now deploy a new nodegroup that uses spot instances by changing dir to ./Infrastructure/eksctl/02-spot-instances and running `eksctl create nodegroup -f cluster.yaml`. Open [cluster.yaml](./Infrastructure/eksctl/02-spot-instances/cluster.yaml) and inspect the nodeGroups section. The config responsible for setting spot instances is shown below. For more info, check out the [ekstctl docs](https://eksctl.io/usage/spot-instances/)
```yaml
instancesDistribution:
      instanceTypes: ["t3.medium", "t3a.medium"]
      onDemandPercentageAboveBaseCapacity: 0
```
3. You can now delete the on-demand nodegroup with `eksctl delete nodegroup --cluster eks-acg eks-node-group`
### Spot Instances and Node Termination Handler
Great, we've reduce the cost of our worker nodes by implementing spot instances. However, as spot instances can be taken away in any given moment by AWS they are not highly available and can cause downtime in your applications. The termination handler exists on a pod inside the worker node and listens out for the AWS Termination notification. When the worker node received this termination notification from AWS the Termination Handler moves the pods to another available worker node before the worker node is shutdown. To implement Termination Handler follow the steps below. 
1. First, ensure that you have the AWS eks-charts repo added in helm with `helm repo add eks https://aws.github.io/eks-charts`. 
2. Create the pods with helm using `helm install aws-node-termination-handler --namespace kube-system eks/aws-node-termination-handler`
### EKS Managed Node Groups
EKS Managed Node groups are exactly what they sound like, fully managed worker nodes from AWS. It has no additional cost, autoscaling is handled by AWS, and supports both On-Demand and Spot Instances. Check out the [cluster.yaml](./Infrastructure/eksctl/03-managed-nodes/cluster.yaml). The config block responsible for managed worker nodes is shown below. 
```yaml
managedNodeGroups:
  - name: eks-node-group-managed-nodes
    instanceType: t3.medium
    desiredCapacity: 3
    privateNetworking: true
```
To configure managed worker nodes follow the instructions below. 
1. Run the following command to configure managed nodes `eksctl create nodegroup -f cluster` from within [Infrastructure/eksctl/03-managed-nodes/](./Infrastructure/eksctl/03-managed-nodes/). 
2. You can confirm that the nodegroup is managed by running `eksctl get nodegrops --cluster eks-acg` and then notice the TYPE column is managed. 
3. Now that we have a fully managed nodegroup delete the spot instance node group with `eksctl delete nodegroup --cluster eks-acg eks-node-group-spot-instances`
### EKS and Fargate
Fargate is Amazon's serverless containers platform that was originally implemented for |ECS but has since been extended to EKS. Pricing is at the resource level for what the pods consume. It integrates with Kubernetes files and is a cost effective option since you never overpay you pay for what the pods require. Fargate uses a scheduler that has a profile, you specify whether to use this profile in your k8s files. If you use it the Fargate scheduler spins up the pod in a Fargate runtime environment. The Fargate scheduler works along side the normal EKS scheduler so if you don't specify the Fargate profile you can still schedule your pods into physical kubernetes nodes. To implement Fargate follow the steps below. 
1. From the [Infrastructure/eksctl/04-fargate dir](./Infrastructure/eksctl/04-fargate/) run the following command to deploy the Fargate profile `eksctl create fargateprofile -f cluster.yaml`. Open the [cluster.yaml](./Infrastructure/eksctl/04-fargate/cluster.yaml) file and notice the `faragteProfiles` section, this configuration will deploy pods with the `development` namespace into a Fargate runtime. 
2. Once deployed you can check it out in the EKS console by scrolling to the bottom of the page, notice the Fargate Profiles section. 
3. After deploying the Fargate profile you'll notice that the pods in the development namespace are still on the physical worker nodes run `kubectl get pods -n development -o wide` and note the NODE is an ec2 instance. This is because the Fargate scheduler only kicks in when pods are being scheduled and not already running. If you delete the pods you'll notice that they get spun up in the Fargate runtime confirmed by fargate prefix on the NODE column entry. 

> Pro tip: you can delete all of the running pods using these commands `kubectl get pods -n development | grep Running | awk '{print $1}'` and ``kubectl delete pods -n development `!!```

Congrats the Bookstore pods are now running serverless! 
### Summary Table
| Comparison Items | Unamanaged | Managed | Fargate |
-----------------------------------------------------
| Pricing Model    | EC2        | EC2     | Pod     |
| Operating System | Custom AMI | Always latest EKS-optimised AMI | N/A |
| Access level to the customer | Customer manages access to the node | SSH access only (if configured) | No possible access to the machine |
| Node/Pod relationship | A node can host multiple Pods | A node can host multiple Pods | A node can host only one Pod | 
| Node control level | Creations, upgrades, and terminations in customer's hands | Creations, upgrades, and terminations, in AWS' hands | Creations and terminations in AWS' hands |    

## Continuous Integration and Continuous Deployment
CICD is the process of safely and efficiently getting code into production. Ok, there's more to it than that but this course is focused on EKS and won't dive too deep into CI/CD if you're interested check out [ACG's Implementing a Full CI/CD Pipeline](https://learn.acloud.guru/course/93e6884d-ac98-49b6-a9cd-5e357d2dbb41/overview). In this section we're going to be integrating a CI/CD pipeline into EKS so that we can package up our Bookstore code and deploy it to a k8s cluster. There are many tools that can help achieve this but the ones we're using in this course are:
- CodeCommit: For source control management 
- CodeBuild: For building docker images and pushing them to ECR 
- CodePipeline: For glueing everything together into a pipeline 
### Workflow Definition
Each microservice has their own repo in CodeCommit, and features are added via a separate branch, when the branch is merged to the main branch then the CI/CD Pipeline is triggered. The workflow can be broken down into steps as follows. 
1. A developer has access to main branch which contains the source code which is stored on CodeCommit
2. The developer creates a new feature branch and makes changes to the code. 
3. The developer merges the feature branch into the main branch. 
4. The above step triggers CodeCommit to send out a CloudWatch event stating that changes have been merged into the main branch. 
5. CodePipeline orchestrates the following. 
  a. Source Stage: CodeBuild retrieves code from CodeCommit
  b. Build Stage: CodeBuild builds the Docker image and pushes it to ECR 
  c. Deploy Dev Stage: CodeBuild updates the app version in EKS using Helm
  d. Manual Approval: CodePipeline awaits for manual intervention to move to the next stage, this can be whilst the QA team tests the dev environment, for example. 
  e. Deploy Prod Stage: CodeBuild pushes the latest docker image into the production k8s cluster

The versioning system we're going to use for this project is MAJOR.MINOR.GIT_COMMIT_SHA. For example, `1.0.af873c03`. For more information on versioning check out [semver.org](https://semver.org/). 
### Source Code and Building 
In the following steps we're going to create the CodeCommit repo for the microservices and create the CodeBuild scripts to push the container images to ECR. 
1. Deploy [cicd-1-codecommit.yaml](./Infrastructure/cloudformation/cicd/cicd-1-codecommit.yaml) with `aws cloudformation deploy --stack-name <MICROSERVICE_NAME>-api-codecommit-repo --template-file cicd-1-codecommit.yaml --parameter-overrides AppName=<MICROSERVICE_NAME>-api`. This CloudFormation Stack simply deploys the CodeCommit repo for the specified microservice. 
2. Browse to your IAM User (cloud_user for ACG Cloud Playground Users) and generate _HTTPS Git credentials for AWS CodeCommit_ and copy and paste the username and password somewhere safe and secure. 
3. Next setup git by changing directory into whichever microservice you just deployed a repo for and quickly running the following commands, note that you'll only have to setup the user and email once.  
```bash
git config --global user.email "cloud-user@acg.com"
git config --global user.name "cloud-user"
git init 
git add . 
git status 
git commit -m "initial upload"
git remote add origin <CODE_COMMIT_URL_FOUND_IN_THE_AWS_CONSOLE>
git push origin master 
<SUPPLY_YOUR_USERNAME_AND_PASSWORD_NOTED_FROM_STEP_2>
```
4. After completing the last step you've uploaded the source code for the microservice to the CodeCommit repo. Next, we'll expand the existing CloudFormation template that created that repo with the following command `aws cloudformation deploy --stack-name <MICROSERVICE_NAME>-api-codecommit-repo --template-file cicd-2-ecr-and-build.yaml --parameter-overrides AppName=<MICROSERVICE_NAME>-api --capabilities CAPABILITY_IAM`. This CloudFormation template creates and ECR Repo for storing docker images, IAM permissions, and a CodeBuild Project which builds the docker image and pushes it to ECR. 
5. Checkout the corresponding [buildspec.yaml](./clients-api/infra/codebuild/buildspec.yml) in each of the microservices `infra/codebuild` directories for details steps that the codebuild project executes. 
6. From the CodeBuild AWS Console click into the newly created project and click on start build. Once it completes checkout the ECR repo for the newly created docker image. Repeat the above steps for each microservice.
7. Once you have the CodeCommit Repo, ECR Repo, and CodeBuild Project for each microservice, it's time to automate the build process with CodePipeline. To do this run `aws cloudformation deploy --stack-name <MICROSERVICES_NAME>-api-codecommit-repo --template-file cicd-3-automatic-build.yaml --parameter-overrides AppName=<MICROSERVICES_NAME>-api --capabilities CAPABILITY_IAM`. This updates the CloudFormation stack to include; A CloudWatch Event Rule that triggers when changes to the source code are made and triggers a CodePipelineBuild, A Code Pipeline with the source and build stage as mentioned at the start of this section. 
8. You can inspect the CodePipline further on the AWS console. You can also test the pipeline by making a change to the master branch which will trigger the pipeline. Repeat for the other microservices. 
### Automatic Build and Deploy to Development Environment
In this section we're going to automate the deployment to the development environment when a change is made to the codebase. We're going to be using CodeBuild with Helm to achieve this. 

First we'll be deploying the [cicd-4-deploy-development](Infrastructure/cloudformation/cicd/cicd-4-deploy-development.yaml) Cloudformation template. This template adds a stage that deploys the app to the development namespace within the kubernetes cluster. It does this by using the [buildspec.yaml](clients-api/infra/codebuild/deployment/buildspec.yml) locates in each microservice's `codebuild/deployment` directory. 

This buildspec.yaml file installs the `awscli`,`helm` and `kubectl` cli tools, updated the kubeconfig to point to the eks cluster, and then finally installs the app using helm through the [create.sh](clients-api/infra/helm-v4/create.sh) script. 

Follow the steps below to complete this process. 
1. Before the stage can run the IAM Role for the pipeline must be able to authenticate with the EKS cluster. To do this we update the clusters config map to add the `IAM Service Role` user to the group `system:masters` (This is overkill but for the demo it works). ` eksctl create iamidentitymapping --cluster eks-acg --username [MICROSERVICE_NAME]-api-deployment --group system:masters --arn [MICROSERVICES_IAM_SERVICE_ROLE_ARN]`. You can grab the Pipeline's IAM Role from CloudFormation. 
2. Repeat this process for all of the Micro services. 
3. Change directories into `Infrastructure/cloudformation/cicd` and run `aws cloudformation deploy --stack-name <MICROSERVICES_NAME>-api-codecommit-repo --template-file cicd-4-deploy-development.yaml --parameter-overrides AppName=<MICROSERVICES_NAME>-api --capabilities CAPABILITY_IAM` for each microservice. 
4. You can test each pipeline by making a change to the codebase and watching the pipeline complete. 
5. You can confirm the success by describing the new pods that gets created by the pipeline. Notice that the Image will end in the latest version of the container image. 

### Automatic Build and Deploy to Production Environment
In this section we're going to focus on setting up the the Production environment and then configuring the CICD Pipeline to deploy into this environment after a manual approval. 
### Setting up the Production environment 
These are the steps we're going to take for creating the production environment. 
#### Point the Development Environment to a different URL
We're going to be deploying a new version of the application with a new [helm chart](inventory-api/infra/helm-v5). In the [ingress.yaml](inventory-api/infra/helm-v5/templates/ingress.yaml) notice on line 19 `host: {{ eq .Release.Namespace "development" | ternary "inventory-api-development" "inventory-api" }}.{{ .Values.baseDomain }}` basically if the pod is deployed to a development namespace then it's host name is postfixed with `-development` otherwise it is not. 

1. Modify the deployment [buildspec.yaml](inventory-api/infra/codebuild/deployment/buildspec.yml) file. Under the build phase change the command `cd infra/helm-v4` to `cd infra/helm-v5`. Push your changes to the master branch. 
2. Watch the pipeline complete and check that the name of the ingress has changed with `kubectl get ingress -n development | grep [MICROSERVICE_NAME]`. Notice the `-development` postfix. 
3. Repeat for the other Microservices 
#### Create DynamoDB Tables
This one is a relatively simple process as we've done it before for the development environment. We need to run the [create--dynamodb-table.sh](inventory-api/infra/cloudformation/create-dynamodb-table.sh) with the namespace of production. 
1. Change into the dir [MICROSERVICE_NAME]-api/infra/cloudformation/ and run `./create-dynamodb-table.sh) production`
2. Repeat for the other microservices 
3. Now we need to add the IAM permissions to allow the production pods to access the DynamoDB tables. Simple run `./create-iam-policy.sh production` 
4. Repeat for all of the Microservices

#### Create IAM Service Accounts 
As with the development environment the production environment needs IAM Roles and K8s Service accounts to apply the principle of least privilege to the pods. To create them follow the steps below. 
1. Grab the IAM Policy Arn we created in the last step by browsing to the `production-iam-policy-[MICROSERVICE_NAME]-api` CloudFormation stack. 
2. In the console run the following command to create the k8s service account `eksctl create iamserviceaccount --name [MICROSERVICE_NAME]-api-iam-service-account --namespace production --cluster eks-acg --attach-policy-arn [IAM_POLICY_ARN] --approve`
3. You can verify this by running `eksctl get iamserviceaccounts --cluster eks-acg`. Notice that the service account name matches the one create for the development environment. This is ok because they're in different namespaces. 
4. Repeat for each microservice. 

#### Install Helm charts microservices
Now let's install the production helm charts to deploy each microservice to the production namespace. 
1. Grab the version of the latest ECR image by running `aws ecr list-images --repository-name [ECR_REPO_NAME]`. In the response grab the latest imageTag. 
2. Change directory into the microservice's /infra/helm-v5 directory. Then run `./create.sh production [imageTag]`. This script simply installs the microservice to the production namespace using the latest image. 
3. Repeat the process for the rest of the microservices.

### Extending the CI/CD Pipeline 
Now let's finish off the CI/CD pipeline. First we'll add in a manual approval step. In the real world this allows Quality Assurance teams to test the applications in the dev environment thoroughly before they're released into a production environment. Then we'll add a stage that deploys the application into the production environment. 
1. Deploy the [cicd-5-deploy-prod.yaml](Infrastructure/cloudformation/cicd/cicd-5-deploy-prod.yaml) CloudFormation template with `aws cloudformation deploy --stack-name <MICROSERVICES_NAME>-api-codecommit-repo --template-file cicd-5-deploy-prod.yaml --parameter-overrides AppName=<MICROSERVICES_NAME>-api --capabilities CAPABILITY_IAM`. This template creates a new CodeBuild project which will deploy the application into the production namespace with the `-production` postfix. It also adds in a manual approval step. 
2. Make a change to codebase to watch the pipeline in action! 

## Service Mesh with App Mesh 
What is a service mesh? 
> A service mesh is a software layer that handles all communication between services in applications - [AWS](https://aws.amazon.com/what-is/service-mesh/)

What is App Mesh?
- Service Mesh in AWS
- Manages how services communicate 
- Provides deep visibility 
- Plugs into the existing architecture 
- Integrates with other AWS services 
- Control plane is AWS's responsibility 

In this section we're going to: 
- Install the App Mesh Controller
- Create the Mesh through Kubernetes
- Create App Mesh Components for Each App
- Enable Visibility with with X-Ray
- Run through Retry Policies
### Installing the App Mesh Controller
In this section we're going to create the `appmesh-system` namespace, App Mesh CRDs and service accounts/IAM Roles and policies. To do so, follow the steps below. 
1. Run the [configure-app-mesh.sh](./Infrastructure/service-mesh/configure-app-mesh.sh). This script deploys a CloudFormation Template that creates the IAM Policy that service account is going to use. It then creates the service account with eksctl before installing the appmesh-controller with helm. 
2. You can check the deployment with `kubectl get pods -n appmesh-system` and `kubectl logs -n appmesh-system ${pod_name}`
### Creating the Mesh through Kubernetes 
In this section we're going to create the mesh from the Custom Resource Definitions(CRD). The mesh object will be created by the controller once the custom resource is added. Then we're going to modify the namespace by adding labels to enable the controller to look at our namespace and create the App Mesh components based on the CRD. 
1. Making sure your in [Infrastructure/service-mesh](Infrastructure/service-mesh) deploy the development mesh with by running `kubectl apply -f development-mesh.yaml`.
2. Confirm with `kubectl get Mesh` you can also browse to the AWS App Mesh console and view it from there. 
3. Label the namespace to match it with the mesh by `kubectl label namespace development mesh=development-mesh` and `kubectl label namespace development "appmesh.k8s.aws/sidecarInjectorWebhook"=enabled`. The last label ensures that the pod is injected with a side car that facilitates internal communication between pods. 
4. Confirm with `kubectl describe namespace development` notice the labels section now contains the above labels. 
### Creating AppMesh Components for Each App
In this section we're going to create the following three components for each microservice:
- Virtual Node - Represents the application 
- Virtual Router - Holds logic and rules for the traffic 
- Virtual Service - Used for accessing the application

The above components will be added by a newer version of the helm chart. You can check out the helm charts in each microservice's [helm-v6 directory](clients-api/infra/helm-v6). Specifically the [templates/mesh-components.yaml](clients-api/infra/helm-v6/templates/mesh-components.yaml). 

1. Before we can implement new App Mesh components we need to update the microservices IAM permissions. Run [create-iam-policy-app-mesh.sh](clients-api/infra/cloudformation/create-iam-policy-app-mesh.sh) in each of the microservice directory. 
2. Up until this point we haven't created a service account for the front end microservice. However, we need to do this now as the service will need access to the apis. Create a service account with `eksctl create iamserviceaccount --name front-end-iam-service-account --namespace development --cluster eks-acg --attach-policy-arn ${FRONT_END_IAM_POLICY_ARN} --approve`. The Arn can be found by checking the outputs of the cloudformation stack that creates it `aws cloudformation describe-stacks --stack development-iam-policy-front-end`
3. Update the app via the latest helm chart. But first grab the latest image of the microservice with `aws ecr list-images --repository-name bookstore.${microservice_name}`. Then run `./create.sh development ${image_tag}` from the [${microservice_name}infra/helm-v6](clients-api/infra/helm-v6) directory. Do this for all of the microservices. 
4. Then delete the pods for the new components to deploy. After the new pods have been created notice that in the READY column there are 2 containers. Describe the pod and notice the new `envoy` container. Which is the sidecar container we setup in the last step. 
5. You can also run  `kubectl get VirtualNodes -n development` to see the newly created virtual nodes. You can also browse to the AWS AppMesg console to view the Virtual Routers, Services, and Nodes. 
Good news! We've implemented an AppMesh and now our applications can communicate with one another internally. 
### Enable visibility with XRAY 
Xray is a tracing service for communications between components that integrates with other AWS services and is a serverless service. To implement Xray follow the steps below. 
1. Update each microservice's IAM policy to allow Xray permissions by running [create-iam-policy-x-ray.sh](inventory-api/infra/cloudformation/create-iam-policy-x-ray.sh). This script simply runs a [cloudformation template](inventory-api/infra/cloudformation/iam-policy-app-x-ray.yaml) that updates the IAM Policy with the permissions needed. 
2. Once the permissions are in place run [x-ray-setup.sh](Infrastructure/service-mesh/x-ray-setup.sh). This script updates the appmesh-controller app via a Helm chart. Notice the `--set tracing.enabled=true and --set tracing.provider=x-ray` on line 9 and 10. 
3. Delete all of the pods and notice that when they spin back up again there are 3 containers in each pod. If you describe the pod there should be a new xray container attached to it. 
4. You can check out Xray on the AWS Console and see how you can use it to troubleshoot communications between applications. 
### Retry Policies
Retry policies are configured on the Virtual routers, they force a communication to keep retrying between two services when one of the services is unavailable. This can be useful if a pod is restarting after an issue. Rather than failing the communication it is kept alive until the service is back. To configure it follow the steps below. 
1. The retry policy is defined in each microservices helm chart. Specifically [mesh-components.yaml](inventory-api/infra/helm-v7/templates/mesh-components.yaml). See, `retryPolicy` on line 62. 
2. From the console  run `./create.sh development ${image_tag}` from [inventory-api/infra/helm-v7](inventory-api/infra/helm-v7/). This script simply runs the latest version of the helm chart which enables the retry policy. 
3. Do this for all microservices. 
4. You can confirm this from the AppMesh AWS Console in the Virtual routers section. 

## Conclusion
Wow that was alot! If you've followed along with me you will have a fully functioning application running on EKS with a tonne of useful features enabled! I hope you had as much fun as I did running through this course! 
