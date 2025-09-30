# DEVOPS TOOLING WEBSITE SOLUTION

## TASK: Implement a 2-tier web application architecture with MySQL database and NFS shared storage


## Prerequisites:
**Infrastructure**: AWS Cloud Provider
**Webserver Linux**: Red Hat Enterprise Linux 9
**Database Server**: Ubuntu 24.04 + MySQL
**Storage Server**: Red Hat Enterprise Linux 9 + NFS Server
**Programming Language**: PHP Scripting Language
**Code Repository**: GitHub
- Open ports in security groups:

HTTP (80), HTTPS (443), SSH (22) for web servers

MySQL (3306) for database server

NFS (2049, 111 TCP/UDP) for NFS server

## ENVIRONMENT:
- **NFS Server**: 13.49.46.137 (172.31.19.59) - RHEL 9
- **MySQL Server**: 16.170.232.158 (172.31.41.146) - Ubuntu
- **Web Server 1**: 13.51.237.76 (172.31.20.187) - RHEL 9
- **Web Server 2**: 51.20.92.125 (172.31.28.1) - RHEL 9
- **Web Servers Subnet CIDR**: 172.31.16.0/20

## STEP ONE: SETTING UP NFS SERVER

*Created t3.micro RHEL 9 instance with 3 EBS volumes*
*Configured LVM with apps-lv, logs-lv, opt-lv*

```bash
# SSH to NFS server
ssh -i your-key.pem ec2-user@13.49.46.137
```

# Create partitions on attached EBS volumes
```bash
sudo gdisk /dev/nvme1n1
sudo gdisk /dev/nvme2n1  
sudo gdisk /dev/nvme3n1
```
# Reason: Partitioning is required for creating physical volumes for LVM

# Create LVM physical volumes and volume group
```bash
sudo pvcreate /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
```
# Reason: LVM allows flexible storage management

# Create logical volumes for apps, logs, and optional storage
```bash
sudo lvcreate -n apps-lv -L 10G webdata-vg
sudo lvcreate -n logs-lv -L 10G webdata-vg
sudo lvcreate -n opt-lv -L 10G webdata-vg
```
# Format LVs with XFS filesystem

```bash
sudo mkfs.xfs /dev/webdata-vg/apps-lv
sudo mkfs.xfs /dev/webdata-vg/logs-lv
sudo mkfs.xfs /dev/webdata-vg/opt-lv
```
# Create mount points and mount volumes
```bash
sudo mkdir /mnt/apps /mnt/logs /mnt/opt
sudo mount /dev/webdata-vg/apps-lv /mnt/apps
sudo mount /dev/webdata-vg/logs-lv /mnt/logs
sudo mount /dev/webdata-vg/opt-lv /mnt/opt
```

# Install and configure NFS
```bash
sudo yum install nfs-utils -y
sudo systemctl start nfs-server
sudo systemctl enable nfs-server
```

# Set permissions for shared directories
```bash
sudo chown -R nobody: /mnt/apps /mnt/logs /mnt/opt
sudo chmod -R 777 /mnt/apps /mnt/logs /mnt/opt
```
# Reason: Ensure NFS clients can read/write to shared directories

# Configure exports
```bash
sudo vi /etc/exports
```

# Add the following lines:
```bash
/mnt/apps 172.31.16.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/logs 172.31.16.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/opt 172.31.16.0/20(rw,sync,no_all_squash,no_root_squash)
```

```bash
sudo exportfs -arv
# Reason: Apply export rules and allow web servers to mount directories
```

- Check the NFS ports and open it in the security group inbound rules , add the ports and allow access from the subnet cidr ipv4 address

```bash
rpcinfo -p | grep nfs
```


## STEP TWO: CONFIGURING MYSQL SERVER
Goal: Provide centralized database access for web servers.
*Ubuntu instance with MySQL installed and configured*

```bash
# SSH to MySQL server
ssh -i your-key.pem ubuntu@16.170.232.158
```

# Install MySQL Server
```bash
sudo apt update
sudo apt install mysql-server -y
# Reason: Required to host the tooling database
```

#Check status:
```bash
sudo systemctl status mysql
#Should show active (running).
```

# Secure MySQL installation
```bash
sudo mysql_secure_installation
```
# Set a root password (if not already).
# Remove anonymous users, disallow remote root login, remove test DB, reload privileges.


# Create database and user for web access
```bash
sudo mysql
CREATE DATABASE tooling;
CREATE USER 'webaccess'@'172.31.16.0/20' IDENTIFIED BY 'StrongPass123!';
GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'172.31.16.0/20';
FLUSH PRIVILEGES;
EXIT;
# Reason: Provides restricted database access to web servers only
```

# Enable remote access
```bash
sudo sed -i 's/bind-address.*=.*127.0.0.1/bind-address = 0.0.0.0/' /etc/mysql/mysql.conf.d/mysqld.cnf
sudo systemctl restart mysql
# Reason: Allows MySQL server to accept connections from web servers
```

## STEP THREE: SETUP THE WEBSERVERS

*Two RHEL 9 instances configured as web servers*

```bash
# For each web server (13.51.237.76 and 51.20.92.125)
ssh -i your-key.pem ec2-user@13.51.237.76
ssh -i your-key.pem ec2-user@51.20.92.125
```

# Install NFS client and mount shared directories

# Install NFS client (do this for the two webservers that would be client to the NFS server)
```bash
sudo yum install nfs-utils nfs4-acl-tools -y
```

# Make /var/www/ directory
```bash
sudo mkdir /var/www
```

# Mount /var/www/ and target the NFS server export for apps
```bash
sudo mount -t nfs -o rw,nosuid 172.31.19.59:/mnt/apps /var/www
```

# make sure the changes persist after reboot
```bash
sudo vi /etc/fstab
# add the following
172.31.19.59:/mnt/apps /var/www nfs defaults 0 0
# save and exit
```

# install and configure REMI repository, apache, as well as php and all dependencies
# Install Apache and PHP
```bash
sudo yum install httpd -y
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm -y
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm -y
sudo dnf module reset php -y
sudo dnf module enable php:remi-7.4 -y
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd -y
# # Reason: Required to run PHP web applications
```

# Start and enable PHP-FPM
```bash
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo setsebool -P httpd_execmem 1
# Reason: Allows Apache to execute PHP scripts safely
```

# Verify that both the webservers /var/www and NFS servers /mnt/apps have the same files and directories, to do this,
```bash
ls /var/www #on the webservers
ls /mnt/apps #on the nfs server
```
```bash
df -h
```

# make /var/log/httpd/ directory
```bash
sudo mkdir /var/log/httpd
```

# Mount /var/log/httpd and target the NFS server export for logs
```bash
sudo mount -t nfs -o rw,nosuid 172.31.19.59:/mnt/logs /var/log/httpd
```

# make sure the changes persist after reboot
```bash
sudo vi /etc/fstab
```

# add the following:
```bash
172.31.19.59:/mnt/logs /var/log/httpd nfs defaults 0 0
# save and exit
```
# Create a file in one webserver and verify it appears on the other webservers
```bash
sudo touch /var/www/test.txt
ls /var/www  #(On both servers)
```

# Start Apache
```bash
sudo systemctl start httpd
sudo systemctl enable httpd
```

## STEP FOUR: TEST MYSQL REMOTE CONNECTION

```bash
# From web servers, test database connection
sudo yum install mysql -y
mysql -h 172.31.41.146 -u webaccess -p -e "SHOW DATABASES;"
# Password: StrongPass123!
# Reason: Verify that web servers can access MySQL database
```

## STEP FIVE: DEPLOYING THE WEBSITE

```bash
# On each web server
# Clone website repository
sudo yum install git -y
git clone https://github.com/dimani001/tooling.git
sudo cp -R tooling/html/* /var/www/html/
sudo chown -R apache:apache /var/www/html
sudo chmod -R 755 /var/www/html
```
# Reason: Deploy web files with correct permissions

# Configure database connection
```bash
sudo vi /var/www/html/functions.php
# Update: $db = mysqli_connect('172.31.41.146', 'webaccess', 'StrongPass123!', 'tooling');
```

# Import database schema
```bash
mysql -h 172.31.41.146 -u webaccess -p tooling < tooling/tooling-db.sql
# Password: StrongPass123!
```

# Create a new admin user and password, to do this, connect to mysql remotely
```bash
sudo mysql -u webaccess -p -h 172.31.41.146
USE DATABASE tooling;
INSERT INTO `users` (`id`, `username`, `password`, `email`, `user_type`, `status`) VALUES
(1, 'myuser', '5f4dcc3b5aa765d61d8327deb882cf99', 'user@mail.com', 'admin', '1');
exit
```

# Configure SELinux for NFS
```bash
sudo setsebool -P httpd_use_nfs 1
sudo setsebool -P httpd_can_network_connect 1
```

# Restart Apache
```bash
sudo systemctl restart httpd
```

## FINAL TESTING

1. **Access website**: http://13.51.237.76/index.php
2. **Login credentials**:
   - Username: `myuser`
   - Password: `password`
3. **Verify both web servers** are accessible

## SECURITY GROUP CONFIGURATION
- **Web servers**: Allow HTTP (80), HTTPS (443), SSH (22) from 0.0.0.0/0
- **MySQL**: Allow MySQL (3306) from 172.31.16.0/20
- **NFS**: Allow NFS ports (2049, 111 TCP/UDP) from 172.31.16.0/20

## CONGRATULATIONS!
*Successfully implemented a web solution for DevOps team using LAMP stack with remote Database and NFS server!*

**Login Credentials**: Username: `myuser`, Password: `password`
**Database Access**: User: `webaccess`, Password: `StrongPass123!`
**NFS Server**: 172.31.19.59
**MySQL Server**: 172.31.41.146

*All servers are properly configured with shared storage, database connectivity, and load balancing capability across multiple web servers.*
Testing Jenkins webhook
