# springboot
Step 0: Before you start
You will need to make sure you have the following components installed and set up before you start with Amazon EKS:

AWS CLI – while you can use the AWS Console to create a cluster in EKS, the AWS CLI is easier. You will need version 1.16.73 at least. For further instructions, click here.
Kubectl – used for communicating with the cluster API server. For further instructions on installing, click here.
AWS-IAM-Authenticator – to allow IAM authentication with the Kubernetes cluster. Check out the repo on GitHub for instructions on setting this up.
Step 1: Creating an EKS role
Our first step is to set up a new IAM role with EKS permissions.

Open the IAM console, select Roles on the left and then click the Create Role button at the top of the page.

From the list of AWS services, select EKS and then Next: Permissions at the bottom of the page.

createrole
Leave the selected policies as-is, and proceed to the Review page.

role review
Enter a name for the role (e.g. eksrole) and hit the Create role button at the bottom of the page to create the IAM role.

The IAM role is created.

Summary
Be sure to note the Role ARN, you will need it when creating the Kubernetes cluster in the steps below.

Step 2: Creating a VPC for EKS
Next, we’re going to create a separate VPC for our EKS cluster. To do this, we’re going to use a CloudFormation template that contains all the necessary EKS-specific ingredients for setting up the VPC.

Open up CloudFormation, and click the Create new stack button.

On the Select template page, enter the URL of the CloudFormation YAML in the relevant section:

https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-01-
09/amazon-eks-vpc-sample.yaml
Copy
create stack
Click Next.

specify details
Give the VPC a name, leave the default network configurations as-is, and click Next.

On the Options page, you can leave the default options untouched and then click Next.

create vpc

On the Review page, simply hit the Create button to create the VPC.

CloudFormation will begin to create the VPC. Once done, be sure to note the various values created — SecurityGroups, VpcId and SubnetIds. You will need these in subsequent steps. You can see these under the Outputs tab of the CloudFormation stack:

demo
Step 3: Creating the EKS cluster
As mentioned above, we will use the AWS CLI to create the Kubernetes cluster. To do this, use the following command:

demo
aws eks --region <region> create-cluster --name <clusterName> 
--role-arn <EKS-role-ARN> --resources-vpc-config 
subnetIds=<subnet-id-1>,<subnet-id-2>,<subnet-id-3>,securityGroupIds=
<security-group-id>
Copy
Be sure to replace the bracketed parameters as follows:

region — the region in which you wish to deploy the cluster.
clusterName — a name for the EKS cluster you want to create.
EKS-role-ARN — the ARN of the IAM role you created in the first step above.
subnetIds — a comma-separated list of the SubnetIds values from the AWS CloudFormation output that you generated in the previous step.
security-group-id — the SecurityGroups value from the AWS CloudFormation output that you generated in the previous step.
This is an example of what this command will look like:

aws eks --region us-east-1 create-cluster --name demo --role-arn
arn:aws:iam::011173820421:role/eksServiceRole --resources-vpc-config 
subnetIds=subnet-06d3631efa685f604,subnet-0f435cf42a1869282,
subnet-03c954ee389d8f0fd,securityGroupIds=sg-0f45598b6f9aa110a
Copy
Executing this command, you should see the following output in your terminal:

{
    "cluster": {
        "status": "CREATING",
        "name": "demo",
        "certificateAuthority": {},
        "roleArn": "arn:aws:iam::011173820421:role/eksServiceRole",
        "resourcesVpcConfig": {
            "subnetIds": [
                "subnet-06d3631efa685f604",
                "subnet-0f435cf42a1869282",
                "subnet-03c954ee389d8f0fd"
            ],
            "vpcId": "vpc-0d6a3265e074a929b",
            "securityGroupIds": [
                "sg-0f45598b6f9aa110a"
            ]
        },
        "version": "1.11",
        "arn": "arn:aws:eks:us-east-1:011173820421:cluster/demo",
        "platformVersion": "eks.1",
        "createdAt": 1550401288.382
    }
}
Copy
It takes about 5 minutes before your cluster is created. You can ping the status of the command using this CLI command:

aws eks --region us-east-1 describe-cluster --name demo --query 
cluster.status
Copy
The output displayed will be:

"CREATING"
Copy
Or you can open the Clusters page in the EKS Console:

clusters
Once the status changes to “ACTIVE”, we can proceed with updating our kubeconfig file with the information on the new cluster so kubectl can communicate with it.

To do this, we will use the AWS CLI update-kubeconfig command (be sure to replace the region and cluster name to fit your configurations):

aws eks --region us-east-1 update-kubeconfig --name demo
Copy
You should see the following output:

Added new context arn:aws:eks:us-east-1:011173820421:cluster/demo to 
/Users/Daniel/.kube/config
Copy
We can now test our configurations using the kubectl get svc command:

kubectl get svc

NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   2m
Copy
Click the cluster in the EKS Console to review configurations:

general configuration
Step 4: Launching Kubernetes worker nodes
Now that we’ve set up our cluster and VPC networking, we can now launch Kubernetes worker nodes. To do this, we will again use a CloudFormation template.

Open CloudFormation, click Create Stack, and this time use the following template URL:

https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-01-
09/amazon-eks-nodegroup.yaml
Copy
select template
Clicking Next, name your stack, and in the EKS Cluster section enter the following details:

ClusterName – the name of your Kubernetes cluster (e.g. demo)
ClusterControlPlaneSecurityGroup – the same security group you used for creating the cluster in previous step.
NodeGroupName – a name for your node group.
NodeAutoScalingGroupMinSize – leave as-is. The minimum number of nodes that your worker node Auto Scaling group can scale to.
NodeAutoScalingGroupDesiredCapacity – leave as-is. The desired number of nodes to scale to when your stack is created.
NodeAutoScalingGroupMaxSize – leave as-is. The maximum number of nodes that your worker node Auto Scaling group can scale out to.
NodeInstanceType – leave as-is. The instance type used for the worker nodes.
NodeImageId – the Amazon EKS worker node AMI ID for the region you’re using. For us-east-1, for example: ami-0c5b63ec54dd3fc38
KeyName – the name of an Amazon EC2 SSH key pair for connecting with the worker nodes once they launch.
BootstrapArguments – leave empty. This field can be used to pass optional arguments to the worker nodes bootstrap script.
VpcId – enter the ID of the VPC you created in Step 2 above.
Subnets – select the three subnets you created in Step 2 above.
stack details
Proceed to the Review page, select the check-box at the bottom of the page acknowledging that the stack might create IAM resources, and click Create.

CloudFormation creates the worker nodes with the VPC settings we entered — three new EC2 instances are created using the

As before, once the stack is created, open Outputs tab:

open outputs
 

Note the value for NodeInstanceRole as you will need it for the next step — allowing the worker nodes to join our Kubernetes cluster.

To do this, first download the AWS authenticator configuration map:

curl -O 
https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-01-09
/aws-auth-cm.yaml
Copy
Open the file and replace the rolearn with the ARN of the NodeInstanceRole created above:

apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: <ARN of instance role>
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
Copy
Save the file and apply the configuration:

kubectl apply -f aws-auth-cm.yaml
Copy
You should see the following output:

configmap/aws-auth created
Copy
worker node :-
           ec2-user# cd /opt
            opt# vi script.sh
        #sudo vi script.sh
         #sudo chmod +x /opt/script.sh
         #vi script.sh
      #!/usr/bin/env bash
sudo /etc/eks/bootstrap.sh --apiserver-endpoint <your endpoint> --b64-cluster-ca <yourca> <yourclustername>

Note :-Please add your eks cluster apiend_point _url and certificate_manager_url and eks_ cluster_name

Use kubectl to check on the status of your worker nodes:

kubectl get nodes --watch

NAME                              STATUS     ROLES     AGE         VERSION
ip-192-168-245-194.ec2.internal   Ready     <none>    <invalid>   v1.11.5
ip-192-168-99-231.ec2.internal   Ready     <none>    <invalid>   v1.11.5
ip-192-168-140-20.ec2.internal   Ready     <none>    <invalid>   v1.11.5



kubectl get svc

guestbook      LoadBalancer   10.100.17.29     aeeb4ae3132ac11e99a8d12b26742fff-1272962453.us-east-1.elb.amazonaws.com   3000:31747/TCP   7m
kubernetes     ClusterIP      10.100.0.1       <none>                                                                    443/TCP          1h
redis-master   ClusterIP      10.100.224.82    <none>                                                                    6379/TCP         8m
redis-slave    ClusterIP      10.100.150.193   <none>  

export M2_HOME=/usr/bin/apache-maven-3.6.3 export M2=$M2_HOME/bin
	export PATH=$M2:$PATH

http://18.212.187.4:5000/

docker tag spring-boot-websocket-chat-demo2 dilleswari/spring-boot-websocket-chat-demo2 :demo2

docker push dilleswari/spring-boot-websocket-chat-demo2:demo2

docker run -p 5000:8080 dilleswari/spring-boot-websocket-chat-demo2:0.0.1-SNAPSHOT

spring-boot-websocket-chat-demo]$ sudo docker run -d -p 5003:8080 dilleswari/spring-boot-websocket-chat-demo2:0.0.1-SNAPSHOT


create the deplyment yaml file for application as below formate

apiVersion: apps/v1           # API version
kind: Deployment              # Type of kubernetes resource
metadata:
  name: chat-server           # Name of the kubernetes resource
  labels:                     # Labels that will be applied to this resource
    app: chat-server
spec:
  replicas: 2                 # No. of replicas/pods to run in this deployment
  selector:
    matchLabels:              # The deployment applies to any pods mayching the specified labels
      app: chat-server
  template:                   # Template for creating the pods in this deployment
    metadata:
      labels:                 # Labels that will be applied to each Pod in this deployment
        app: chat-server
    spec:                     # Spec for the containers that will be run in the Pods
      containers:
      - name: chat-server
        image: dilleswari/spring-boot-websocket-chat-demo2:0.0.1-SNAPSHOT
        imagePullPolicy: IfNotPresent
        ports:
          - name: http
            containerPort: 8080 # The port that the container exposes
---
apiVersion: v1                # API version
kind: Service                 # Type of the kubernetes resource
metadata:
  name: chat-server           # Name of the kubernetes resource
  labels:                     # Labels that will be applied to this resource
    app: chat-server
spec:
  type: NodePort              # The service will be exposed by opening a Port on each node and proxying it.
  selector:
    app: chat-server          # The service exposes Pods with label `app=polling-app-server`
  ports:                      # Forward incoming connections on port 8080 to the target port 8080
  - name: http
    port: 8080
    targetPort: 8080
    
    and run the command 
    kubectl apply -f deployment.yaml
    Below commands for kuberenetes
    kubectl get svc --namespace=
   
  kubectl get pod --namespace=
  
  kubectl get ing --namespace=
  
  kubectl get pod --namespace=
  
   kubectl describe pod --namespace=
  
  https://34.203.232.228:30000

 kubectl delete serviceaccount --namespace kube-system tiller
    kubectl delete clusterrolebinding tiller-cluster-rule
   kubectl delete clusterrolebinding tiller-binding
kubectl delete serviceaccount --namespace kube-system tiller

kubectl get nodes -o wide
    
    
    
    
    
    Installing jenkins with helm charts
    
    prerequisites
    
    install the helm
    https://helm.sh/docs/intro/install/
    Initialize helm:
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl create clusterrolebinding tiller-binding --clusterrole=cluster-admin --serviceaccount kube-system:tiller
helm init --service-account tiller
Setup RBAC:
kubectl apply -f https://raw.githubusercontent.com/jenkinsci/kubernetes-plugin/master/src/main/kubernetes/service-account.yml
kubectl create clusterrolebinding jenkins --clusterrole cluster-admin --serviceaccount=jenkins:default


