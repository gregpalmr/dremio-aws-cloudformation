
# Create a Dremio Cluster in AWS Using the Cloudformation Template - Macbook

This readme file describes how to launch a Dremio Data Lake Engine into an AWS Cloudformation stack using the AWS CLI commands on a Macbook Pro laptop computer.

## Step 1. Install Required Command Line tools

### 1.a Install the AWS CLI program

See: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-mac.html

     $ brew install awscli

### 1.b Configure AWS CLI

     $ aws configure
          < enter aws access key id, etc>

     $ cat ~/.aws/config
     $ cat ~/.aws/credentials

### 1.c Install git

See: https://git-scm.com/download/mac

     $ brew install git

## Step 2. Create AWS Required Resources

Create the AWS key pair, VPC and Subnet objects needed by the Dremio Cloudformation template.

### 2.a Create an AWS Key pair

  See: https://docs.aws.amazon.com/cli/latest/reference/ec2/create-key-pair.html

  An AWS Key pair is used to setup a secure SSH session to your AWS EC2 instances. Create a new key pair using the following commands:
  
    $ aws ec2 create-key-pair --key-name Dremio-Keypair --query 'KeyMaterial' --output text > dremio_keypair.pem

  The public key is stored in AWS, but the private key is not stored in AWS. It is stored locally on your laptop. To view the gernetated private key, use the command:
  
    $ cat .\dremio_deypair.pem

    -----BEGIN RSA PRIVATE KEY-----
    MIIEpQIBAAKCAQEAzfSctGzsVzu8DHlWqjPybtJE4SO3ehop2AMqcEXj2Mb2PzWD84M/qH/uxZBM
    FP9lPXbIv3I/v0S2aAgKJkDxbSP+G+59jtMPrZJjj/VctHNEPCcaFlW/dsNAAKvur/AmeT1rZqOa
    kq1gcUE7BDjMgImkap2rD4HwQcGqg6Cz69Hl+UnFyraGO/tHQLWAVCA9IBUXO62wC42k6wlqLpZg
    ...
    7hEKcM9BodW5LBXpZ4T65nv1pdzOVMkiyvXEygM8lIe6AWMCgYEA0l+pWxIkcshmWIUbz5XpdQDq
    HHZdxXSjht4e0vbF6pp8GYnw7pSxLAAc3iawXnwvK06TuQBZHqbIsEG1bhi8NhphAhtPQb9OpiI2
    1FX23VLX3zP927/dclZRbPgF6aXHJcWUNsgrwhtqw5A1XtL/5Dy9piuF4DDsME2nZWuoA8g=
    -----END RSA PRIVATE KEY-----

  Save this file in a safe place because it is not stored anyplace else. It will be used for your PuttyTelnet or other SSH client if you wish to SSH into any of your Dremio EC2 instances. You can display information about the new key pair using the command:
  
     $ aws ec2 describe-key-pairs --key-name Dremio-Keypair
	 
     KEYPAIRS        28:14:35:ba:7c:b7:0e:3f:4c:62:c9:45:6a:6e:54:eb:0a:19:d2:5c     Dremio-Keypair       key-073ec4d75b1b670d1

### 2.b Create an AWS VPC and Subnets

See: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-subnets-commands-example.html

  When you deploy an AWS Dremio cluster, you must identify an existing AWS Virtual Private Cloud environment as well as a PRIVATE subnet (and optionally, a PUBLIC subnet).
  
  If you have permissions from your AWS administrator you can execute these commands yourself. If not, you should contact your AWS administrator and request that they create them for you (or provide you with existing VPC and subnet information).
  
  To create a new VPC, use the command:
  
     $ aws ec2 create-vpc --cidr-block 10.0.0.0/16

     VpcId: vpc-2f09d348
	 
  Using the VPC ID from the previous step, create a subnet with a 10.0.1.0/24 CIDR block.

     $ aws ec2 create-subnet --vpc-id vpc-2f09d348 --cidr-block 10.0.1.0/24

     ID: subnet-b46032ec
	 
  Now make your subnet public, so you can access the Dremio cluster and later, Dremio via your desktop or laptop computer. If your organization does not allow public subnets, then you can ask your AWS administrator to make sure you can access your VPC subnets via a private network.

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
  
     $ aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-2f09a348" --query 'Subnets[*].{ID:SubnetId,CIDR:CidrBlock}'
	 
     [
         {
             "CIDR": "10.0.0.0/24", 
             "ID": "subnet-b46032ec"
         }
     ]

  You can choose which subnet to associate with the custom route table, for example, subnet-b46032ec. This subnet will be your public subnet.

     $ aws ec2 associate-route-table  --subnet-id subnet-b46032ec --route-table-id rtb-c1c8faa6

  You can optionally modify the public IP addressing behavior of your subnet so that an instance launched into the subnet automatically receives a public IP address. Otherwise, you should associate an Elastic IP address with your instance after launch so that it's reachable from the Internet.
  
     $ aws ec2 modify-subnet-attribute --subnet-id subnet-b46032ec --map-public-ip-on-launch

## Step 3. Download the Dremio Cloudformation template

### Use git or curl to download the Dremio Cloudformation template file

     $ git clone https://github.com/dremio/dremio-cloud-tools

     $ cp dremio-cloud-tools/resources/dremio_cf.yaml .

OR

     $ curl -O https://raw.githubusercontent.com/dremio/dremio-cloud-tools/master/aws/cloudformation/dremio_cf.yaml

NOTE: If you are deploying to the AWS GovCloud, then copy the dremio_govcloud_cf.yaml file like this:

     $ cp dremio-aws-cloudformation/resources/dremio_govcloud.yaml .

## Step 4. Modify the Cloudformation template

### (Optional) Modify the Cloudformation template to include your AWS region specific resources

     $ vi dremio_cf.yaml

OR

     $ vi dremio_govcloud_cf.yaml

## Step 5. Launch a Dremio cluster using an AWS Cloudformation template

### 5.a Validate the Cloudformation template 

     $ aws cloudformation validate-template --template-body file://dremio_cf.yaml

### 5.b Launch the cloudformation stack

The Dremio Cloudformation template requires a parameter to tell it where to download the Dremio installer program from. If you would like to use the Enterprise version of Dremio, contact your local Dremio sales representative to get a copy of the file and upload it to your S3 bucket. If you would like to install the Community Edition of Dremio, use the parameter:

     ParameterKey=dremioDownloadURL,ParameterValue=https://download.dremio.com/community-server/dremio-community-LATEST.noarch.rpm

To launch the Dremio cluster in the Cloudformation stack, use this command:

     $ aws cloudformation create-stack --stack-name My-Dremio-Cluster \
         --disable-rollback \
         --capabilities CAPABILITY_IAM \
         --template-body file://dremio_cf.yaml \
         --tags "Key=Name,Value=My-Dremio-Cluster" "Key=Owner,Value=Greg-Palmer" "Key=Business-Unit,Value=Sales" \
         --parameters \
           ParameterKey=useVPC,ParameterValue=vpc-2f09d348 \
           ParameterKey=useSubnet,ParameterValue=subnet-b46032ec \
           ParameterKey=securityGroupInboundSourceCidr,ParameterValue=73.224.23.46/32 \
           ParameterKey=keyName,ParameterValue=Dremio-Keypair \
           ParameterKey=clusterSize,ParameterValue=Small--5-executors \
           ParameterKey=dremioS3BucketName,ParameterValue=<s3 bucket name> \
           ParameterKey=dremioDownloadURL,ParameterValue=s3://<s3 bucket name>/<path to dremio rpm file> 
     
     arn:aws:cloudformation:us-west-2:384816939103:stack/My-Dremio-Cluster/9dc09010-aa5b-11ea-816f-0aa27834ab52

  Get various progress reports on the stack creation process

     $ aws cloudformation describe-stacks --stack-name My-Dremio-Cluster

     $ aws cloudformation describe-stack-events --stack-name My-Dremio-Cluster

     $ aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE

### 5.c Get Dremio Web UI address

  Get the stack output that describes the URL for the Dremio Web UI

     $ aws cloudformation describe-stacks --stack-name My-Dremio-Cluster --query "Stacks[0].Outputs[?OutputKey=='DremioUI'].OutputValue" --output text

### Step 6. Delete the Dremio Cloudformation Stack

     $ aws cloudformation delete-stack --stack-name My-Dremio-Cluster 

---

Please direct questions or comments to greg@dremio.com



