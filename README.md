# AWSApplicationModernization

Welcome to my AWS Application Modernization Project that will guide you on how to create a Wordpress website using EC2 instance and RDS instance. Once you have confirmed that your Wordpress website is running well, we will then replatform the web service to Containers with help of ECS Fargate Services.
### Pre-requisites for this project
- AWS Account
- IAM Roles for running EC2 Instances, ECS Tasks

#### Lets Begin!!! 

### Stage 1 - Build AWS Core Infrastructure

We first need to build our Core Infrastructure. Navigate to AWS CloudFormation Service and Select Create Stack. Under Create Stack configurtion, Select the option “upload a template file”, then select Choose file to add your cloudformation yaml template (eg: appmodernization-cloudformation.yaml) and then create a stack. You can find the appmodernization-cloudformation.yaml in this repository. 
![](/Images/Img1.PNG)

Cloudformation template “appmodernization-cloudformation.yaml” will then create the following:

- 1 VPC CIDR 172.31.0.0/16
- 2 Public Subnets (one in each availability zone)
- 2 Private Subnets for Web server (one in each availability zone)
- 2 Private Subnets for DB (one in each availability zone)
- 2 Elastic IP's
- 1 InternetGateway
- 2 NatGateways (one in each availability zone)

Once your core infrastructure is ready, lets then create an EC2 instance.
### Stage 2 - Create an EC2 instance to host your Wordpress Web Service

Navigate to EC2 service, Click on Instances and then select “Launch Instance”, Select Amazon Linux 2 AMI 64-bit(x86) as your image

![](/Images/Img2.PNG)

Select the size of the instance to be t2.micro (part of free tier).

![](/Images/Img3.PNG)

Under Configure instance details, configure the details as seen below:

- Network: Target VPC (eg: Word Press Fargate Base Infrastructure)
- Subnet: Subnet-Word Press Fargate Base infrastructure-private-web-a
- Auto-assign Public IP: Enable 

![](/Images/Img4.PNG)

Pass the bootstrap script from the resources in the "user data" section when configuring an instance

![](/Images/Img5.PNG)

Give a name to your ec2 web sever by adding a tag. Add a tag by following the steps in the screenshot below. 

![](/Images/Img6.PNG)

Configure your Security Group, Select the option “Create a new Security group”, provide a security group name (eg: SSH and HTTP access) and provide a short description (eg: Allowed SSH and HTTP access to this instance from anywhere (0.0.0.0/0).

![](/Images/Img7.PNG)

Review Instance Launch Page and then selected “Launch” at the bottom right to bring up the ec2 instance. In the below screenshot, you can see that my ec2 instance is running well (2/2 checks passed)

![](/Images/Img8.PNG)

### Stage 3 - Create an RDS instance to setup your Wordpress website.

Navigate to RDS service and then click on “Create Database”. Select the Engine type to be MySQL and select the image version to be MySQL 5.7.22. 

![](/Images/Img9.PNG)

Select the template type to be of Free tier

![](/Images/Img10.PNG)

Under Settings, provide the name for DB instance identifier (eg: wordpress-DB, Master username (eg: admin), Master password (eg:set your own password)

![](/Images/Img11.PNG)

Select your DB instance class size to be “Burstable classes” – db.t2.micro

![](/Images/Img12.PNG)

For your VPC, select your Target VPC (eg: Word Press Fargate Base Infrastructure VPC) and select your subnet group associated with your database (eg: wp-db-subnet-group, this db subnet group should have already been created when you created your cloudformation stack), Under existing security groups, select your RDS security group and provide an availability zone. 

![](/Images/Img13.PNG)

Provide the name of your Database (eg: wpdatabase) and save the DB name as you will need this for installing your wordpress website.

![](/Images/Img14.PNG)

Navigate to RDS instance security group to allow traffic from your ec2 instance to your RDS instance over TCP port 3306. Under inbound rules, select the Target Type = MySQL/Aurora and Source = Security Group of your EC2 web server instance. 

![](/Images/Img15.PNG)

Head over to your web server EC2 instance and grab the DNS and then paste in your web browser.

![](/Images/Img16.PNG)

The following page will appear, then select “Let’s go!”

![](/Images/Img17.PNG)

You will get the page to enter the following details:
- Database Name = Initial Database Name (taken from RDS); wp-database in my case
- Database Host = RDS instance end-point (After creating an RDS instance, you will be able to retrieve the RDS end-point from AWS console)
- DB username = admin (in my case)
- DB password = Password provided when configuring RDS instance
- Table Prefix = wp_

Once you have entered all the above details, go ahead and click on “Submit”

![](/Images/Img18.PNG)

You will then be taken to the page which looks like below. Copy the contents on the file. 

![](/Images/Img19.PNG)

SSH into your ec2 web server instance, go to path /var/www/html and create a file wp-config.php. Paste the content from the file and paste it here onto wp-config.php.

![](/Images/Img20.PNG)

After you have finished pasting the script on your ec2 instance, go back to the wordpress web page and select “Run the installation”. Once the installation is successful, I was then able to login to my wordpress website successfully without any errors.

![](/Images/Img21.PNG)

Now that the Wordpress webiste is up and running well, the next step is to move my web service to Containers.
### Stage 4 - Time to migrate the webservice to Containers!!!

The question is how should I start?? Security groups is good starting point to get working. 

#### Create New Security Groups

We will need to create some additional security groups (4 security groups in total to be precise)

1. Load Balancer Security Group
 - Security Group name =  LB-SG
 - Description = loadbalancersecuritygroup
 - VPC = Target VPC

![](/Images/Img22.PNG)

2. Elastic Container Service Task Security Group
 - Security group name = ECS-Tasks-SG
 - Description = Allow communication between the LB and the ECS Tasks
 - VPC = Target VPC

 ![](/Images/Img23.PNG)

3. Elastic File System Security Group
 - Security Group name = EFS-SG
 - Description = Allow communication between ECS Tasks and EFS
 - VPC = Target VPC

  ![](/Images/Img24.PNG)

4. Modify existing Database Security Groups

Navigate to RDS Security group and modify the security group to allow inbound traffic on port 3306 from ECS-Task-SG to the target database (there should already be inbound rules for Webserver instance)

 ![](/Images/Img25.PNG)

Now that security groups are in place, the next step is it to create an Amazon EFS File system.
#### Create Amazon EFS file system to mount file system to web server

 ![](/Images/Img26.PNG)

Navigate to Network Tab and select “Create mount target”

 ![](/Images/Img27.PNG)

Selected Target VPC and added 2 mount targets
- Mount Target 1
    - Availability zone = eu-west-1a
	- Subnet ID = Private-web-a
	- Security groups = EFS-SG

- Mount Target 2
    - Availability zone = eu-west-1b
	- Subnet ID = Private-web-b
	- Security groups = EFS-SG

 ![](/Images/Img28.PNG)

 Now that my mount targets were setup, I then decided to mount the file system temporarily into the ec2 webserver instance to copy the source wordpress content

 #### Mounting file system to webserver instance

 SSH into your ec2instance webserver and ran the following commands:

 ![](/Images/Img29.PNG)

Ran the following command to mount Amazon EFS file system on a Linux Instance. Please do not copy the same command as your file system ID may be different in your case. In my case, File system ID is fs-06fa8bcfef6dbac05

 ![](/Images/Img30.PNG)

 Once filesystem was mounted, copy the whole /var/www/html/wp-content folder from the web server to the mounted file system with the following command:

 ![](/Images/Img31.PNG)

 Since we will be using the official wordpress docker image with RDS database, we will need to provide the database credentials, database name and server details for the wordpress configuration. 

 The best platform to use these parameters is the AWS Systems Manager Parameter Store instead of storing them inside the docker image or ECS task definition. 
#### AWS Systems Manager to store your database credentials

 Navigate to AWS console and search for AWS Systems manager and then select Parameter Store

 ![](/Images/Img32.PNG)

 Click on Create Parameter and enter the parameter details as seen below:

 ![](/Images/Img33.PNG)

 ![](/Images/Img34.PNG)

 ![](/Images/Img35.PNG)

#### Next up: Create Load Balancer

I then created and configured an AWS Elastic load balancer to support large number of request to our incoming web server (deployed in the form of containers)

Navigate to AWS console, under EC2 service, select Load Balancers

Click on create load balancer and select the option “Application Load Balancer”






