# WordPress-AWS
Host WordPress Website using AWS Management Console and CLI

# Overview
This project demonstrates how to host a WordPress website on AWS, utilizing various AWS services to ensure high availability, security, and scalability. 

# Architecture
The architecture includes the following components:
- **VPC**: A Virtual Private Cloud (VPC) with both public and private subnets across two availability zones.
- **Internet Gateway**: Facilitates connectivity between VPC instances and the Internet.
- **Security Groups**: Acts as a network firewall.
- **Availability Zones**: Enhances system reliability and fault tolerance.
- **Public Subnets**: For infrastructure components like the NAT Gateway and Application Load Balancer.
- **Private Subnets**: Positions web servers (EC2 instances) for enhanced security.
- **EC2 Instance Connect Endpoint**: For secure connections to assets within both public and private subnets.
- **NAT Gateway**: Enables instances in private subnets to access the Internet.
- **EC2 Instances**: Hosts the website.
- **Application Load Balancer**: Distributes web traffic to an Auto Scaling Group of EC2 instances.
- **Auto Scaling Group**: Manages EC2 instances automatically to ensure website availability.
- **SNS**: Alerts about activities within the Auto Scaling Group.
- **EFS**: Provides a shared file system.
- **RDS**: Manages the database.
- **Route 53**: Manages the domain name and DNS records.
- **Certificate Manager**: Secures application communications.
  
# Prerequisites
- AWS account with permissions to create and manage EC2 instances, VPC, and other resources.
- A registered domain name.

# Setup Instructions
1. VPC Configuration
Create a VPC with public and private subnets across two availability zones.
2. Internet Gateway
Attach an Internet Gateway to the VPC.
3. Security Groups
Configure security groups to control inbound and outbound traffic.
4. EC2 Instances in Private Subnets
Launch EC2 instances in the private subnets for enhanced security.
5. NAT Gateway
Deploy a NAT Gateway in the public subnets to allow private subnet instances to access the Internet.
6. Application Load Balancer
Set up an Application Load Balancer in the public subnets to distribute incoming traffic.
7. Auto Scaling Group
Configure an Auto Scaling Group to manage the EC2 instances, ensuring high availability and scalability.
8. EFS Configuration
Create an EFS and mount it to the EC2 instances.
9. RDS Configuration
Set up an RDS instance for the WordPress database.
10. Certificate Manager
Use AWS Certificate Manager to secure communications.
11. Route 53
Register your domain and configure DNS records using Route 53.

# Lessons Learned
This project provided my first hands-on experience with DevOps and AWS for provisioning infrastructure. It offered valuable insights into cloud architecture and networking. While there were challenges, each one provided an opportunity for growth and improvement.

## Example Issues and Resolutions:
Issue: Images Not Displaying on WordPress
- Attempted reinstalling, but the issue persisted.
- Inspected the problem using Chrome's error messages.
- Researched solutions via AWS documentation, Reddit, and Stack Overflow.
- Ultimately, decided to restart the project from scratch but couldn't pinpoint the exact mistake.
  
Issue: Bugs When Using vi Editor
- Experienced issues with line duplication, despite previous experience with vim.
- Determined it was a command line error, resolved by reconnecting to the EC2 instance.

Issue: Unable to Terminate EC2 Instance Due to Network Interface in Use
- Spent considerable time identifying the service causing the issue.
- Used Amazon CloudTrail logs to trace resource creation.
- Discovered it was an RDS proxy service remaining active even after terminating the main RDS instance.

# Scripts to Install WordPress
```bash
# create to root user
sudo su

# update the software packages on the ec2 instance 
sudo yum update -y

# create an html directory 
sudo mkdir -p /var/www/html

# environment variable
EFS_DNS_NAME=fs-08a13a38cbbcc0524.efs.ap-southeast-1.amazonaws.com

# mount the efs to the html directory 
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport "$EFS_DNS_NAME":/ /var/www/html

# install the apache web server, enable it to start on boot, and then start the server immediately
sudo yum install -y httpd
sudo systemctl enable httpd 
sudo systemctl start httpd

# install php 8 along with several necessary extensions for wordpress to run
sudo dnf install -y \
php \
php-cli \
php-cgi \
php-curl \
php-mbstring \
php-gd \
php-mysqlnd \
php-gettext \
php-json \
php-xml \
php-fpm \
php-intl \
php-zip \
php-bcmath \
php-ctype \
php-fileinfo \
php-openssl \
php-pdo \
php-tokenizer

# install the mysql version 8 community repository
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm 

# install the mysql server
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm 
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf repolist enabled | grep "mysql.*-community.*"
sudo dnf install -y mysql-community-server 

# start and enable the mysql server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# set permissions
sudo usermod -a -G apache ec2-user
sudo chown -R ec2-user:apache /var/www
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
sudo find /var/www -type f -exec sudo chmod 0664 {} \;
chown apache:apache -R /var/www/html 

# download wordpress files
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo cp -r wordpress/* /var/www/html/

# [OWN] enter html folder before creating wp config file
cd var/www/html/

# create the wp-config.php file
sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php

# edit the wp-config.php file
sudo vi /var/www/html/wp-config.php

# restart the webserver
sudo service httpd restart
```

# Launch Template Scripts to Install WordPress Using EC2 Auto Scaling
```bash
#!/bin/bash

# update the software packages on the ec2 instance 
sudo yum update -y

# install the apache web server, enable it to start on boot, and then start the server immediately
sudo yum install -y httpd
sudo systemctl enable httpd 
sudo systemctl start httpd

# install php 8 along with several necessary extensions for wordpress to run
sudo dnf install -y \
php \
php-cli \
php-cgi \
php-curl \
php-mbstring \
php-gd \
php-mysqlnd \
php-gettext \
php-json \
php-xml \
php-fpm \
php-intl \
php-zip \
php-bcmath \
php-ctype \
php-fileinfo \
php-openssl \
php-pdo \
php-tokenizer

# install the mysql version 8 community repository
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm 

# install the mysql server
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm 
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf repolist enabled | grep "mysql.*-community.*"
sudo dnf install -y mysql-community-server 

# start and enable the mysql server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# environment variable
EFS_DNS_NAME=fs-08a13a38cbbcc0524.efs.ap-southeast-1.amazonaws.com

# mount the efs to the html directory 
echo "$EFS_DNS_NAME:/ /var/www/html nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 0 0" >> /etc/fstab
mount -a

# set permissions
chown apache:apache -R /var/www/html

# restart the webserver
sudo service httpd restart
``
