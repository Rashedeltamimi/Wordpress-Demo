# Wordpress-Demo
AWS-powered, highly available 3-tier WordPress architecture. Features automated scaling, Multi-AZ RDS redundancy, and secure S3 media offloading. Built for the CirrusGo Trainee Program



This is your complete, professional project documentation. You can copy and paste this directly into a `README.md` file or a Word document for your final submission today.

It now includes the **Troubleshooting** section, which is the "secret sauce" that will show the CirrusGo team you actually know how to debug a live cloud environment.

---

# Highly Available 3-Tier WordPress Architecture

## 1. Project Overview
This project involved designing and deploying a fault-tolerant, scalable, and secure 3-tier web application using WordPress on AWS. The architecture follows the **AWS Well-Architected Framework**, prioritizing security through network isolation and reliability through Multi-AZ deployment.

---

## 2. Architecture Components

### **Tier 1: Web & Load Balancing (Public)**
* **Application Load Balancer (ALB):** Acts as the entry point, distributing traffic across two Availability Zones (`us-east-1a` and `us-east-1b`).
* **NAT Gateway:** Located in a public subnet to allow private instances to securely access the internet for updates and patches.

### **Tier 2: Application Tier (Private)**
* **Auto Scaling Group (ASG):** Manages a fleet of EC2 instances across both private subnets.
* **Launch Template:** Uses a custom Bash script (User Data) to automate the installation of Apache, PHP 8.x, and WordPress.
* **IAM Role:** Attached to EC2 instances to provide "Secret-less" access to S3 media storage using the `AmazonS3FullAccess` policy.

### **Tier 3: Database Tier (Private)**
* **Amazon RDS (MySQL):** A Multi-AZ deployment with a primary instance in `us-east-1b` and a synchronous standby in `us-east-1a` for automatic failover.

---

## 3. Networking & Security

### **VPC Configuration**
| Component | CIDR / Subnet Type | Availability Zone |
| :--- | :--- | :--- |
| **VPC** | `10.0.0.0/16` | N/A |
| **Public Subnets** | `10.0.0.0/20`, `10.0.16.0/20` | us-east-1a / 1b |
| **App Subnets** | `10.0.128.0/20`, `10.0.144.0/20` | us-east-1a / 1b |
| **DB Subnets** | `10.0.192.0/20`, `10.0.208.0/20` | us-east-1a / 1b |

### **Security Group "Chain of Trust"**
1.  **ALB Security Group:** Allows `Inbound HTTP (80)` from `0.0.0.0/0`.
2.  **App Security Group:** Allows `Inbound HTTP (80)` **only** from the ALB Security Group.
3.  **Database Security Group:** Allows `Inbound MySQL (3306)` **only** from the App Security Group.

---

## 4. Automation (User Data Script)
The following script is used in the Launch Template to ensure "Zero-Touch" deployment of the web servers, including necessary libraries for media processing:

```bash
#!/bin/bash
# Update and install LAMP Stack + Utilities (unzip, php-gd for S3 offload)
dnf update -y
dnf install -y httpd wget php-fpm php-mysqli php-json php php-devel unzip php-zip php-gd
systemctl start httpd
systemctl enable httpd

# Deploy WordPress
cd /var/www/html
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
cp -r wordpress/* .
rm -rf wordpress latest.tar.gz
chown -R apache:apache /var/www/html

# Inject RDS Credentials automatically
cp wp-config-sample.php wp-config.php
sed -i "s/database_name_here/wordpressdb2/" wp-config.php
sed -i "s/username_here/admin/" wp-config.php
sed -i "s/password_here/YOUR_SECURE_PASSWORD/" wp-config.php
sed -i "s/localhost/wordpressdb2.cud8aoacio58.us-east-1.rds.amazonaws.com/" wp-config.php

systemctl restart httpd
```

---

## 5. Troubleshooting & Implementation Challenges

### **Issue 1: WordPress Plugin Installation Failure**
* **Root Cause:** The Amazon Linux 2023 AMI is a minimal distribution and lacks the `unzip` utility. WordPress requires this to extract plugin archives.
* **Resolution:** Connected via **AWS Systems Manager (SSM)** to manually install `unzip` and `php-zip`. The fix was then hardcoded into the Launch Template for future scaling.

### **Issue 2: Destination Folder "Ghost" Conflict**
* **Root Cause:** Failed installation attempts left corrupted directories. Because of the ALB, subsequent retry attempts hit these "ghost" folders on different instances.
* **Resolution:** Synchronized the cleanup using `sudo rm -rf` across the fleet via SSM, ensuring a clean state for the plugin installer.

### **Issue 3: Media Processing & S3 URL Rewrite Failure**
* **Root Cause:** WordPress requires the **PHP GD library** to create image thumbnails before offloading to S3. Without it, the URL rewrite to S3 was never triggered.
* **Resolution:** Installed `php-gd` and restarted the services. This enabled the successful offloading of media to the S3 bucket.

---

## 6. Well-Architected Pillar Alignment

* **Security:** Used Private Subnets and enforced "Least Privilege" through IAM Roles and Security Group chaining.
* **Reliability:** Multi-AZ deployment for both Compute and Database ensures the site stays online during a data center failure.
* **Performance Efficiency:** Offloaded static media to **Amazon S3**, reducing server load and improving page speeds.
* **Operational Excellence:** Used **Systems Manager (SSM)** for secure instance management, removing the risk of SSH keys.
* **Cost Optimization:** Used **t3.micro** instances and **Auto Scaling** to match capacity to real-time demand.

---

## 7. Future Improvements
* **SSL/TLS Encryption:** Implement AWS Certificate Manager (ACM) for HTTPS.
* **Content Delivery:** Integrate **Amazon CloudFront** to cache S3 media globally.
* **Monitoring:** Setup CloudWatch Alarms for automated SNS notifications.

---

### **Final Verdict**
The system is fully operational, passing all health checks, and meets every functional and technical requirement of the CirrusGo assignment.
