# DevOps Tooling Solution — 3-Tier Web Application

A highly available 3-tier web application infrastructure built on **AWS EC2**, using multiple web servers, a dedicated MySQL database server, and an NFS server for shared application and log storage.

---

## Table of Contents

* [1. Project Overview](#1-project-overview)
* [2. Architecture](#2-architecture)
* [3. Infrastructure Components](#3-infrastructure-components)
* [4. Web Tier](#4-web-tier)
* [5. NFS Shared Storage](#5-nfs-shared-storage)
* [6. Database Tier](#6-database-tier)
* [7. PHP and Apache Configuration](#7-php-and-apache-configuration)
* [8. SELinux Configuration](#8-selinux-configuration)
* [9. Database Configuration](#9-database-configuration)
* [10. Application Deployment](#10-application-deployment)
* [11. Troubleshooting](#11-troubleshooting)
* [12. Important Commands](#12-important-commands)
* [13. Final Architecture Flow](#13-final-architecture-flow)
* [14. Security Considerations](#14-security-considerations)
* [15. Lessons Learned](#15-lessons-learned)
* [16. Future Improvements](#16-future-improvements)

---

# 1. Project Overview

This project demonstrates the deployment of a **3-tier web application architecture** using AWS infrastructure.

The application consists of:

1. **Web Tier**

   * Three Apache web servers
   * PHP/PHP-FPM
   * Shared application files through NFS

2. **Database Tier**

   * Dedicated MySQL database server
   * Centralized application database

3. **Shared Storage Tier**

   * Dedicated NFS server
   * Shared application files
   * Shared Apache logs
   * Shared storage for other application data

The objective is to demonstrate practical DevOps concepts including:

* Linux server administration
* AWS EC2
* EBS storage
* LVM
* NFS
* Apache
* PHP-FPM
* MySQL
* SELinux
* Network communication between application tiers
* Shared storage
* Multi-server application deployment
* Troubleshooting and system diagnosis

---

# 2. Architecture

The overall architecture is:

```text
                         INTERNET
                            |
                            |
                    +----------------+
                    | Load Balancer  |
                    +----------------+
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
      +-------------+ +-------------+ +-------------+
      | Web Server 1| | Web Server 2| | Web Server 3|
      |   Apache    | |   Apache    | |   Apache    |
      |   PHP-FPM   | |   PHP-FPM   | |   PHP-FPM   |
      +-------------+ +-------------+ +-------------+
             |              |              |
             +--------------+--------------+
                            |
                            |
                  +-------------------+
                  |  MySQL Database   |
                  |     Server        |
                  +-------------------+
                            |
                       tooling DB


             Shared Storage
                    |
                    v
            +----------------+
            |   NFS Server   |
            |                |
            | /mnt/apps      |
            | /mnt/logs      |
            | /mnt/opt       |
            +----------------+
               /    |    \
              /     |     \
             v      v      v
          Web 1  Web 2  Web 3
```

---

# 3. Infrastructure Components

| Component           | Purpose                                     |
| ------------------- | ------------------------------------------- |
| Web Server 1        | Hosts application                           |
| Web Server 2        | Hosts application                           |
| Web Server 3        | Hosts application                           |
| MySQL Server        | Central database                            |
| NFS Server          | Shared application/log storage              |
| Apache              | Web server                                  |
| PHP-FPM             | Executes PHP applications                   |
| MySQL Client        | Allows web servers to test/connect to MySQL |
| NFS Client          | Mounts shared NFS storage                   |
| SELinux             | Provides mandatory access control           |
| EBS                 | Persistent block storage for EC2            |
| LVM                 | Storage management                          |
| AWS Security Groups | Network-level access control                |

---

# 4. Web Tier

Three independent EC2 instances were configured as web servers.

Each web server contains:

```text
Apache
   |
PHP-FPM
   |
PHP Application
   |
MySQL Database
```

The web servers use the same application files through the NFS server.

The application is located under:

```text
/var/www/html
```

The `/var/www` directory is mounted from the NFS server.

Example:

```bash
findmnt /var/www
```

Expected output:

```text
TARGET   SOURCE
/var/www 172.31.20.171:/mnt/apps
```

This means `/var/www` is not stored locally on the web server.

Instead, it is being provided by the NFS server.

---

# 5. NFS Shared Storage

The NFS server provides shared storage to all web servers.

The NFS server contains:

```text
/mnt/apps
/mnt/logs
/mnt/opt
```

The exports were configured for the web-server network:

```text
/mnt/apps 172.31.16.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/logs 172.31.16.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/opt 172.31.16.0/20(rw,sync,no_all_squash,no_root_squash)
```

The NFS service was enabled and started using:

```bash
sudo systemctl enable nfs-server.service
sudo systemctl start nfs-server.service
```

The service was verified with:

```bash
sudo systemctl status nfs-server.service
```

---

## 5.1 Application Storage

The application storage:

```text
NFS Server
172.31.20.171:/mnt/apps
            |
            v
       /var/www
```

was mounted on each web server.

Example:

```bash
sudo mount -t nfs -o rw,nosuid 172.31.20.171:/mnt/apps /var/www
```

This allows all web servers to access the same application files.

For example:

```text
Web Server 1 ─┐
Web Server 2 ─┼──> /mnt/apps
Web Server 3 ─┘
```

---

## 5.2 Shared Apache Logs

Apache logs were also stored on the NFS server.

The NFS export:

```text
172.31.20.171:/mnt/logs
```

was mounted to:

```text
/var/log/httpd
```

Example:

```bash
sudo mount -t nfs -o rw,nosuid 172.31.20.171:/mnt/logs /var/log/httpd
```

Verification:

```bash
findmnt /var/log/httpd
```

Example:

```text
TARGET         SOURCE
/var/log/httpd 172.31.20.171:/mnt/logs
```

This allows Apache logs from multiple web servers to be stored on shared storage.

---

# 6. Database Tier

A dedicated EC2 instance was configured as the MySQL database server.

Database server:

```text
172.31.25.251
```

The application database is:

```text
tooling
```

The database contains the application tables, including:

```text
users
```

The database was verified using:

```sql
SHOW DATABASES;
```

Result included:

```text
tooling
```

The tables were verified with:

```sql
USE tooling;

SHOW TABLES;
```

---

## 6.1 Users Table

The application contains a `users` table with fields including:

```text
id
username
password
email
user_type
status
```

Example records included:

```text
1 | admin  | admin@steghub.com | admin | 1
2 | myuser | user@mail.com     | admin | 1
```

---

# 7. PHP and Apache Configuration

Apache was used as the web server.

PHP-FPM was used to process PHP requests.

The services were enabled using:

```bash
sudo systemctl enable httpd
sudo systemctl enable php-fpm
```

They were started with:

```bash
sudo systemctl start httpd
sudo systemctl start php-fpm
```

Service status can be checked using:

```bash
sudo systemctl status httpd
```

and:

```bash
sudo systemctl status php-fpm
```

---

## 7.1 PHP MySQL Extensions

The web servers require PHP MySQL support.

The installed modules were verified with:

```bash
php -m | grep -Ei "mysqli|pdo_mysql|mysqlnd|session"
```

Expected modules:

```text
mysqli
mysqlnd
pdo_mysql
session
```

These modules allow PHP applications to communicate with MySQL.

---

# 8. SELinux Configuration

One of the major troubleshooting issues encountered during the project was SELinux.

SELinux was running in:

```text
Enforcing
```

mode.

This is important because SELinux can prevent applications from performing actions even when normal Linux permissions appear correct.

---

## 8.1 The Problem

Web Server 2 initially returned:

```text
HTTP ERROR 500
```

The PHP error revealed:

```text
Fatal error: Uncaught mysqli_sql_exception:
Permission denied
```

The application was attempting to connect to:

```text
172.31.25.251:3306
```

The SELinux audit log revealed:

```text
avc: denied { name_connect }
dest=3306
scontext=system_u:system_r:httpd_t:s0
tcontext=system_u:object_r:mysqld_port_t:s0
```

This indicated that PHP-FPM, running under the Apache SELinux context, was being prevented from connecting to the MySQL port.

---

## 8.2 Diagnosing the SELinux Problem

SELinux mode was checked using:

```bash
getenforce
```

Result:

```text
Enforcing
```

The relevant SELinux boolean was checked with:

```bash
getsebool httpd_can_network_connect_db
```

The result was:

```text
httpd_can_network_connect_db --> off
```

This confirmed that Apache/PHP was not permitted to make database network connections.

---

## 8.3 Fix

The boolean was enabled using:

```bash
sudo setsebool -P httpd_can_network_connect_db on
```

The `-P` option makes the change persistent across reboots.

The configuration was then verified:

```bash
getsebool httpd_can_network_connect_db
```

Expected:

```text
httpd_can_network_connect_db --> on
```

The services were restarted:

```bash
sudo systemctl restart php-fpm
sudo systemctl restart httpd
```

After this change, the application successfully connected to MySQL.

---

# 9. Database Configuration

The application uses MySQL as a centralized database.

The connection follows this model:

```text
PHP Application
      |
      | MySQL connection
      |
      v
172.31.25.251:3306
      |
      v
   tooling
```

The application connects using the `webaccess` MySQL user.

---

## 9.1 Testing MySQL Connectivity

From a web server, the MySQL client can be used to test connectivity:

```bash
mysql -h 172.31.25.251 -u webaccess -p tooling
```

Once connected:

```sql
SHOW TABLES;
```

and:

```sql
SELECT * FROM users;
```

can be used to verify database access.

---

# 10. Application Deployment

The application files are stored on the shared NFS application volume.

The web root is:

```text
/var/www/html
```

The application contains files such as:

```text
index.php
login.php
register.php
create_user.php
admin_tooling.php
functions.php
style.css
tooling_stylesheets.css
img/
```

The application is therefore accessible from all web servers because they share the same NFS application storage.

---

## 10.1 Application Structure

```text
/var/www/html
├── index.php
├── login.php
├── register.php
├── create_user.php
├── admin_tooling.php
├── functions.php
├── style.css
├── tooling_stylesheets.css
└── img/
```

---

# 11. Troubleshooting

## 11.1 Apache Failed to Start

Apache initially failed because it could not access:

```text
/etc/httpd/logs/error_log
```

The Apache log directory was linked to:

```text
/var/log/httpd
```

which was mounted from NFS.

The mount was verified using:

```bash
findmnt /var/log/httpd
```

Once the NFS log mount and permissions were correctly configured, Apache started successfully.

---

## 11.2 HTTP 500 Error

The application initially returned:

```text
HTTP 500 Internal Server Error
```

The first step was to check Apache:

```bash
sudo systemctl status httpd
```

Apache was running.

PHP-FPM was also running:

```bash
sudo systemctl status php-fpm
```

PHP MySQL modules were verified:

```bash
php -m | grep -Ei "mysqli|pdo_mysql|mysqlnd|session"
```

The real PHP error was exposed by temporarily enabling:

```text
display_errors = On
display_startup_errors = On
error_reporting = E_ALL
log_errors = On
```

The resulting error was:

```text
mysqli_sql_exception: Permission denied
```

SELinux audit logs then revealed the actual cause.

---

## 11.3 SELinux Troubleshooting Commands

Useful commands:

```bash
getenforce
```

Check SELinux boolean:

```bash
getsebool httpd_can_network_connect_db
```

Check recent SELinux denials:

```bash
sudo ausearch -m AVC -ts recent
```

Filter for web/database issues:

```bash
sudo ausearch -m AVC -ts recent | grep -Ei "httpd|php|mysql|denied"
```

Enable database network connections:

```bash
sudo setsebool -P httpd_can_network_connect_db on
```

---

# 12. Important Commands

## Apache

Check status:

```bash
sudo systemctl status httpd
```

Start:

```bash
sudo systemctl start httpd
```

Restart:

```bash
sudo systemctl restart httpd
```

Enable at boot:

```bash
sudo systemctl enable httpd
```

Test configuration:

```bash
sudo apachectl configtest
```

or:

```bash
sudo httpd -t
```

---

## PHP-FPM

Check status:

```bash
sudo systemctl status php-fpm
```

Restart:

```bash
sudo systemctl restart php-fpm
```

---

## NFS

Check mounts:

```bash
findmnt /var/www
```

Check log mount:

```bash
findmnt /var/log/httpd
```

Mount application storage:

```bash
sudo mount -t nfs -o rw,nosuid 172.31.20.171:/mnt/apps /var/www
```

Mount log storage:

```bash
sudo mount -t nfs -o rw,nosuid 172.31.20.171:/mnt/logs /var/log/httpd
```

---

## MySQL

Connect to database:

```bash
mysql -h 172.31.25.251 -u webaccess -p tooling
```

Check databases:

```sql
SHOW DATABASES;
```

Select database:

```sql
USE tooling;
```

Check tables:

```sql
SHOW TABLES;
```

Check users:

```sql
SELECT id, username, email, user_type, status FROM users;
```

---

## SELinux

Check mode:

```bash
getenforce
```

Check database connection permission:

```bash
getsebool httpd_can_network_connect_db
```

Enable:

```bash
sudo setsebool -P httpd_can_network_connect_db on
```

Check audit events:

```bash
sudo ausearch -m AVC -ts recent
```

---

# 13. Final Architecture Flow

The complete application request flow is:

```text
                         USER
                           |
                           v
                    AWS Load Balancer
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
      Web Server 1     Web Server 2     Web Server 3
          |                |                |
          | Apache         | Apache         | Apache
          | PHP-FPM        | PHP-FPM        | PHP-FPM
          |                |                |
          +----------------+----------------+
                           |
                           |
                     PHP Application
                           |
                           v
                  MySQL Database Server
                    172.31.25.251
                           |
                           v
                       tooling DB
```

Shared storage:

```text
                    NFS SERVER
                   172.31.20.171
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
       /mnt/apps      /mnt/logs      /mnt/opt
          |              |              |
          v              v              v
       /var/www      /var/log/httpd   /var/opt
       Web Servers    Web Servers      Web Servers
```

---

# 14. Security Considerations

The architecture uses multiple layers of security.

### AWS Security Groups

Security groups should restrict traffic between infrastructure components.

Example:

```text
Internet
   |
   | 80/443
   v
Web Servers
   |
   | 3306
   v
Database Server
```

The MySQL server should not expose port `3306` publicly.

Only the web-server security group/subnet should be allowed to access it.

Similarly, NFS traffic should only be accessible from the required web-server network.

---

## SELinux

SELinux remains enabled in enforcing mode.

Instead of disabling SELinux to make the application work, the required policy was enabled:

```bash
sudo setsebool -P httpd_can_network_connect_db on
```

This is preferable to disabling SELinux entirely.

---

# 15. Lessons Learned

This project provided practical experience with several important DevOps concepts.

## 15.1 Shared Storage

NFS allows multiple servers to access the same application files.

This means:

```text
Web 1
Web 2
Web 3
```

can all serve the same application without maintaining separate copies of the application files.

---

## 15.2 Separation of Responsibilities

The architecture separates:

```text
Web Layer
Database Layer
Storage Layer
```

This makes the infrastructure easier to scale and maintain.

---

## 15.3 Centralized Database

Instead of installing MySQL on every web server, a dedicated database server is used.

```text
Web 1 ─┐
Web 2 ─┼──> MySQL
Web 3 ─┘
```

This provides a centralized source of application data.

---

## 15.4 Linux Security

A service being "running" does not necessarily mean that it can perform every operation it needs.

In this project:

```text
Apache = running
PHP-FPM = running
MySQL = running
Network = available
```

but:

```text
SELinux = blocking PHP → MySQL
```

The issue was diagnosed through SELinux audit logs.

This demonstrates why DevOps engineers must understand both:

* application configuration
* operating-system security policies

---

## 15.5 Troubleshooting Methodology

The HTTP 500 error demonstrated a useful troubleshooting process:

```text
Application error
      |
      v
Check Apache
      |
      v
Check PHP-FPM
      |
      v
Check PHP modules
      |
      v
Test database connectivity
      |
      v
Check SELinux
      |
      v
Check audit logs
      |
      v
Identify exact denial
      |
      v
Apply targeted fix
```

Rather than disabling security controls or reinstalling services, the specific cause was identified and corrected.

---

# 16. Future Improvements

The current architecture can be extended into a more production-oriented DevOps platform.

Potential improvements include:

### Load Balancer

Add an AWS Application Load Balancer:

```text
Internet
   |
   v
Application Load Balancer
   |
   +---- Web 1
   +---- Web 2
   +---- Web 3
```

---

### Auto Scaling

Configure an Auto Scaling Group so additional web servers can automatically be created when demand increases.

---

### HTTPS

Configure TLS/SSL using AWS Certificate Manager and HTTPS.

---

### CI/CD

The existing application repository can be integrated with:

* GitHub
* Jenkins
* GitHub Actions

A CI/CD pipeline could automatically:

```text
Developer
   |
   v
Git Push
   |
   v
GitHub
   |
   v
CI/CD Pipeline
   |
   v
Build / Test
   |
   v
Deployment
   |
   v
Web Servers
```

---

### Infrastructure as Code

The infrastructure can eventually be recreated using:

* Terraform
* Ansible
* AWS CloudFormation

Instead of manually configuring every server.

---

### Monitoring

Monitoring can be introduced using:

* AWS CloudWatch
* Prometheus
* Grafana
* ELK/OpenSearch

---

### Database High Availability

The current architecture uses a single MySQL database server.

For higher availability, this could eventually be replaced with:

```text
Amazon RDS
```

or a MySQL replication/cluster architecture.

---

# Conclusion

This project demonstrates a practical **3-tier DevOps infrastructure** consisting of multiple Apache/PHP web servers, a centralized MySQL database server, and shared NFS storage.

The project also demonstrates real-world Linux administration and troubleshooting, particularly the interaction between:

```text
Apache
   ↓
PHP-FPM
   ↓
SELinux
   ↓
Network
   ↓
MySQL
```

A key troubleshooting lesson was that an HTTP 500 error does not necessarily mean Apache is broken. The application may be running correctly while an underlying operating-system security policy prevents it from communicating with another service.

The final infrastructure provides a foundation that can later be extended with:

* Load balancing
* Auto scaling
* CI/CD
* Infrastructure as Code
* Monitoring
* HTTPS
* Database high availability
* Automated deployments

This project therefore serves as a practical foundation for progressing from traditional Linux administration toward modern **DevOps and cloud engineering practices**.
