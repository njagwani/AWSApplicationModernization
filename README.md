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

### Stage 3 - Create an RDS instance to setup your wordpress website.

Navigate to RDS service and then click on “Create Database”. Select the Engine type to be MySQL and select the image version to be MySQL 5.7.22. 

![](/Images/Img9.PNG)

Select the template type to be of Free tier

![](/Images/Img10.PNG)

Under Settings, provide the name for DB instance identifier (eg: wordpress-DB, Master username (eg: admin), Master password (eg:set your own password)

![](/Images/Img11.PNG)

Select your DB instance class size to be “Burstable classes” – db.t2.micro

![](/Images/Img12.PNG)

For your VPC, select your Target VPC (eg: Word Press Fargate Base Infrastructure VPC) and select your subnet group associate with your database (eg: wp-db-subnet-group, this db subnet group should have already been created when you created your cloudformation stack), Under existing security groups, select your RDS security group and provide an availability zone. 

![](/Images/Img13.PNG)

Provide the name of your Database (eg: wpdatabase) and save it as you will need this info to install wordpress. 

![](/Images/Img14.PNG)

Navigate to RDS instance security group to allow traffic from your ec2 instance to your RDS instance over TCP port 3306. Under inbound rules, select the Target Type = MySQL/Aurora and Source = Security Group of your EC2 web server instance. 

![](/Images/Img15.PNG)

Head over to your web server EC2 instance and grab the DNS and then paste in your web browser.

![](/Images/Img16.PNG)

The following page will appear, then select “Let’s go!”

![](/Images/Img17.PNG)



