### Reference project: https://github.com/ojoimmanuel/ACS-project-config.git
### ami used: ami-03951dc3553ee499f (available in community ami)  

### project 15 infrastructure in an image
![infrac](./images/ref%20image.png)  

## Starting Off Your AWS Cloud Project
There are few requirements that must be met before you begin:

1. Properly configure your AWS account and Organization Unit
    - Create an AWS Master account. (Also known as Root Account)
    - Within the Root account, create a sub-account and name it DevOps. (You will need another email address to complete this)
    - Within the Root account, create an AWS Organization Unit (OU). Name it Dev. (We will launch Dev resources in there)
    - Move the DevOps account into the Dev OU.
    - Login to the newly created AWS account using the new email address.
2. Create a hosted zone in AWS, and map it to your domain from Freenom.  

### SET UP A VIRTUAL PRIVATE NETWORK (VPC)
Always make reference to the architectural diagram and ensure that your configuration is aligned with it.

1. Create a VPC
![vpc](./images/01-vpc.png)  

2. Create subnets as shown in the architecture
![subnets](./images/02-subnets.png)  

3. Create a route table and associate it with public subnets

4. Create a route table and associate it with private subnets
![rtb](./images/03-routetables.png)  

5. Create an Internet Gateway
![igw](./images/04-igw.png)  

6. Edit a route in public route table, and associate it with the Internet Gateway. (This is what allows a public subnet to be accisble from the Internet)
![igw-rtb](./images/05-rtb-igw.png)  

7. Create 3 Elastic IPs  

8. Create a Nat Gateway and assign one of the Elastic IPs (*The other 2 will be used by Bastion hosts)
![nat](./images/06-nat-gw.png) 
 
9. Create a Security Group for:

- Nginx Servers: Access to Nginx should only be allowed from a Application Load balancer (ALB). At this point, we have not created a load balancer, therefore we will update the rules later. For now, just create it and put some dummy records as a place holder.  

- Bastion Servers: Access to the Bastion servers should be allowed only from workstations that need to SSH into the bastion servers. Hence, you can use your workstation public IP address. To get this information, simply go to your terminal and type curl www.canhazip.com  

- Application Load Balancer: ALB will be available from the Internet  

- Webservers: Access to Webservers should only be allowed from the Nginx servers. Since we do not have the servers created yet, just put some dummy records as a place holder, we will update it later.  

- Data Layer: Access to the Data layer, which is comprised of Amazon Relational Database Service (RDS) and Amazon Elastic File System (EFS) must be carefully desinged – only webservers should be able to connect to RDS, while Nginx and Webservers will have access to EFS Mountpoint.  

![SG](./images/07-sg.png)  

### Proceed With Compute Resources
You will need to set up and configure compute resources inside your VPC. The recources related to compute are:

- EC2 Instances
- Launch Templates
- Target Groups
- Autoscaling Groups
- TLS Certificates
- Application Load Balancers (ALB)  

### Set Up Compute Resources for Nginx
#### Provision EC2 Instances for Nginx
1. Create an EC2 Instance based on CentOS Amazon Machine Image (AMI) in any 2 Availability Zones (AZ) in any AWS Region (it is recommended to use the Region that is closest to your customers). Use EC2 instance of T2 family (e.g. t2.micro or similar)

2. Ensure that it has the following software installed:

- python
- ntp
- net-tools
- vim
- wget
- telnet
- epel-release
- htop

#### hint: refer to Installation.md in https://github.com/ojoimmanuel/ACS-project-config.git for server installation codes

3. Create an AMI out of the EC2 instance  

![ami](./images/08-ami.png)  

### Prepare Launch Template For Nginx (One Per Subnet)
1. Make use of the AMI to set up a launch template
2. Ensure the Instances are launched into a public subnet
3. Assign appropriate security group
4. Configure Userdata to update yum package repository and install nginx
#### hint: refer to https://github.com/ojoimmanuel/ACS-project-config.git for nginx user data 

![userdata](./images/10-nginx-userdata.png)  

![nginx](./images/09-nginx-template.png)  

### Configure Target Groups
1. Select Instances as the target type
2. Ensure the protocol HTTPS on secure TLS port 443
3. Ensure that the health check path is /healthstatus
4. Register Nginx Instances as targets
5. Ensure that health check passes for the target group
  
 ![target-groups](./images/11-target-groups.png)  

### Configure Autoscaling For Nginx
1. Select the right launch template
2. Select the VPC
3. Select both public subnets
4. Enable Application Load Balancer for the AutoScalingGroup (ASG)
5. Select the target group you created before
6. Ensure that you have health checks for both EC2 and ALB
7. The desired capacity is 2
8. Minimum capacity is 2
9. Maximum capacity is 4
10. Set scale out if CPU utilization reaches 90%
11. Ensure there is an SNS topic to send scaling notifications

![asg](./images/12-auto-sg.png)  

### Set Up Compute Resources for Bastion
Provision the EC2 Instances for Bastion
1. Create an EC2 Instance based on CentOS Amazon Machine Image (AMI) per each Availability Zone in the same Region and same AZ where you created Nginx server
2. Ensure that it has the following software installed

- python
- ntp
- net-tools
- vim
- wget
- telnet
- epel-release
- htop

#### refer to reference project for installation codes  

3. Associate an Elastic IP with each of the Bastion EC2 Instances  
4. Create an AMI out of the EC2 instance

![ami](./images/08-ami.png)  

### Prepare Launch Template For Bastion (One per subnet)
1. Make use of the AMI to set up a launch template
2. Ensure the Instances are launched into a public subnet
3. Assign appropriate security group
4. Configure Userdata to update yum package repository and install Ansible and git
 

### refer to ref project for userdata
![userdata](./images/13-bastion-userdata.png)

![template](./images/14-launch-template.png)


### Configure Target Groups
1. Select Instances as the target type
2. Ensure the protocol is TCP on port 22
3. Register Bastion Instances as targets
4. Ensure that health check passes for the target group

![target-gp](./images/11-target-groups.png)

### Configure Autoscaling For Bastion
1. Select the right launch template
2. Select the VPC
3. Select both public subnets
4. Enable Application Load Balancer for the AutoScalingGroup (ASG)
5. Select the target group you created before
6. Ensure that you have health checks for both EC2 and ALB
7. The desired capacity is 2
8. Minimum capacity is 2
9. Maximum capacity is 4
10. Set scale out if CPU utilization reaches 90%
11. Ensure there is an SNS topic to send scaling notifications

![asg](./images/12-auto-sg.png)

### Set Up Compute Resources for Webservers
Provision the EC2 Instances for Webservers
Now, you will need to create 2 separate launch templates for both the WordPress and Tooling websites

1. Create an EC2 Instance (Centos) each for WordPress and Tooling websites per Availability Zone (in the same Region).

2. Ensure that it has the following software installed

- python
- ntp
- net-tools
- vim
- wget
- telnet
- epel-release
- htop
- php

**hint** edit key name in file after apache cert installation 
```
vi /etc/httpd/conf.d/ssl.conf
```
![editkey](./images/change-key-name.png)

3. Create an AMI out of the EC2 instance

![ami](./images/08-ami.png)

### Prepare Launch Template For Webservers (One per subnet)
1. Make use of the AMI to set up a launch template
2. Ensure the Instances are launched into a public subnet
3. Assign appropriate security group
4. Configure Userdata to update yum package repository and install wordpress (Only required on the WordPress launch template)

![template](./images/14-launch-template.png)

## TLS Certificates From Amazon Certificate Manager (ACM)
You will need TLS certificates to handle secured connectivity to your Application Load Balancers (ALB).

1. Navigate to AWS ACM
2. Request a public wildcard certificate for the domain name you registered in Freenom
3. Use DNS to validate the domain name
4. Tag the resource

![cert](./images/15-cert.png)

## CONFIGURE APPLICATION LOAD BALANCER (ALB)
#### Application Load Balancer To Route Traffic To NGINX
Nginx EC2 Instances will have configurations that accepts incoming traffic only from Load Balancers. No request should go directly to Nginx servers. With this kind of setup, we will benefit from intelligent routing of requests from the ALB to Nginx servers across the 2 Availability Zones. We will also be able to offload SSL/TLS certificates on the ALB instead of Nginx. Therefore, Nginx will be able to perform faster since it will not require extra compute resources to valifate certificates for every request.

1. Create an Internet facing ALB
2. Ensure that it listens on HTTPS protocol (TCP port 443)
3. Ensure the ALB is created within the appropriate VPC | AZ | Subnets
4. Choose the Certificate from ACM
5. Select Security Group
6. Select Nginx Instances as the target group

![loadbalancer](./images/16-loadbalancer.png)

### Application Load Balancer To Route Traffic To Web Servers

Since the webservers are configured for auto-scaling, there is going to be a problem if servers get dynamically scalled out or in. Nginx will not know about the new IP addresses, or the ones that get removed. Hence, Nginx will not know where to direct the traffic.

To solve this problem, we must use a load balancer. But this time, it will be an internal load balancer. Not Internet facing since the webservers are within a private subnet, and we do not want direct access to them.

1. Create an Internal ALB
2. Ensure that it listens on HTTPS protocol (TCP port 443)
3. Ensure the ALB is created within the appropriate VPC | AZ | Subnets
4. Choose the Certificate from ACM
5. Select Security Group
6. Select webserver Instances as the target group
7. Ensure that health check passes for the target group
NOTE: This process must be repeated for both WordPress and Tooling websites.

![alb](./images/16-loadbalancer.png)  

## Setup EFS 

Amazon Elastic File System (Amazon EFS) provides a simple, scalable, fully managed elastic Network File System (NFS) for use with AWS Cloud services and on-premises resources. In this project, we will utulize EFS service and mount filesystems on both Nginx and Webservers to store data.

1. Create an EFS filesystem
2. Create an EFS mount target per AZ in the VPC, associate it with both subnets dedicated for data layer
3. Associate the Security groups created earlier for data layer.
4. Create an EFS access point for tooling and wordpress. (Give it a name and leave all other settings as default)

**hint** EFS should attach to subnets that contain the EC2 that need to mount to EFS  

![efs](./images/20-efs.png)  

![efs-ap](./images/21-efs-ap.png)

## Setup RDS
**Pre-requisite**: Create a KMS key from Key Management Service (KMS) to be used to encrypt the database instance.

Amazon Relational Database Service (Amazon RDS) is a managed distributed relational database service by Amazon Web Services. This web service running in the cloud designed to simplify setup, operations, maintenans & scaling of relational databases. Without RDS, Database Administrators (DBA) have more work to do, due to RDS, some DBAs have become jobless

To ensure that yout databases are highly available and also have failover support in case one availability zone fails, we will configure a multi-AZ set up of RDS MySQL database instance. In our case, since we are only using 2 AZs, we can only failover to one, but the same concept applies to 3 Availability Zones. We will not consider possible failure of the whole Region, but for this AWS also has a solution – this is a more advanced concept that will be discussed in following projects.

To configure RDS, follow steps below:

1. Create a subnet group and add 2 private subnets (data Layer)

![subnet](./images/17-rds-subnet.png)

2. Create an RDS Instance for mysql 8.*.*  

3. To satisfy our architectural diagram, you will need to select either Dev/Test or Production Sample Template. But to minimize AWS cost, you can select the Do not create a standby instance option under Availability & durability sample template (The production template will enable Multi-AZ deployment)

4. Configure other settings accordingly (For test purposes, most of the default settings are good to go). In the real world, you will need to size the database appropriately. You will need to get some information about the usage. If it is a highly transactional database that grows at 10GB weekly, you must bear that in mind while configuring the initial storage allocation, storage autoscaling, and maximum storage threshold.

5. Configure VPC and security (ensure the database is not available from the Internet)

6. Configure backups and retention

7. Encrypt the database using the KMS key created earlier

8. Enable CloudWatch monitoring and export Error and Slow Query logs (for production, also include Audit)

**Note** This service is an expensinve one. Ensure to review the monthly cost before creating. (DO NOT LEAVE ANY SERVICE RUNNING FOR LONG)

![rds](./images/18-rds.png)

**hint** Create toolingdb and wordpressdb by connecting to rds endpoint through Bastion

![dbcreate](./images/create%20db%20thru%20bastion.png)

## Configuring DNS with Route53
Earlier in this project you registered a free domain with Freenom and configured a hosted zone in Route53. But that is not all that needs to be done as far as DNS configuration is concerned.

You need to ensure that the main domain for the WordPress website can be reached, and the subdomain for Tooling website can also be reached using a browser.

Create other records such as CNAME, alias and A records.

**NOTE**: You can use either CNAME or alias records to achieve the same thing. But alias record has better functionality because it is a faster to resolve DNS record, and can coexist with other records on that name. Read here to get to know more about the differences.

- Create an alias record for the root domain and direct its traffic to the ALB DNS name.
- Create an alias record for tooling.<yourdomain>.com and direct its traffic to the ALB DNS name.
Congratulations!

![route53](./images/19-route53.png)

Result

![wp](./images/wpfinalresult.png)

![tooling](./images/toolingfinalresult.png)