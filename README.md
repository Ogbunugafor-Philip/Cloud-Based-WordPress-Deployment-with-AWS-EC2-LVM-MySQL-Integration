# Cloud-Based WordPress Deployment with AWS EC2, LVM & MySQL Integration

## Introduction

In today's digital landscape, businesses and individuals rely on scalable and secure web hosting solutions to ensure high availability and performance. This project focuses on deploying WordPress, a widely used Content Management System (CMS), on Amazon Web Services (AWS) using a three-tier architecture.

The deployment is designed to achieve high availability, efficient storage management, and secure database connectivity by leveraging AWS EC2 instances, Logical Volume Management (LVM) for dynamic storage allocation, and a dedicated MySQL database server. The architecture follows best practices for cloud-based web hosting, ensuring a robust and maintainable environment.

## Project Objectives

- Deploy a scalable three-tier architecture for WordPress on AWS, utilizing separate EC2 instances for the Web Server and Database Server.
- Implement Logical Volume Management (LVM) on both servers to enable efficient storage management and scalability.
- Establish secure database connectivity between the Web Server (Apache, PHP, WordPress) and the Database Server (MySQL) using security best practices.
- Optimize system performance, security, and high availability by configuring Apache, MySQL, firewall rules, and SELinux policies.

## Three-Tier Architecture

The Three-Tier Architecture is a widely used software architecture pattern for web and mobile applications. It consists of three distinct layers, each responsible for a specific function:

- **Presentation Layer (PL):** This is the user interface layer, where users interact with the application through a web browser or client application.
- **Business Layer (BL):** This is the application or web server that processes user requests and executes business logic.
- **Data Access Layer (DAL):** This is where data is stored and managed, typically in a database or file system.

### Three-Tier Architecture Setup for WordPress Deployment

This project implements a three-tier architecture using AWS EC2 instances to ensure scalability, security, and performance. The setup consists of:

1. **Client (Presentation Layer):** A laptop or PC used to access the WordPress site through a web browser.
2. **Web Server (Business Layer):** An EC2 Linux Server that hosts the WordPress application.
3. **Database Server (Data Access Layer):** A separate EC2 Linux Server running MySQL to store and manage WordPress data.

## Project Steps

### Step 1: Prepare the Web Server (EC2 Instance)
- Launch an EC2 instance for the Web Server.
- Create and attach three EBS volumes (10 GiB each).
- Verify the attached storage devices using:
  ```bash
  lsblk
  ```

### Step 2: Configure Storage Using LVM on the Web Server
- Partition the new disks using `gdisk`.
  ```bash
  sudo gdisk /dev/xvdbb
  sudo gdisk /dev/xvdbc
  sudo gdisk /dev/xvdbd
  ```
- Verify partitions:
  ```bash
  lsblk
  ```
- Install and configure LVM:
  ```bash
  sudo apt update
  sudo apt install -y lvm2
  ```
- Scan for available disks:
  ```bash
  sudo lvmdiskscan
  ```
- Initialize disks as LVM Physical Volumes:
  ```bash
  sudo pvcreate /dev/xvdbb1 /dev/xvdbc1 /dev/xvdbd1
  ```
- Verify Physical Volumes:
  ```bash
  sudo pvs
  ```
- Create a Volume Group:
  ```bash
  sudo vgcreate webdata-vg /dev/xvdbb1 /dev/xvdbc1 /dev/xvdbd1
  ```
- Create Logical Volumes:
  ```bash
  sudo lvcreate -n apps-lv -L 14G webdata-vg
  sudo lvcreate -n logs-lv -L 14G webdata-vg
  ```
- Format the Logical Volumes:
  ```bash
  sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
  sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
  ```
- Mount the directories:
  ```bash
  sudo mkdir -p /var/www/html
  sudo mount /dev/webdata-vg/apps-lv /var/www/html/
  ```

### Step 3: Prepare the Database Server (EC2 Instance)
- Launch a second EC2 instance for the Database Server.
- Create and attach one EBS volume (10 GiB).
- Configure LVM storage and mount `db-lv` to `/db`.

### Step 4: Install and Configure MySQL on the Database Server
- Install MySQL Server:
  ```bash
  sudo yum install mysql-server
  ```
- Start and enable MySQL:
  ```bash
  sudo systemctl restart mysqld
  sudo systemctl enable mysqld
  ```
- Create a WordPress database and user:
  ```sql
  CREATE DATABASE wordpress;
  CREATE USER 'wordpress-user'@'172.31.92.14' IDENTIFIED BY 'osita1989';
  GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress-user'@'172.31.92.14';
  FLUSH PRIVILEGES;
  ```

### Step 5: Install and Configure WordPress on the Web Server
- Install Apache and PHP:
  ```bash
  sudo apt update && sudo apt upgrade -y
  sudo apt install apache2 -y
  sudo apt install -y php php-mysql php-fpm php-opcache php-gd php-xml php-mbstring
  ```
- Download and configure WordPress:
  ```bash
  sudo wget http://wordpress.org/latest.tar.gz
  sudo tar -xzvf latest.tar.gz
  sudo cp -R wordpress/* /var/www/html/
  ```

### Step 8: Access WordPress from a Web Browser
- Open a browser and go to: `http://<Web-Server-Public-IP>`.
- Complete the WordPress installation wizard.

## Conclusion

This project successfully deployed a scalable, secure, and efficient WordPress hosting environment on AWS using a three-tier architecture. The use of Logical Volume Management (LVM) ensured efficient storage allocation, and best security practices were implemented for database connectivity and web server protection. Future improvements could include auto-scaling, load balancing, and automated backups to enhance reliability and fault tolerance.
