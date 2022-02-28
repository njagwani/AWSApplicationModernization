# AWSApplicationModernization

Welcome to my AWS Application Modernization Project.

This page guide you on how to create a Wordpress website using EC2 instance and RDS instance. Once you have confirmed that your Wordpress website is running well, you will then migrate the Wordpress web service to Containers with help of ECS Fargate Services.

## Key Learnings and Takeaways!!!

1. Create your AWS core infrastructure using Cloud formation template.
2. Deploy your web service on EC2 instance and associate your DB with RDS instance.
3. How to work around creating Security groups and add necessary rules to route traffic securely. 
4. Deploy an Elastic File System and Application Load Balancer to be associated with your Containers. 
5. Using AWS Systems Manager to store your database credentials.
6. Deploy and Migrate current running webservice on ec2 instances to containers using Elastic Container Service (Fargate) by creating ECS tas
### Pre-requisites for this project

- AWS Account
- IAM Roles for running EC2 Instances, ECS Tasks
## AWS Application Migration Architecture  Diagram 

![](/Images/AWSNetworkDiagram.PNG)

### Lets Begin!!! ###
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
#### Using AWS Systems Manager to store your database credentials

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

 ![](/Images/Img36.PNG)

Configured Load Balancer by entering the Load Balancer name (eg: WP-LB), choose the VPC (Target VPC) and select your public subnets (public-subnet-a, public-subnet-b) as seen below:

 ![](/Images/Img37.PNG)

 ![](/Images/Img38.PNG)

Configured Security groups by choosing the LB-SG security group

 ![](/Images/Img39.PNG)

Configured routing, by selecting “Create Target Group”

 ![](/Images/Img40.PNG)

IP addresses based Target Group

 ![](/Images/Img41.PNG)

Provided name of the target group, select the Target VPC and registered the target successfully. 

 ![](/Images/Img42.PNG)

 ![](/Images/Img43.PNG)

Finally the time has arrived to work on Elastic Container Service 

#### Elastic Container Service

Started off by creating an ECS Cluster and selected “Networking only” then click on Next step

 ![](/Images/Img44.PNG)

In the Cluster configuration, provide the Cluster name (eg: wp-web-cluster) and check the box for “Enable Container Insights” to get in metrics through CloudWatch.  Then click on Create.

 ![](/Images/Img45.PNG)

The next step is to create an Amazon ECS Task Definition

Navigate to ECS service and then go to Task Definitions and Create new Task Definition. Choose FARGATE. Then click on “Next step”

 ![](/Images/Img46.PNG)

Configure Task and Container definition by filling the following fields:
 - Enter the Task Definition Name
 - Select ecsTaskExecutionRole for both Task Role and Task execution role
 - Select awsvpc for Network Mode
 - In the Task size, enter the Task memory (GB) and Task CPU (Vcpu)

 ![](/Images/Img47.PNG)

Since we are going to mount the Amazon EFS volume to container, we need to add volume first to our task definition before adding the container. Go down to the bottom of the task definition page and click “Add Volume”. 

In the “Add Volume”, provide a name for your volume (eg: wp-content) and select volume type as EFS. In the File system ID, select the EFS volume that was created earlier and enable “Transit encryption”. Then click on Add.

 ![](/Images/Img48.PNG)

Scroll up to container section and click on “Add container”

 ![](/Images/Img49.PNG)

Specify the Container name (eg: wp-wordpress-container) and Image (wordpress:latest – official image of Wordpress taken from docker hub marketplace). Enter Memory Limits and Port Mappings as seen below.

 ![](/Images/Img50.PNG)

Scroll down to the Environment variables section and configure parameters from the Parameter Store as seen below.

 ![](/Images/Img51.PNG)

 ![](/Images/Img52.PNG)

In STORAGE AND LOGGING, select the mount point and specify the container path /var/www/html/wp-content. Wordpress files were copied into Amazon EFS file system path (/var/www/html/wp-content) which should be allowed for wordpress to pick these up. Go ahead and click on Create on the task definition page.

 ![](/Images/Img53.PNG)

Last step is to create an Amazon ECS Service.

Navigate to your ECS cluster, click the Services tab and then select Create

 ![](/Images/Img54.PNG)

Configure the ECS Service by selecting the following configuration:
 - Select the Task Definition that was configured earlier. 
 - Select the Platform version to be LATEST
 - Select the ECS cluster that was created earlier and provide name for your ECS Service (eg: wp-svc-appmig)
 - Enter the number of tasks to be 2

You can leave the rest as defaults and select Next step

 ![](/Images/Img55.PNG)

Configure Network
 - Select the Target VPC and select the private subnets for your web service.
 - Select ECS-Tasks-SG for your security group

 ![](/Images/Img56.PNG)

Select the Application Load Balancer and select the load balancer that was created earlier. Then click on add to load balancer to add the container name:port.

 ![](/Images/Img57.PNG)

 ![](/Images/Img58.PNG)

Leave the service discovery integration unchecked. 

 ![](/Images/Img59.PNG)

Set Auto Scailing. 

In the Auto scailing configuration, select the option “Configure Service Auto Scailing”. Enter the Minimum, Desired and Maximum number of tasks. 

 ![](/Images/Img60.PNG)

Select the Scaling policy to be Target Tracking, provide the Policy name, select the ECS service metric to be ALBRequestCountPerTarget and enter the Target value (eg: 300)

 ![](/Images/Img61.PNG)

Finally the ECS service has been created. 

Navigate to your Services under your ECS cluster, and you can see that wp-svc-appmig service is ACTIVE 

 ![](/Images/Img62.PNG)

You should also be able to see that your tasks are in a RUNNING state. 

To test your target website, navigate to your loadbalancer, grab the loadbalancer DNS and paste it on your web browser. In the screenshot below, you can see that Wordpress website loaded successfully which confirms that my web service is being hosted on Containers.  

 ![](/Images/Img63.PNG)

