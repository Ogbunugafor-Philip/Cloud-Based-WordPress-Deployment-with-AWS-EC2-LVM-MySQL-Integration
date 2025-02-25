# Cloud-Based WordPress Deployment with AWS EC2, LVM & MySQL Integration

## Introduction

In today's digital landscape, businesses and individuals rely on scalable and secure web hosting solutions to ensure high availability and performance. This project focuses on deploying WordPress, a widely used Content Management System (CMS), on Amazon Web Services (AWS) using a three-tier architecture.

The deployment is designed to achieve high availability, efficient storage management, and secure database connectivity by leveraging AWS EC2 instances, Logical Volume Management (LVM) for dynamic storage allocation, and a dedicated MySQL database server. The architecture follows best practices for cloud-based web hosting, ensuring a robust and maintainable environment.

## Project Objectives

1. Deploy a scalable three-tier architecture for WordPress on AWS, utilizing separate EC2 instances for the Web Server and Database Server.
2. Implement Logical Volume Management (LVM) on both servers to enable efficient storage management and scalability.
3. Establish secure database connectivity between the Web Server (Apache, PHP, WordPress) and the Database Server (MySQL) using security best practices.
4. Optimize system performance, security, and high availability by configuring Apache, MySQL, firewall rules, and SELinux policies.

## Three-Tier Architecture

The Three-Tier Architecture is a widely used software architecture pattern for web and mobile applications. It consists of three distinct layers, each responsible for a specific function:

- **Presentation Layer (PL)**: This is the user interface layer, where users interact with the application through a web browser or client application.
- **Business Layer (BL)**: This is the application or web server that processes user requests and executes business logic.
- **Data Access Layer (DAL)**: This is where data is stored and managed, typically in a database or file system.

### Three-Tier Architecture Setup for WordPress Deployment

In this project, we implement a three-tier architecture using AWS EC2 instances to ensure scalability, security, and performance. The setup consists of:

- **Client (Presentation Layer)**: A laptop or PC used to access the WordPress site through a web browser.
- **Web Server (Business Layer)**: An EC2 Linux Server that hosts the WordPress application.
- **Database Server (Data Access Layer)**: A separate EC2 Linux Server running MySQL to store and manage WordPress data.

## Project Steps

### Step 1: Prepare the Web Server (EC2 Instance)
Amazon Elastic Compute Cloud (EC2) is a cloud computing service provided by AWS that allows users to run virtual machines (instances) on the cloud. When used as a web server, an EC2 instance hosts and serves websites, web applications, or APIs over the internet.


Launch an EC2 instance for the Web Server
```bash
aws ec2 run-instances --image-id ami-xxxxxxxx --count 1 --instance-type t2.micro --key-name MyKeyPair --security-groups MySecurityGroup
```

We would need to create and attach three EBS volumes (10 GiB each) to our webserver instance. Amazon Elastic Block Store (EBS) is a scalable, high-performance block storage service for EC2 instances, used for storing persistent data, databases, file systems, and application 

```bash
aws ec2 attach-volume --volume-id vol-xxxxxxxx --instance-id i-xxxxxxxx --device /dev/xvdbb
aws ec2 attach-volume --volume-id vol-yyyyyyyy --instance-id i-xxxxxxxx --device /dev/xvdbc
aws ec2 attach-volume --volume-id vol-zzzzzzzz --instance-id i-xxxxxxxx --device /dev/xvdbd
```


Verify Attached Volumes
```bash
lsblk
```

### Step 2: Configure Storage Using LVM on the Web Server
LVM (Logical Volume Manager) is a storage management system that allows flexible and dynamic partitioning of disk space, enabling easier resizing, snapshots, and efficient allocation of storage across multiple physical drives.

Partition the new disks using gdisk
```bash
gdisk /dev/xvdbb
gdisk /dev/xvdbc
gdisk /dev/xvdbd
```

Install and configure LVM (Physical Volumes, Volume Group, Logical Volumes). LVM (Logical Volume Manager) is a storage management system in Linux that provides flexibility and scalability for managing disk space. Run;
```bash
sudo apt update
sudo apt install -y lvm2
```

Scan all available disk devices and partitions to identify those that can be used for Logical Volume Manager (LVM). Run;
```bash
sudo lvmdiskscan
```
Initialize disks or partitions as LVM Physical Volumes (PVs). This step is necessary before these disks can be added to a Volume Group (VG) and further divided into Logical Volumes (LVs). Run;
```bash
sudo pvcreate /dev/xvdbb1 /dev/xvdbc1 /dev/xvdbd1
```
To confirm that the physical volumes were created successfully, run;
```bash
sudo pvs
```
Next, we need to create a Volume Group (VG) by combining all our Physical Volumes (PVs). A Volume Group acts as a storage pool from which Logical Volumes (LVs) can be created. The below command creates a VG and names in webdata-vg. Run;
```bash
sudo vgcreate webdata-vg /dev/xvdbb1 /dev/xvdbc1 /dev/xvdbd1
```

Use lvcreate utility to create 2 logical volumes. apps-lv (Use half of the PV size), and logs-lv Use the remaining space of the PV size. 
NOTE: apps-lv will be used to store data for the website while logs-lv will be used to store data for logs. Run;

```bash
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
```
Verify that your Logical Volume has been created successfully. Run;
```bash
sudo lvs
```
Verify the entire setup
```bash
sudo vgdisplay -v
sudo lsblk
```

Use mkfs.ext4 to format the logical volumes with ext4 filesystem. Use mkfs.ext4 to format the logical volumes with ext4 filesystem. Run; 
```bash
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```
Create /var/www/html directory to store website files. Run;
```bash
sudo mkdir -p /var/www/html
```

Create /home/recovery/logs to store backup of log data
```bash
sudo mkdir -p /home/recovery/logs
```
To confirm the creation of these two directories, run;
```bash
ls -ld /var/www/html /home/recovery/logs
```

Mount /var/www/html on apps-lv logical volume and
```bash
sudo mount /dev/webdata-vg/apps-lv /var/www/html/
```
Use rsync utility to back-up all the files in the log directory /var/log into /home/recovery/logs (This is required before mounting the file system). Run;
```Bash
sudo rsync -av /var/log/. /home/recovery/logs/
```
Mount /var/log on logs-lv logical volume. (Note that all the existing data on /var/log will be deleted. That is why step 15 above is very important)
```Bash
sudo mount /dev/webdata-vg/logs-lv /var/log
```
Restore log files back into /var/log directory
```Bash
sudo rsync -av /home/recovery/logs/. /var/log
```

Check to confirm created directories are mounted correctly. Run
```Bash
lsblk
```
Next, we would need to update our UUID. Updating the UUID in /etc/fstab ensures that your logical volumes (LVM partitions) are automatically mounted to the correct directories (/var/www/html and /var/log) even after a reboot.
To get the UUID of the /var/www/html and /var/log, run;
```Bash
sudo blkid
```
Update /etc/fstab using your own UUID. Run;
```Bash
sudo vi /etc/fstab
```
Test the configuration and reload the daemon
```bash
sudo mount -a
sudo systemctl daemon-reload
```
Verify your setup. Run;
```bash
df -h
```

### Step 3: Prepare the Database Server (EC2 Instance)

Repeat the same steps as for the Web Server, but instead of `apps-lv`, create `db-lv` and mount it to `/db`.

### Step 4: Install and Configure MySQL on the Database Server

MySQL is an open-source relational database management system (RDBMS) that uses SQL for data storage and management. It is fast, scalable, and widely used in web applications
```bash
sudo yum install mysql-server -y
```
Start and verify MySQL is running correctly. Run;
```bash
sudo systemctl restart mysqld
sudo systemctl enable mysqld
sudo systemctl status mysqld
```
Start MySQL. Run
```bash
sudo mysql 
```

Create a WordPress database
```bash
CREATE DATABASE wordpress;
CREATE USER 'wordpress-user'@'<Web-Server-IP>' IDENTIFIED BY 'yourpassword';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress-user'@'<Web-Server-IP>';
FLUSH PRIVILEGES;
```
Note: Run these commands one by one

### Step 5: Install and Configure WordPress on the Web Server
Install Apache. Run these commands;
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install apache2 -y
```
Start and enable Apache. Run;
```bash
sudo systemctl start apache2
sudo systemctl enable apache2
sudo systemctl status apache2
```
Install PHP and it’s dependencies, run;
```bash
sudo apt install -y php php-mysql php-fpm php-opcache php-gd php-xml php-mbstring
```
Check if php is installed with the command;
```bash
php -v
```
Download and configure WordPress. Run these commands one by one
```bash
mkdir wordpress
cd   wordpress
sudo wget http://wordpress.org/latest.tar.gz
sudo tar -xzvf latest.tar.gz
sudo rm -rf latest.tar.gz
sudo cp -R wordpress/* /var/www/html/
sudo systemctl restart apache2
```
cd into /var/www/html directory. Run these commands;
```bash
cd /var/www/html/
sudo cp wp-config-sample.php wp-config.php
```


### Step 6: Configure WordPress Database Connection
Do not forget to open MySQL port 3306 on DB Server EC2. For extra security, you shall allow access to the DB server ONLY from your Web Server’s IP address, so in the Inbound Rule configuration specify source as /32

Install MySQL client, start, enable and check the status. Run;
```bash
sudo apt install -y mysql-server
sudo systemctl start mysql
sudo systemctl enable mysql
sudo systemctl status mysql 
```
Test that you can connect from your Web Server to your DB server by using mysql-client. Run;
```bash
sudo mysql -u wordpress-user -p -h 172.31.92.14
```
We would need to update our database details on our web server. Run;
```bash
sudo vi /var/www/html/wp-config.php
```
### Step 7: Security and Final Configuration
We want the webserver to serve the wordpress page with only the public ip on the browser.
Go to the main configuration file. Run

```bash
sudo vi /etc/apache2/sites-available/000-default.conf
```
Change it from "/var/www/html" to "/var/www/html/wordpress"

Apply SELinux policies for Apache. SELinux (Security-Enhanced Linux) is a security mechanism in Linux that controls access to system resources based on policies. It acts as an extra layer of security by enforcing rules on what processes (like Apache, MySQL, etc.) can access specific files, directories, ports, and system functions. 
Run;
```bash
sudo chown -R www-data:www-data /var/www/html/wordpress
sudo chmod -R 755 /var/www/html/wordpress
sudo ufw allow "Apache Full"
```

### Step 8: Access WordPress

Open a browser and navigate to:

```
http://<Web-Server-Public-IP>
```

## Conclusion

This project successfully implemented a scalable, secure, and efficient WordPress deployment on AWS using a three-tier architecture. By separating the Web Server (Apache, PHP, WordPress) and Database Server (MySQL) on individual EC2 instances, we enhanced performance, security, and maintainability. The implementation of Logical Volume Management (LVM) allowed dynamic storage allocation, ensuring efficient disk space utilization.

Security measures, including firewall rules, SELinux policies, and restricted database access, were enforced to safeguard the infrastructure. The successful configuration of remote database connectivity, Apache web server, and MySQL database integration ensured seamless communication between application layers.

Future improvements could include auto-scaling, load balancing, and automated backups to enhance reliability and fault tolerance.

