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
In a previous section we added Dynamo DB permissions to the whole worker node meaning that any application deployed to the worker nodes had access to all of DynamoDB. The next steps are going to implement _least privelege principles_ by creating application specific IAM policies. To implement this follow the steps below. 
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