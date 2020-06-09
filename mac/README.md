
# Create a Dremio Cluster in AWS Using the Cloudformation Template - Macbook

This readme file describes how to launch a Dremio Data Lake Engine into an AWS Cloudformation stack using the AWS CLI commands on a Macbook Pro laptop computer.

## Step 1. Prerequisites

### 1.a Create an AWS Key pair

  SEE: https://docs.aws.amazon.com/cli/latest/reference/ec2/create-key-pair.html

  An AWS Key pair is used to setup a secure SSH session to your AWS EC2 instances. Create a new key pair using the following commands:
  
    $ aws ec2 create-key-pair --key-name Dremio_Keypair --query 'KeyMaterial' --output text > Dremio_Keypair.pem

  The public key is stored in AWS, but the private key is not stored in AWS. It is stored locally on your laptop. To view the gernetated private key, use the command:
  
    $ cat .\Dremio_Keypair.pem

    -----BEGIN RSA PRIVATE KEY-----
    MIIEpQIBAAKCAQEAzfSctGzsVzu8DHlWqjPybtJE4SO3ehop2AMqcEXj2Mb2PzWD84M/qH/uxZBM
    FP9lPXbIv3I/v0S2aAgKJkDxbSP+G+59jtMPrZJjj/VctHNEPCcaFlW/dsNAAKvur/AmeT1rZqOa
    kq1gcUE7BDjMgImkap2rD4HwQcGqg6Cz69Hl+UnFyraGO/tHQLWAVCA9IBUXO62wC42k6wlqLpZg
    ...
    7hEKcM9BodW5LBXpZ4T65nv1pdzOVMkiyvXEygM8lIe6AWMCgYEA0l+pWxIkcshmWIUbz5XpdQDq
    HHZdxXSjht4e0vbF6pp8GYnw7pSxLAAc3iawXnwvK06TuQBZHqbIsEG1bhi8NhphAhtPQb9OpiI2
    1FX23VLX3zP927/dclZRbPgF6aXHJcWUNsgrwhtqw5A1XtL/5Dy9piuF4DDsME2nZWuoA8g=
    -----END RSA PRIVATE KEY-----

  Save this file in a safe place because it is not stored anyplace else. It will be used for your PuttyTelnet or other SSH client if you wish to SSH into any of your EKS EC2 instances. You can display information about the new key pair using the command:
  
     $ aws ec2 describe-key-pairs --key-name Dremio_Keypair
	 
     KEYPAIRS        28:14:35:ba:7c:b7:0e:3f:4c:62:c9:45:6a:6e:54:eb:0a:19:d2:5c     Dremio_Keypair       key-073ec4d75b1b670d1

### 1.b Create an AWS VPC and Subnets

SEE: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-subnets-commands-example.html

  When you deploy an AWS EKS cluster, you must identify an existing AWS Virtual Private Cloud environment as well as a PRIVATE subnet (and optionally, a PUBLIC subnet).
  
  If you have permissions from your AWS administrator you can execute these commands yourself. If not, you should contact your AWS administrator and request that they create them for you (or provide you with existing VPC and subnet information).
  
  To create a new VPC, use the command:
  
     $ aws ec2 create-vpc --cidr-block 10.0.0.0/16

     VpcId: vpc-2f09d348
	 
  Using the VPC ID from the previous step, create a subnet with a 10.0.1.0/24 CIDR block.

     $ aws ec2 create-subnet --vpc-id vpc-2f09d348 --cidr-block 10.0.1.0/24

     ID: subnet-b46032ec

  Create a second subnet in your VPC with a 10.0.0.0/24 CIDR block.

     $ aws ec2 create-subnet --vpc-id vpc-2f09d348 --cidr-block 10.0.0.0/24

     ID: subnet-a46032fc
	 
  Now make your subnet public, so you can access the EKS cluster and later, Dremio via your desktop or laptop computer. If your organization does not allow public subnets, then you can ask your AWS administrator to make sure you can access your VPC subnets via a private network.

     $ aws ec2 create-internet-gateway

	 InternetGatewayId: igw-1ff8a07b 

  Attach the internet gateway to your VPC using the command:
  
     $ aws ec2 attach-internet-gateway --vpc-id vpc-2f09d348 --internet-gateway-id igw-1ff8a07b

  Create a custom route table for your VPC.

     $ aws ec2 create-route-table --vpc-id vpc-2f09d348

     RouteTableId: rtb-c1c8faa6 

  Create a route in the route table that points all traffic (0.0.0.0/0) to the Internet gateway (change "0.0.0.0/0" to the local subnet that your have your laptop running on).

     $ aws ec2 create-route --route-table-id rtb-c1c8faa6 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-1ff8a07b

  Confirm that your route has been created and is active, by describing the route table with the command:

     $ aws ec2 describe-route-tables --route-table-id rtb-c1c8faa6

  The route table is currently not associated with any subnet. You need to associate it with a subnet in your VPC so that traffic from that subnet is routed to the Internet gateway. First, use the describe-subnets command to get your subnet IDs. You can use the --filter option to return the subnets for your new VPC only, and the --query option to return only the subnet IDs and their CIDR blocks.
  
     PS C:\> aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-2f09a348" --query 'Subnets[*].{ID:SubnetId,CIDR:CidrBlock}'
	 
     [
         {
             "CIDR": "10.0.1.0/24", 
             "ID": "subnet-b46032ec"
         }, 
         {
             "CIDR": "10.0.0.0/24", 
             "ID": "subnet-a46032fc"
         }
     ]

  You can choose which subnet to associate with the custom route table, for example, subnet-b46032ec. This subnet will be your public subnet.

     $ aws ec2 associate-route-table  --subnet-id subnet-b46032ec --route-table-id rtb-c1c8faa6

  You can optionally modify the public IP addressing behavior of your subnet so that an instance launched into the subnet automatically receives a public IP address. Otherwise, you should associate an Elastic IP address with your instance after launch so that it's reachable from the Internet.
  
     $ aws ec2 modify-subnet-attribute --subnet-id subnet-b46032ec --map-public-ip-on-launch



## Step 2. Download the Dremio Cloudformation template

### Use git or curl to download the Dremio Cloudformation template file

     $ git clone https://github.com/dremio/dremio-cloud-tools

     $ cp dremio-cloud-tools/blob/master/aws/cloudformation/dremio_cf.yaml .

OR

     $ curl -O https://raw.githubusercontent.com/dremio/dremio-cloud-tools/master/aws/cloudformation/dremio_cf.yaml

## Step 3. Modify the Cloudformation template

### Modify the Cloudformation template to include your AWS region specific resources

     $ vi dremio_cf.yaml

## Step 4. Launch a Dremio cluster using an AWS Cloudformation template

### 4.a Validate the Cloudformation template 

     aws cloudformation validate-template --template-body file://dremio_cf.yaml

### 4.b Launch the cloudformation stack

     aws --region us-west-2 cloudformation create-stack --stack-name Gregs-Dremio-Cluster \
         --template-body file://dremio_cf.yaml \
         --tags "Key=Owner,Value=Greg-Palmer" "Key=Business-Unit,Value=Pre-Sales" \
         --parameters \
           "ParameterKey=useVPC,            ParameterValue=vpc-72cc4c0a" \
           "ParameterKey=useSubnet,         ParameterValue=subnet-3fa74b62" \
           "ParameterKey=dremioDownloadURL, ParameterValue=https://download.dremio.com/community-server/dremio-community-LATEST.noarch.rpm" \
           "ParameterKey=keyName,           ParameterValue=greg_palmer_eks_keypair" \
           "ParameterKey=clusterSize,       ParameterValue=Small--5-executors"

     
     arn:aws:cloudformation:us-west-2:384816939103:stack/Gregs-Dremio-Cluster/9dc09010-aa5b-11ea-816f-0aa27834ab52

  Get various progress reports on the stack creation process

     aws --region us-west-2 cloudformation describe-stacks --stack-name Gregs-Dremio-Cluster

     aws cloudformation describe-stack-events --stack-name Gregs-Dremio-Cluster

     aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE

### 4.c Get Dremio Web UI address

  Get the stack output that describes the URL for the Dremio Web UI

     aws --region us-west-2 cloudformation describe-stacks --stack-name Gregs-Dremio-Cluster --query "Stacks[0].Outputs[?OutputKey=='DremioUI'].OutputValue" --output text

## End of document

### Please direct questions or comments to greg@dremio.com




