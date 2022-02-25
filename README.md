# AWSApplicationModernization

Welcome to my AWS Application Modernization Project that will guide you on how to create a Wordpress website using EC2 instance and RDS instance. Once you have confirmed that your Wordpress website is running well, we will then replatform the web service to Containers with help of ECS Fargate Services.
### Pre-requisites for this project
- AWS Account
- IAM Roles for running EC2 Instances, ECS Tasks

Lets go ahead and get started. 

Stage 1 - Build AWS Core Infrastructure

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

Stage 2 - Create an EC2 instance to host your Wordpress Web Service

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






