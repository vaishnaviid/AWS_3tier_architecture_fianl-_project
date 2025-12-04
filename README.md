# AWS 3-Tier Architecture Deployment 

This guide provides a **step-by-step approach** to deploy a scalable, highly available AWS 3-Tier Architecture using a **bottom-up methodology**.

---

## Services Used
- **VPC**  
- **RDS (MySQL)**  
- **EC2 Instances (App, DB, Web)**  
- **Load Balancer (ALB/NLB)**  
- **Auto Scaling Group**  
- **Target Group**  
- **Security Groups**  
- **AMI (Application/Web images)**  
- **SNS**  
- **CloudWatch**  

---

## Why Bottom-Up Approach?

1. **Dependency Alignment:**  
   - Core components like VPC, subnets, and the database must be created first because the application and web tiers depend on them.

2. **Network Foundation First:**  
   - Routing, IP addressing, NAT/IGW setup, and security groups must exist early to enable correct communication across all tiers.

3. **Secure Layering:**  
   - DB and App layers are built in private subnets first, ensuring the Web layer is exposed to the internet only after backend security is in place.

4. **Error Reduction:**  
   - Building bottom-up helps identify issues in networking or database layers early, preventing failures in upper layers.

5. **Stable Infrastructure:**  
   - Each tier is validated before moving to the next, ensuring consistent and predictable deployment.

6. **Scalability Readiness:**  
   - Auto Scaling and Load Balancing require stable app and DB layers to function effectively, making bottom-up the logical sequence.
---

## Bottom-Up Deployment Approach
The architecture is deployed **from the foundational layer to the top layer**:

1. **Networking (VPC & Subnets)**
2. **Database Layer (RDS / DB Servers)**
3. **Application Layer (App EC2 / Auto Scaling)**
4. **Web Layer (Web EC2 / Internet-Facing Load Balancer)**
5. **Monitoring & Notifications (SNS & CloudWatch)**

---

## 1. Networking Layer (VPC & Subnets)

1. **Create VPC**
   - CIDR range example: `10.0.0.0/16`.

2. **Create Subnets**
   - 2 Availability Zones (AZ)
   - 1 Public subnet per AZ  
   - 3 Private subnets per AZ for Web, App, DB  
   - Example:  
     ```
     PublicSubnet-A, PublicSubnet-B
     WebSubnet-A, WebSubnet-B
     AppSubnet-A, AppSubnet-B
     DBSubnet-A, DBSubnet-B
     ```

3. **Internet Gateway**
   - Create and attach to VPC.

4. **Route Tables**
   - Default route table → rename to `Private-RT`  
   - Create `Public-RT` → attach IG  
   - Associate public subnets to `Public-RT`.

---

## 2. Database Layer (RDS / Dev2 EC2)

1. **Create DB Subnet Group**
   - Include all private DB subnets.

2. **Launch RDS Instance**
   - Engine: MySQL  
   - Availability: Single-AZ  
   - Username: `root`  
   - Password: `pass123`  
   - Storage: General Purpose SSD  
   - Enable Enhanced Monitoring  

3. Temporary Dev2 EC2 (for Initial DB Setup Only)

     This EC2 instance is created **only for the purpose of initializing the RDS database** and testing the connection. It is not part of the final architecture.

4.  **Launch EC2 Instance**
   - Subnet: **DB Subnet-A (Private Subnet)**
   - Security Group:  
     - SSH (22) – for access  
     - MySQL (3306) – to connect to RDS

5. **Install MariaDB Client**
   ```bash
   sudo yum update -y
   sudo yum install mariadb105-server -y
6. **Connect to the RDS Instance**
   ```bash
   mysql -u root -p -h <RDS_ENDPOINT>
   ```
7. Create Database and Table

   ```sql
   CREATE DATABASE registration_db;
   USE registration_db;

   CREATE TABLE users (
      id INT AUTO_INCREMENT PRIMARY KEY,
      fullname VARCHAR(200) NOT NULL,
      email VARCHAR(200) NOT NULL UNIQUE,
      password_hash VARCHAR(255) NOT NULL,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );
   ```
### Purpose of This Step
  - Initialize the database.
  - Test RDS connectivity.
  - Retrieve and validate the RDS endpoint for use in the PHP application.
  - Delete the Temporary Instance
  - After DB creation and connectivity testing, this EC2 instance is no longer required.
  It is safe to terminate to reduce cost and maintain clean architecture.

| Note: This temporary server is only for database setup. The final architecture does not include a separate DB EC2 instance—RDS handles the database layer.

---

## 3. Application Layer (Dev1 EC2 & Auto Scaling Group)

This layer contains the **PHP application servers** that interact with the RDS database.  
We create a temporary EC2 instance to configure the application, test DB connectivity, then convert it into an AMI for Auto Scaling.

---

1. **Launch Dev1 EC2 (Temporary App Server)**

- Subnet: **App Subnet-A (Private Subnet)**  
- Public IP: **Enabled** (only for initial SSH access)  
- Security Group:
  - SSH (22)
  - HTTP (80)

---

2. **Install PHP, Nginx & Required Extensions**

SSH into the instance:

```bash
sudo yum update -y
sudo yum install php nginx -y
sudo systemctl start nginx
sudo systemctl start php-fpm
sudo systemctl enable nginx
sudo systemctl enable php-fpm

sudo yum install php8.4-mysqlnd.x86_64 -y
php -v

sudo systemctl restart php-fpm
sudo systemctl restart nginx
```

3. **Add Temporary Test Files (For Verification Only)**

Navigate to web root:
```bash
cd /usr/share/nginx/html
```
Create:

index.html → for checking nginx

index.php → for checking PHP

reg.php → to test DB connection and form functionality

These files are only for initial testing to verify:

Nginx is running

PHP is processing requests correctly

RDS connection works

Step 4: Verify Application + DB Integration
Ensure PHP file successfully connects to RDS using endpoint

Confirm data insertion works

Verify nginx serves responses correctly

Step 5: Clean Up Test Files
After testing:

bash
Copy code
sudo rm -f index.html
sudo rm -f *.css
Keep only:

✔ reg.php (or your main PHP app file)

This prevents exposing unnecessary test files in production.

Step 6: Create AMI for Auto Scaling
Once everything works:

Stop creating/modifying files

Go to EC2 console

Select instance → Create Image (AMI)

Name: AppServer-AMI

This AMI will be used by the Auto Scaling Group.

Step 7: Create Internal Load Balancer
Create an Internal Application Load Balancer:

Scheme: Internal

Subnets: App Subnet-A & App Subnet-B

Target Group:

Target Type: Instance

Protocol: HTTP:80

Health Check Path: /

Attach target group but instances will auto-register through the ASG.

Step 8: Create Auto Scaling Group (ASG)
Launch Template

Use the App AMI created above

Attach App Security Group

User Data (optional): restart nginx on boot

bash
Copy code
#!/bin/bash
systemctl restart nginx
systemctl restart php-fpm
ASG Settings

AZ Distribution: Balanced Across AZs

VPC: 3-tier VPC

Subnets: App Subnet-A & App Subnet-B

Attach to Internal Load Balancer

Choose the previously created ALB target group

Health Checks

Type: ELB + EC2

Grace Period: 60 seconds

Scaling Policy

Target Tracking: 50% CPU Utilization

Minimum: 1 instance

Desired: 2 instances

Maximum: 4 instances

Instance Warm-up: 60 seconds

Monitoring

Enable Enhanced Monitoring & CloudWatch metrics

Why This Temporary Setup?
The Dev1 EC2 is only used to build, test, and verify the application code.

After creating the AMI, this instance is terminated, and the ASG takes over to maintain,
scale, and self-heal the app tier.

This ensures:

Consistency across all application servers

Automatic scaling

Seamless load balancing

No manual configuration on new servers

Result:
The Application Tier becomes fully automated, scalable, secure, and connected to the RDS DB through a properly configured internal load balancer + ASG system.

---

## 4. Web Layer (Web EC2 & Internet-Facing Load Balancer)

1. **Launch Web EC2 (Web Subnet-A)**
   - Security Group: SSH (22), HTTP (80)  
   - Install Nginx:
     ```bash
     sudo yum update -y
     sudo yum install nginx -y
     sudo service nginx start
     sudo systemctl enable nginx
     ```
   - Add HTML/CSS files to `/usr/share/nginx/html`.

2. **Configure Nginx to Proxy Requests**
   - Update `/etc/nginx/nginx.conf`:
     ```nginx
     server {
         listen 80;
         server_name _;
         location / {
             proxy_pass http://<Internal-ALB-DNS>;
         }
     }
     ```
3. **Create AMI** and delete instance.

4. **Create Internet-Facing Load Balancer**
   - Target: Web AMI instances

---

## 5. SNS Topic

- Create SNS topic for notifications (e.g., ASG scaling events).

---

## 6. Monitoring (CloudWatch)

- Enable CloudWatch metrics for:
  - EC2 instances
  - Auto Scaling events
  - RDS performance

---

## Best Practices

- Use **private subnets** for App & DB layers.  
- Only expose Web layer to the Internet.  
- Tag all resources for easy management.  
- Use Multi-AZ RDS for production.  
- Keep Security Groups least-privileged.  

---

> This README follows a **bottom-up approach**, starting from foundational networking and database layers, moving up to application and web layers, ensuring **dependency order, security, and scalability**.
