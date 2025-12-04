# 📘 AWS 3-Tier Architecture Deployment Guide

This guide provides a **step-by-step approach** to deploy a scalable, highly available AWS 3-Tier Architecture using a **bottom-up methodology**.

---

## 🛠 Services Used
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

## 🔽 Why Bottom-Up Approach?

| Benefit | Explanation |
|---------|-------------|
| **Dependency Alignment** | Core components (VPC, subnets, DB) must exist before app/web tiers. |
| **Network Foundation First** | Routing, IP addressing, NAT/IGW, and security groups are prerequisites. |
| **Secure Layering** | DB & App layers in private subnets; Web exposed only after backend security. |
| **Error Reduction** | Issues in networking/DB identified early, preventing failures in upper layers. |
| **Stable Infrastructure** | Each tier validated before moving to the next. |
| **Scalability Readiness** | Auto Scaling & Load Balancing require stable DB/App layers. |

---

## 🏗 Bottom-Up Deployment Sequence
1. **Networking Layer (VPC & Subnets)**
2. **Database Layer (RDS / DB Servers)**
3. **Application Layer (App EC2 / Auto Scaling)**
4. **Web Layer (Web EC2 / Internet-Facing Load Balancer)**
5. **Monitoring & Notifications (SNS & CloudWatch)**

---

## 1️⃣ Networking Layer (VPC & Subnets)

- **Create VPC** → CIDR: `10.0.0.0/16`
- **Subnets** (2 AZs):
  - PublicSubnet-A, PublicSubnet-B
  - WebSubnet-A, WebSubnet-B
  - AppSubnet-A, AppSubnet-B
  - DBSubnet-A, DBSubnet-B
- **Internet Gateway** → Attach to VPC
- **Route Tables**:
  - Rename default → `Private-RT`
  - Create `Public-RT` → attach IGW
  - Associate public subnets to `Public-RT`

---

## 2️⃣ Database Layer (RDS + Temporary Dev2 EC2)

- **DB Subnet Group** → include all private DB subnets
- **Launch RDS Instance**:
  - Engine: MySQL  
  - Availability: Single-AZ  
  - Storage: GP SSD  
  - Enhanced Monitoring enabled  
- **Temporary Dev2 EC2** (for DB setup only):
  - Subnet: DB Subnet-A (Private)  
  - SG: SSH (22), MySQL (3306)  
  - Install MariaDB client:
    ```bash
    sudo yum update -y
    sudo yum install mariadb105-server -y
    ```
  - Connect to RDS:
    ```bash
    mysql -u root -p -h <RDS_ENDPOINT>
    ```
  - Create DB & table:
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
- **Terminate Dev2 EC2** after DB setup ✅

---

## 3️⃣ Application Layer (Dev1 EC2 → AMI → Auto Scaling Group)

- **Temporary Dev1 EC2**:
  - Subnet: App Subnet-A (Private)  
  - Public IP: Enabled (for SSH only)  
  - SG: SSH (22), HTTP (80)  
- **Install PHP + Nginx**:
  ```bash
  sudo yum update -y
  sudo yum install php nginx -y
  sudo systemctl start nginx php-fpm
  sudo systemctl enable nginx php-fpm
  sudo yum install php8.4-mysqlnd.x86_64 -y
  sudo systemctl restart php-fpm
  sudo systemctl restart nginx
  ```
- Test Files (index.html, index.php, reg.php) → verify Nginx, PHP, DB connectivity
- Clean Up → keep only reg.php
- Create AMI → AppServer-AMI
---
## 4️⃣ Internal Load Balancer:
- Scheme: Internal
- Subnets: App Subnet-A & B
- Target Group: HTTP:80, Health Check /

- Auto Scaling Group (ASG):
   - Launch Template → AppServer-AMI
   - Subnets: App Subnet-A & B
   - Attach to Internal ALB
   - Scaling Policy → Target Tracking 50% CPU
   - Min: 1, Desired: 2, Max: 4
   - Monitoring → CloudWatch enabled
---
## 5️⃣ Web Layer (Web EC2 → AMI → Internet-Facing Load Balancer)
- Web EC2:
   - Subnet: Web Subnet-A
   - SG: SSH (22), HTTP (80)
   - Install Nginx:
      ```bash
       sudo yum install nginx -y
       sudo systemctl start nginx
       sudo systemctl enable nginx
      ```
   - Add HTML/CSS files → /usr/share/nginx/html
     
- Configure Proxy:
update /etc/nginx/nginx.conf:
```nginx
server {
    listen 80;
    server_name _;
    location / {
        proxy_pass http://<Internal-ALB-DNS>;
    }
}
```
Restart:
``` bash
sudo systemctl restart nginx
```
- Create AMI → WebServer-AMI -> Delete original instance
---
## 6️⃣ Internet-Facing Load Balancer:

- Scheme: **Internet-Facing**
- Subnets: Web Subnet-A & Web Subnet-B
- Target Group: HTTP:80, Health Check `/`
- Listener: HTTP:80 → Forward to Web Target Group

- Auto Scaling Group (ASG):
   - Launch Template → WebServer-AMI
   - Subnets: Web Subnet-A & B
   - Attach to Internet-Facing ALB
   - Scaling Policy → Target Tracking 50% CPU
   - Min: 1, Desired: 2, Max: 4
   - Monitoring → CloudWatch enabled

---
## 7️⃣ Notifications (SNS Topic)
Create SNS topic for:
  - ASG scale-out events
  - Health check failures
  - subscribe for ASG scaling events
---
## 8️⃣ Monitoring (CloudWatch)
Monitor:
  - EC2 CPU / Memory
  - ASG Scaling Events
  - RDS Performance
  - ALB Target Health
---
## 9️⃣ Best Practices

- Keep **App** & **DB** tiers in **private subnets**.  
- Only expose the **Web tier** to the Internet.  
- Tag all resources for easy management and cost allocation.  
- Use **Multi-AZ RDS** in production for high availability.  
- Apply **least-privileged Security Groups**.  
- Use SSM for server management instead of SSH in production.  

---

## Final Outcome

After completing all steps, you will have a fully **scalable, secure, and highly available 3-Tier AWS architecture**:

- **Application Tier** → Auto Scaling Group + Internal ALB  
- **Web Tier** → Internet-facing ALB  
- **Database Tier** → Managed RDS instance  
- **Notifications** → SNS for scaling & health events  
- **Monitoring** → CloudWatch metrics and alarms  

All tiers follow a **bottom-up deployment**, ensuring dependencies, security, and scalability are properly handled.




