##Pre-requisites :
#1. An AWS account
#2. An EC2 instance to run awscli and kubernetes CLI or install the clients locally
#3. A VPC, a  security group and subnets in different availability zone of the same region
#4. A role for Kubernetes service EKS to create resources. Follow the link: https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html

#Follow the below steps in a VM
#Install Python-pip to use awscli in a virtualenv
wget -q -O-  https://bootstrap.pypa.io/get-pip.py | python
pip install virtualenv
#Python should be of the version > 2.7.9
virtualenv --python /usr/lib/python3 /root/.venv
source /root/.venv/bin/activate
#install awscli using pip
pip install awscli

#Install AWS CLI to create a Kubernetes Cluster
#Set up credentials in a credentials file and export in the environment

aws config
export AWS_ACCESS_KEY_ID=
export AWS_SECRET_ACCESS_KEY=

#Create an EKS using the command:
KEY_NAME=''
SUBNET_ID_LIST=''
#SUBNET_ID_LIST="subnet-6c42f127,subnet-bfa332c6,subnet-95375ccf"
SECURITY_GROUP=''
CLUSTER_NAME=''
EKS_ROLE_ARN=''
VPC_ID=''
#EKS_ROLE_ARN="arn:aws:iam::698623830975:role/eksrole"

aws eks create-cluster --name $CLUSTER_NAME --role-arn $EKS_ROLE_ARN --resources-vpc-config subnetIds=$SUBNET_ID_LIST,securityGroupIds=$SECURITY_GROUP_LIST
#SAMPLE: aws eks create-cluster --name $CLUSTER_NAME --role-arn arn:aws:iam::698623830975:role/eksrole --resources-vpc-config subnetIds=subnet-6c42f127,subnet-bfa332c6,subnet-95375ccf,securityGroupIds=sg-8bd13ffa

#We need to set up heptio-authenticator-aws binary to authenticate for EKS cluster. Set it up before working on the cluster.
curl -o heptio-authenticator-aws https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-06-05/bin/linux/amd64/heptio-authenticator-aws
chmod +x ./heptio-authenticator-aws
#Path should have heptio-authenticator-aws as the first dir.
cp ./heptio-authenticator-aws /usr/bin/heptio-authenticator-aws && export PATH=/usr/bin:$PATH

#Once the EKS cluster is up and running, we need to setup kubectl client to work with it.
#Install kubectl on the instance
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
touch /etc/apt/sources.list.d/kubernetes.list
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install kubectl=1.10.0-00
#Check the version of kubectl
kubectl version

#After installing kubectl, create a kube config
mkdir /root/.kube

wget -O /root/.kube/config https://raw.githubusercontent.com/harshitshah65/task-todo-to-kubernetes/master/config
CLUSTER_CA_DATA=`aws eks describe-cluster --name manasi --query cluster.certificateAuthority.data`
CLUSTER_ENDPOINT=`aws eks describe-cluster --name manasi --query cluster.endpoint`
##Edit the contents of the config file with the above variables

##TO_DO: change the variables using sed command directly

#Export your kubeconfig file
export KUBECONFIG=/root/.kube/config

#Run the command to check if cluster is getting connected.
kubectl get nodes

#you can add nodes to it using the below procedure
#using cloudformation stack available on S3 URL: https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-06-05/amazon-eks-nodegroup.yaml,
#create a stack which will help you add worker nodes to your cluster
STACK_NAME='demo'
aws cloudformation create-stack --stack-name $STACK_NAME --template-url https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-06-05/amazon-eks-nodegroup.yaml --capabilities CAPABILITY_IAM --parameters '[{"ParameterKey": "ClusterName", "ParameterValue": "'$CLUSTER_NAME'"},{"ParameterKey": "KeyName","ParameterValue": "'$KEY_NAME'"},{"ParameterKey": "NodeImageId","ParameterValue": "ami-73a6e20b"},{"ParameterKey": "Subnets","ParameterValue": "'$SUBNET_ID_LIST'"},{"ParameterKey": "NodeGroupName","ParameterValue": "demo-nodegroup"},{"ParameterKey": "ClusterControlPlaneSecurityGroup","ParameterValue": "'$SECURITY_GROUP'"},{"ParameterKey": "VpcId","ParameterValue": "'$VPC_ID'"},{"ParameterKey": "NodeAutoScalingGroupMinSize","ParameterValue": "3"},{"ParameterKey": "NodeInstanceType","ParameterValue": "t2.medium"},{"ParameterKey": "NodeAutoScalingGroupMaxSize","ParameterValue": "5"}]'

ARN_INSTANCE_ROLE=`aws cloudformation describe-stacks --stack-name $STACK_NAME --query Stacks[0].Outputs[0].OutputValue`

#Download the yaml file to apply kubernetes configuration onto the worker nodes, so that they are added into the cluster.
wget -O /root/aws-auth-cm.yaml https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-06-05/aws-auth-cm.yaml

##Edit the contents of the file with $ARN_INSTANCE_ROLE variable.

##TO_DO: change the variables using sed command directly

#Apply the config to the worker nodes
kubectl apply -f /root/aws-auth-cm.yaml

#Check the nodes are added to the cluster
kubectl get nodes -o wide --watch


##After the set up of Kubernetes master and the worker nodes is done, start with the deployment of the app.

#Clone the repository for the yaml files to be deployed.
git clone https://github.com/harshitshah65/task-todo-to-kubernetes.git
cd task-todo-to-kubernetes
##For High Availability Set up, we need to set up dynamic provisioning of the volumes for the MongoDB database.
#Issue with manually creating a Volume and attaching is - the Volume would have been created in us-west-2a;
#But the scheduled pod might be in the region us-west-2b as your worker nodes are spreaded across multiple AZs for HA purpose.
#Hence we use dynamic provisioning of volumes using storage class and persistent volume claims


#create a storage class
kubectl create -f storage-class.yaml

#Create a PVC for 5GB of storage on Elastic Block Storage
kubectl create -f persistent-volume-claim.yaml

#Create a mongo-db-deployment using mongo as an image
kubectl create -f mongo-controller.yaml

#Create a service for mongo-db to be used by web-deployment service.
kubectl create -f mongo-service.yaml

#Create a nodejs deployment with image node and version 0.10.40. Git clone the repository and run npm install and start the application while exporting the port.
kubectl create -f web-controller.yaml

#Create a load balancer service to export the URL to the outside world and load balance the requests between the pods.
kubectl create -f web-service.yaml

##And the deployment is DONE!

##Access your application using the External IP of web service
kubectl get svc/web -o wide

##PS: To change the repository of the branch of the app source, modify the web-controller.yaml.
