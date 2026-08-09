# DevOps Tooling Solution — 3-Tier Web Application

A highly available 3-tier web application infrastructure built on **AWS EC2**, using multiple web servers, a dedicated MySQL database server, and an NFS server for shared application and log storage.

---

- [DevOps Tooling Solution — 3-Tier Web Application](#devops-tooling-solution--3-tier-web-application)
- [1. Project Overview](#1-project-overview)
- [2. Architecture](#2-architecture)
- [3. Preparing NFS Server](#3-preparing-nfs-server)
- [4. Configure the Database Server](#4-configure-the-database-server)
- [5. Preparing the Web Servers](#5-preparing-the-web-servers)
- [5.1 Web Server 1](#51-web-server-1)
- [5.2 Web Server 2](#52-web-server-2)
- [5.3 Web Server 3](#53-web-server-3)
- [6. Tooling Configuration](#6-tooling-configuration)
- [From Web Server 2](#from-web-server-2)
- [From Web Server 3](#from-web-server-3)
- [7. Lessons Learned](#7-lessons-learned)
  - [7.1 Shared Storage](#71-shared-storage)
  - [7.2 Separation of Responsibilities](#72-separation-of-responsibilities)
  - [7.3 Centralized Database](#73-centralized-database)
  - [7.4 Linux Security](#74-linux-security)
- [Conclusion](#conclusion)

---

# 1. Project Overview

This project demonstrates the deployment of a **3-tier web application architecture** using AWS infrastructure.

The application consists of:

1. **Shared Storage Tier**

   * Dedicated NFS server
   * Shared application files
   * Shared Apache logs
   * Shared storage for other application data


2. **Database Tier**

   * Dedicated MySQL database server
   * Centralized application database

3. **Web Tier**

   * Three Apache web servers
   * PHP/PHP-FPM
   * Shared application files through NFS



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
                        NFS Server 
                            |
                            v
                    +----------------+
                    | /mnt/apps      |
                    | /mnt/logs      |
                    | /mnt/opt       |
                    +----------------+
                       /    |    \
                      /     |     \
                     v      v      v
                    Web 1  Web 2  Web 3
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


             
```

---

# 3. Preparing NFS Server

The NFS server provides shared storage to all web servers.

The NFS server contains:

```text
/mnt/apps
/mnt/logs
/mnt/opt
```


1.  Launching an EC2 instance that will server as "NFS Server" 

![nfs](<Images/1- nfs ec2 instance.png>)

2.  Create 3 Elastic Block Storage (EBS) and attach it to the NFS server EC2

![nfs ebs](<Images/2- nfs volume.png>)


3.  Login into the EC2 via the linux terminal

```
ssh -i "key.pem" ec2-user@web-server-public-ip
```

![ssh](<Images/3- nfs server login.png>)

4.  Check that block are attached to the web server

```
lsblk
```
   
![blocks](<Images/4- nfs volumes.png>)

5.  Update the web server

```
sudo yum -y update 
```

![update](<Images/4- server update.png>)

6.  Create a single partition on each of the 3 disks attached to the Database server

```
sudo fdisk /dev/nvme1n1
sudo fdisk /dev/nvme2n1
sudo fdisk /dev/nvme3n1
```

![d1](Images/5-fdisk1.png)
![d2](Images/6-fdisk2.png)
![d3](<Images/7- fdisk3.png>)


7.  Install the Logical Volume Manager (LVM)

```
sudo yum install lvm2 
```
![lvm](Images/8-lvm2.png)



8.  Create Physical Volumes (PV) on each of the newly created partitions and verify the Physical volumes has been created successfully.

```
sudo pvcreate /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1

sudo pvs
```

![pv](Images/9-pvs.png)


9.  Add all 3 PVs to a volume group and verify that the VG has been created successfully

```
sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1

sudo vgs
```

![vg](Images/10-vgs.png)

10.   Create 3 logical volumes (LV) from the volume group. Name it **"lv-apps"**, **"lv-opt"** and **"lv-logs"** and verify that the LV has been created successfully.

```
sudo lvcreate -L 10G -n lv-apps webdata-vg

sudo lvcreate -L 10G -n lv-opt webdata-vg

sudo lvcreate -L 5G -n lv-logs webdata-vg

sudo lvs
```

![lvs](Images/11-lvs.png)


11.  list all blocks

```
lsblk
```

![lsblk](<Images/12- lsblk.png>)

12.  Format the logical volumes with the xfs filesystem and verify that the LV has been formatted successfully.

```
sudo mkfs.xfs /dev/webdata-vg/lv-apps

sudo mkfs.xfs /dev/webdata-vg/lv-logs

sudo mkfs.xfs /dev/webdata-vg/lv-opt
```

![format](<Images/14-format volume.png>)

13.  Create Mount directory for the logical volumes

lv-apps > /mnt/apps

lv-opt > /mnt/opt

lv-logs > /mnt/logs

```
sudo mkdir -p /mnt/apps

sudo mkdir -p /mnt/logs

sudo mkdir -p /mnt/opt
```
![mount](<Images/13-mounting directory.png>)


14.  Mount **/mnt/apps** on **lv-apps**
   
    Mount **/mnt/logs** on **lv-logs**

    Mount **/mnt/opt** on **lv-opt**

```
sudo mount /dev/webdata-vg/lv-apps /mnt/apps

sudo mount /dev/webdata-vg/lv-logs /mnt/logs

sudo mount /dev/webdata-vg/lv-opt /mnt/opt
```

![mount](Images/15-mount.png)


15.   Verify all setup

```
df -h
```

![setup](Images/16-verify.png)


16.  list all blocks

```
lsblk
```

![lsblk](Images/17-lsblk.png)


17. Check block UUIID for fstab configuration

```
sudo blkid
```

![blkid](Images/18-blkid.png)




18.  Update **/etc/fstab** file to ensure that the mounted configuration persists after the restart of the server.

```
sudo vi /etc/fstab
```
![fstab](Images/19-fstab.png)


19.  Test the configuration and reload the daemon

```
sudo mount -a  

sudo systemctl daemon-reload
```

![configuration](<Images/20-verify fstab.png>)


20.   Install NFS Server, configure it to start on reboot and ensure it is up and running

```
sudo yum install nfs-utils -y
sudo systemctl enable nfs-server.service
sudo systemctl start nfs-server.service
sudo systemctl status nfs-server.service
```

![nfs](<Images/22-nfs install.png>)
![nfs](Images/23-nfs.png)


21.   Set up permission that will allow the Web Servers to read, write and execute files on NFS.
The exports were configured for the web-server network:

![perm](<Images/24-file permission.png>)

22.    Verify subnet CIDR ip address

```
hostname -I
ip route
```

![subnet](<Images/24b-verify subnet.png>)


23.   Export the mounts for Webservers' subnet cidr(IPv4 cidr) to connect as clients.
 ```text
/mnt/apps 172.31.16.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/logs 172.31.16.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/opt 172.31.16.0/20(rw,sync,no_all_squash,no_root_squash)

sudo exportfs -arv
```  

![export](<Images/25-export file.png>)
![ex](<Images/26-export directory.png>)


24.   Check which port is used by NFS and open it using the security group (add new inbound rule)

```
rpcinfo -p | grep nfs
```

![nfs port](<Images/27- nfs port.png>)


**For NFS Server to be accessible from the client, the following ports added to the security inbound rules: TCP 111, UDP 111, NFS 2049. The Web Server subnet cidr was set as the traffic source.

![sec](<Images/28-inbound rules.png>)



# 4. Configure the Database Server


1.  Launching an EC2 instance that will server as "Database Server" 

![db server](<Images/29-database server.png>)


2.  Login into the EC2 via the linux terminal

```
ssh -i "key.pem" ec2-user@web-server-public-ip
```

![db](<Images/30-db ssh login.png>)

3. Update the Database server

```
sudo yum -y update 
```

![db upd](<Images/31-database server update.png>)

4. Confirm MySQL package name on RHEL 10.2

```
sudo dnf search mysql | grep -i server
```

![db package](<Images/32- confirm mysql package.png>)


5. Install Mysql Server and ensure it us up and running
```
sudo dnf install -y mysql8.4-server

sudo systemctl enable mysqld
sudo systemctl start mysqld
sudo systemctl status mysqld
```
![mysql](<Images/33- mysql.png>)
![mysql](<Images/34 -mysql2.png>)

6. Create a database and name it tooling

Create a database user and name it webaccess

Grant permission to webaccess user on tooling database to do anything only from the webservers subnet cidr

```
sudo mysql

CREATE DATABASE tooling;
CREATE USER 'webaccess'@'172.31.16.0/20' IDENTIFIED WITH mysql_native_password BY 'Admin123@';
GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'172.31.16.0/20' WITH GRANT OPTION;
FLUSH PRIVILEGES;
show databases;

```

![mysql](<Images/35- MYSQL USER.png>)
![mysql](<Images/35- MYSQL DATABASE.png>)


7. Set Bind Address and restart MySQL

```
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
```

![m](<Images/36- myqsl config file .png>)

The command that was used shows an empty configuration file because the RHEL mysql configuration file directory is different from Ubuntu. Therefore i had to find the RHEL mysql configuration file directory using this command

```
sudo find /etc -name "my.cnf" -o -name "*.cnf" | grep mysql

```

![mysql](Images/37-mysql.png)

The RHEL mysql configuration file directory was displayed and the configuration file was updated with the new bind address

```
sudo vi /etc/my.cnf.d/mysql-server.cnf
```

![mysql](<Images/38-mysql config.png>)


8. Open MySQL port 3306 on the DB Server EC2.
Access to the DB Server is allowed only from the Subnet Cidr configured as source.

![port](<Images/39-port 3306.png>)



# 5. Preparing the Web Servers
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


# 5.1 Web Server 1
1. Launch a new EC2 instance with RHEL Operating System

![web 1](<Images/40-web server 1.png>)

2. Login into the EC2 via the linux terminal

```
ssh -i "key.pem" ec2-user@web-server-public-ip
```

![login](<Images/41-server login.png>)

3. Install NFS Client

```
sudo yum install nfs-utils nfs4-acl-tools -y
```

![nfs](<Images/42- nfs client.png>)


4. Mount /var/www/ and target the NFS server's export for apps. NFS Server private IP address = 172.31.20.171

5. Verify that NFS was mounted successfully by running df -h. 

```
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid 172.31.20.171:/mnt/apps /var/www

df -h
```

![alt text](Images/43-mount.png)


6. Update the fstab file to ensure that the changes will persist after reboot. 

```
sudo vi /etc/fstab

172.31.20.171:/mnt/apps /var/www nfs defaults 0 0
```

![fstab](<Images/44- fstab file.png>)


7. Install Apache, Remi's repository and PHP

```
sudo yum install httpd -y
```

![httpd](<Images/45- install apache.png>)


```
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
```

![/](Images/46.png)

```
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm
```
![remi](Images/47-remi.png)

```
sudo dnf module reset php
```
![reset](Images/48-resetphp.png)


```
sudo dnf module enable php:remi-8.2
```

![remi](Images/49-enablermi.png)

```
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd
```

![php](Images/50-php.png)


```
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo systemctl status php-fpm
```

![php](<Images/51-php running.png>)



# 5.2 Web Server 2
1. Launch a new EC2 instance with RHEL Operating System

![web 2](<Images/52-web server 2.png>)

2. Login into the EC2 via the linux terminal

```
ssh -i "key.pem" ec2-user@web-server-public-ip
```

![alt text](<Images/53-server login.png>)

3. Install NFS Client

```
sudo yum install nfs-utils nfs4-acl-tools -y
```

![alt text](<Images/54- nfs client.png>)


4. Mount /var/www/ and target the NFS server's export for apps. NFS Server private IP address = 172.31.20.171

5. Verify that NFS was mounted successfully by running df -h. 

```
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid 172.31.20.171:/mnt/apps /var/www

df -h
```

![alt text](Images/55-mount.png)


6. Update the fstab file to ensure that the changes will persist after reboot. 

```
sudo vi /etc/fstab

172.31.20.171:/mnt/apps /var/www nfs defaults 0 0
```

![alt text](<Images/56- fstab file.png>)


7. Install Apache, Remi's repository and PHP

```
sudo yum install httpd -y
```

![alt text](<Images/57- install apache.png>)


```
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
```

![alt text](Images/58.png)

```
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm
```
![alt text](Images/59-remi.png)

```
sudo dnf module reset php
```
![alt text](Images/60-resetphp.png)


```
sudo dnf module enable php:remi-8.2
```

![alt text](<Images/61-enable remi.png>)

```
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd
```

![alt text](Images/62-php.png)


```
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo systemctl status php-fpm
```

![alt text](<Images/63-php running.png>)


# 5.3 Web Server 3
1. Launch a new EC2 instance with RHEL Operating System

![alt text](<Images/88-web server 3.png>)

2. Login into the EC2 via the linux terminal

```
ssh -i "key.pem" ec2-user@web-server-public-ip
```

![alt text](<Images/89-server login.png>)

3. Install NFS Client

```
sudo yum install nfs-utils nfs4-acl-tools -y
```

![alt text](<Images/90- nfs client.png>)


4. Mount /var/www/ and target the NFS server's export for apps. NFS Server private IP address = 172.31.20.171

5. Verify that NFS was mounted successfully by running df -h. 

```
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid 172.31.20.171:/mnt/apps /var/www

df -h
```

![alt text](Images/91-mount.png)

6. Update the fstab file to ensure that the changes will persist after reboot. 

```
sudo vi /etc/fstab

172.31.20.171:/mnt/apps /var/www nfs defaults 0 0
```

![alt text](<Images/92- fstab file.png>)


7. Install Apache, Remi's repository and PHP

```
sudo yum install httpd -y
```

![alt text](<Images/93- install apache.png>)


```
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
```

![alt text](Images/94.png)

```
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm
```
![alt text](Images/95-remi.png)

```
sudo dnf module reset php
```
![alt text](Images/96-resetphp.png)


```
sudo dnf module enable php:remi-8.2
```

![alt text](<Images/97-enable remi.png>)

```
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd
```

![alt text](Images/98-php.png)


```
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo systemctl status php-fpm
```

![alt text](<Images/99-php running.png>)



# 6. Tooling Configuration

1. Verify that Apache files and directories are available on the Web Servers in /var/www and also on the NFS Server in /mnt/apps. Confirm that the NFS was mounted correctly by checking if the same files are present in both server. 
   
![alt text](<Images/64- webserver file.txt.png>)

File.txt file was created from Web Server 1, and it was accessible from Web Server 2.

![alt text](<Images/65- nfs server confirm.png>)


2. Locate the log folder for Apache on the Web Server and mount it to NFS server's export for logs. Update the etc/fstab file to ensure that the mount persists.


```
ls /var/log

sudo vi /etc/fstab
```
![alt text](<Images/65b- var-logs.png>)
![alt text](<Images/66- fstab logs.png>)


3. Fork the tooling source code from StegHub GitHub Account

![alt text](<Images/Screenshot 2026-08-09 112646.png>)


4. Deploy the tooling Website's code to the Web Server. Ensure that the html folder from the repository is deplyed to /var/www/html

Install Git and clone the repository

```
sudo yum install git
sudo git clone https://github.com/StegTechHub/tooling.git
```

![alt text](Images/67-git.png)


![alt text](<Images/68-clone steghub tooling.png>)


![alt text](Images/69-tooling.png)


5. Open selinux file and set selinux to disable to make the change permanent.

```
sudo vi /etc/sysconfig/selinux

SELINUX=disabled
```

![alt text](Images/70-SElinux.png)

6. Update the website's configuration to connect to the database (in /var/www/html/function.php file). 

```
sudo vi /var/www/html/functions.php
```

![alt text](<Images/71-web conf database.png>)


7. Apply tooling-db.sql to the database server

``` sudo mysql -h <db-private-IP> -u <db-username> -p <db-password < tooling-db.sql
```

![alt text](<Images/72- mysql.png>)

The server does not recognize the file because the command was made from the server root directory not the /var/www/tooling.


![alt text](Images/73-mysql.png)

The directory was changed to the tooling directorr but the server does not recognize the commad because the mysql client has not been installed on any of the web servers.

Install Mysql client

![alt text](<Images/74-mysql clint.png>)

After the Mysql client installation, the tooling-db.sql application was retried once again and the server detect an error stating that the user already exists on the database.

![alt text](<Images/75b-mysql user exist.png>)

Verify the user on the database for confirmation

![alt text](<Images/75c-verify database.png>)

In order to successfully apply the tooling-db.sql, the existing database was dropped and recreated.

![alt text](<Images/75d-drop database.png>)

![alt text](<Images/76- mysyl success.png>)

Database Application successful

8. Access the database server from Web Server
9. Create in MyQSL a new admin user with username: myuser and password: password

```
sudo mysql -h 172.31.8.129 -u webaccess -p

INSERT INTO users(id, username, password, email, user_type, status) VALUES (2, 'myuser', '5f4dcc3b5aa765d61d8327deb882cf99', 'user@mail.com', 'admin', '1');
```

![alt text](<Images/77- mysql access.png>)


1.   Open a browser and access the website using the Web Server public IP address http://<Web-Server-public-IP-address>/index.php. 


![alt text](<Images/78- web access 1.png>)

![alt text](<Images/78- web access 2.png>)



# From Web Server 2

i tried accessing the website from the web server 2 but the site refused to load stating and error 500.

While going through the troubleshooting process, i realized that the
SELinux remains enabled in enforcing mode.

Instead of disabling SELinux to make the application work, the required policy was enabled:

```bash
sudo setsebool -P httpd_can_network_connect_db on
```

This is preferable to disabling SELinux entirely.

After disabling the SELinux, i retried accessing the website from server 2 and it was successful

![alt text](<Images/80- web server2.png>)

![alt text](<Images/80b- web server2.png>)


# From Web Server 3

![alt text](<Images/79a- web server3.png>)

![alt text](<Images/79b- web server3.png>)


---

# 7. Lessons Learned

This project provided practical experience with several important DevOps concepts.

## 7.1 Shared Storage

NFS allows multiple servers to access the same application files.

This means:

```text
Web 1
Web 2
Web 3
```

can all serve the same application without maintaining separate copies of the application files.

---

## 7.2 Separation of Responsibilities

The architecture separates:

```text
Web Layer
Database Layer
Storage Layer
```

This makes the infrastructure easier to scale and maintain.

---

## 7.3 Centralized Database

Instead of installing MySQL on every web server, a dedicated database server is used.

```text
Web 1 ─┐
Web 2 ─┼──> MySQL
Web 3 ─┘
```

This provides a centralized source of application data.

---

## 7.4 Linux Security

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
