# WordPress Hosting on AWS EC2 with RDS (LAMP Stack)

## Project Overview
This project demonstrates deploying a production-style WordPress website on AWS using a decoupled architecture — the web server (Apache/PHP) 
runs on an EC2 instance while the database (MySQL) runs on a separate, private Amazon RDS instance. The setup also includes a High Availability 
extension using an Application Load Balancer (ALB), Auto Scaling Group (ASG).

**Stack Used:** Linux (Amazon Linux 2023) + Apache + MySQL (RDS) + PHP → **LAMP**

---

## Architecture Diagram (Conceptual)

```
                 ┌─────────────────────┐
   Internet ───▶ │   EC2 (Amazon Linux) │
                 │  Apache + PHP + WP   │
                 └──────────┬──────────┘
                            │  (Private connection, port 3306)
                            ▼
                 ┌─────────────────────┐
                 │   Amazon RDS MySQL   │
                 │  (Public Access: No) │
                 └─────────────────────┘
```

---

## Key AWS Services Used
| Service | Purpose |
|---|---|
| Amazon EC2 | Hosts Apache web server + WordPress application |
| Amazon RDS (MySQL) | Managed, private database backend |
| Security Groups | Controls network access (HTTP, SSH, MySQL) |
| Application Load Balancer (ALB) | Distributes traffic across multiple EC2 instances |
| Auto Scaling Group (ASG) | Maintains desired instance count using a Launch Template |
| AMI | Reusable image of the configured WordPress server |

---

## Step 1: Launch EC2 Instance
- Launched an **Amazon Linux 2023** EC2 instance.
- Configured a Security Group to allow:
  - HTTP (port 80) — from IPV4
  - SSH (port 22) — for admin access

## Step 2: Create RDS MySQL Database
- Created an **Amazon RDS MySQL** instance.
- **Public Access: No** — kept the database private, accessible only from within the VPC.
- Updated the RDS Security Group to allow **MySQL/Aurora (port 3306)** traffic from the EC2 instance's security group (or as required).

## Step 3: Install MySQL Client on EC2
Connect to the EC2 instance via SSH, then install the MySQL client:
```bash
dnf install -y mariadb105
```

Export the RDS endpoint as an environment variable for convenience:
```bash
export MYSQL_HOST=database-1.cbo6mcsyoalv.ap-south-1.rds.amazonaws.com
```

## Step 4: Connect to RDS and Create Database
```bash
mysql -h $MYSQL_HOST -P 3306 -u admin -p
```

Inside the MySQL shell:
```sql
CREATE DATABASE wordpress;
CREATE USER 'wpuser' IDENTIFIED BY 'admin123456789!';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'%';
FLUSH PRIVILEGES;
EXIT;
```
> ⚠️ **Security Note:** Use a strong, unique password in production. Avoid hardcoding credentials directly in scripts or documentation.

## Step 5: Install and Configure Apache
```bash
sudo yum install -y httpd
sudo service httpd start
```

## Step 6: Download and Extract WordPress
```bash
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
cd wordpress
```

## Step 7: Configure `wp-config.php`
Create the config file from the sample:
```bash
cp wp-config-sample.php wp-config.php
vi wp-config.php
```

Update the database connection details:
```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'admin123456789' );
define( 'DB_HOST', 'database-1.cbo6mcsyoalv.ap-south-1.rds.amazonaws.com' );
```

Generate unique security keys and salts from:
👉 https://api.wordpress.org/secret-key/1.1/salt/

Paste the generated block in place of the default `AUTH_KEY`, `SECURE_AUTH_KEY`, `LOGGED_IN_KEY`, `NONCE_KEY`, and corresponding `*_SALT` values.

## Step 8: Install PHP and Required Extensions
```bash
dnf install -y php8.4 php-mysqlnd
```

## Step 9: Deploy WordPress Files
```bash
cd /home/ec2-user
sudo cp -r wordpress/* /var/www/html/
sudo service httpd restart
```
---

## High Availability Extension: Blue-Green Deployment

To make the architecture scalable and highly available:

1. **Create an AMI** of the fully configured WordPress EC2 instance (`WordpressAMI`).
2. **Launch a second EC2 instance** from this AMI (skips manual DB setup steps since config is baked in).
3. **Create a Target Group** — configure health checks against `/readme.html` or `index.php`.
4. **Create an Application Load Balancer (ALB)** and attach the Target Group.
5. **Create a Launch Template** using `WordpressAMI`.
6. **Create an Auto Scaling Group (ASG)** using the Launch Template, attached to the Target Group.
7. **Access the application via the ALB DNS name** — traffic is now distributed and can scale automatically.

This setup supports **Blue-Green deployment**: new versions can be rolled out on a fresh AMI/target group, tested, and traffic can be shifted over via the ALB with minimal downtime.

---
## Step 10: Complete Installation via Browser
- Navigate to: `<http://13.233.26.20/>` → completes the WordPress setup wizard.
- Navigate to: `<http://13.233.26.20/>/wp-admin` → WordPress admin dashboard login.
- Navigate to: `<http://Wordpress-ALB-1048355765.ap-south-1.elb.amazonaws.com> → WordPress ALB DNS link.


## Key Learnings
- Decoupling the database (RDS) from the application server (EC2) improves security, scalability, and maintainability.
- Keeping RDS **Publicly Inaccessible** and restricting access via Security Groups follows AWS security best practices.
- Using an AMI + Launch Template + ASG + ALB combination enables a horizontally scalable, fault-tolerant WordPress deployment.
- WordPress salts/keys should always be uniquely generated per deployment for session security.

## Tech Stack Summary
`Amazon Linux 2023` · `Apache (httpd)` · `PHP 8.4` · `Amazon RDS (MySQL)` · `WordPress` · `ALB` · `Auto Scaling Group` · `AMI`

---

