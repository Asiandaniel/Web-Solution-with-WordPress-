# Web-Solution-with-WordPress-




# 1. Web Solution Implementation Using WordPress on AWS EC2

## 1.1. Table of Contents
1. [Introduction](#1-2-introduction)
2. [Prerequisites](#1-3-prerequisites)
3. [Architecture Overview](#1-4-architecture-overview)
4. [Setting Up AWS EC2 Instances](#1-5-step-by-step-guide)
5. Configuring EBS and Logical Volume Manager (LVM) web-server
    - [EBS Setup](#1-5-1-launch-an-ec2-instance)
    - [LVM Configuration](#1-5-2-configure-security-group)
6. [Configuring EBS and Logical Volume Manager (LVM) db-server](#1-5-step-by-step-guide)
    - EBS Setup
    - LVM Configuration
7. AWS Security Group Configuration
8. Install and Configure MYSQL Database server
9. Installing Apache, PHP 8.3, and PHP Extensions on web-server instance
    - Step 1: Verify RHEL Version
    - Step 2: Install Apache (httpd)
    - Step 3: Enable Necessary Repositories
    - Step 4: Install PHP 8.3 and Extensions
    - Step 5: Configure SELinux (Security-Enhanced Linux)
    - Step 6: Verify PHP Installation
    - Step 7: Test PHP Functionality
10. WordPress Installation
11. Final Steps and Reflections

This project entails configuring WordPress on an EC2 instance with Red Hat Enterprise Linux (RHEL) 9.4, utilizing three EBS volumes (each 10GB) distributed across two different instances:

- **web-server**: Hosts the WordPress application.
- **db-server**: Runs the MySQL database.

The web-server will be at least a t2.small instance, while the db-server will use a t2.micro instance.
.
.
IMAGE
.
.
.
# Prerequisites
Before starting, I ensured:
- Two AWS EC2 instances (for WordPress and MySQL) running Red Hat Enterprise Linux 9.4.
- Familiarity with basic AWS EC2 and LVM concepts.
- SSH access to both instances.
- Minimum instance size: t2.small for the web-server instance (to avoid issues with package installations).


# Architecture Overview

Web-Server Instance:

- Instance Type: t2.small
- 3 EBS Volumes: 10GB each (/dev/xvdb, /dev/xvdc, /dev/xvdd)

DB-Server Instance:

- Instance Type: t2.micro
- 3 EBS Volumes: 10GB each (/dev/xvdb, /dev/xvdc, /dev/xvdd)

* Each server will use Logical Volume Manager (LVM) to dynamically manage the attached EBS volumes for application* *data and logs. Make sure the EBS volume blocks are created in the same availability zone as the instance they will be attached to.*
  *also note that  Using LVM allows dynamic scaling and better management of storage volumes without downtime*
  
![image](https://github.com/user-attachments/assets/1fcd2201-f6a5-4caa-9f00-35a73335ff3d)


# Setting Up AWS EC2 Instances

1 Launch EC2 Instances:

- Created two EC2 instances running Red Hat Enterprise Linux 9.4 for WordPress and MySQL, ensuring the instance for the webserver is at least t2.small and take note of the availability zone used in the instance creation.

2 Attach EBS Volumes:

- Added three EBS volumes to each instance for managing application data and logs.
To be able to identify the EBS, name them as follows in the AWS console:

Web Server:

- web-server-root - for the default EBS attached to the web-server instance when created
- web-server-vol-1 - for the xvdb attached to the web-server
- web-server-vol-2 - for the xvdc attached to the web-server
- web-server-vol-3 - for the xvdd attached to the web-server

DB Server:

- db-server-root - for the default EBS attached to the db-server instance when created
- db-server-vol-1 - for the xvdb attached to the db-server
- db-server-vol-2 - for the xvdc attached to the db-server
- db-server-vol-3 - for the xvdd attached to the db-server

.
.
.
.
.
IMAGE 
.
.
.
.

.
# Configuring EBS and Logical Volume Manager (LVM) web-server
Connect to the web-server instance via the ssh terminal to have access to the system.

 ## EBS Setup
- Check Volume Visibility: After SSH'ing into the instances, I listed the attached block devices using:
```lsblk```
The new volumes appeared as /dev/xvdb, /dev/xvdc, and /dev/xvdd.
.
.
.
.
.
IMAGE 
.
.
.
.
```note down the name of the EBB volumes as shown on the output of the lsblk command```

# LVM Configuration
1. Install LVM Tools: Since Red Hat Enterprise Linux 9.4 was being used, I ensured LVM was installed, I also installed nano (text editor) and wget (for downloading)
 ```
sudo dnf update
sudo dnf install lvm2 nano wget
```
you can check for available partition using lvmdiskscan command, however, since it is a fresh EBS volumes, there is no partition on it at the moment.

2.  Create partitions on EBS volumes:
  Using the gdisk utitlity to create a single partition on each EBS block as follows:

```
sudo gdisk /dev/xvdb
```
This will launch the partition utility interface as shown below:

.
.
.
.
.
IMAGE 
.
.
.
.
use the following commands to create and save the partition table to disk:

- n - To create a partition table, accept all defaults by pressing Enter key
- p - Print partition table informatoon
- w - Write changes to disk

.
.
.
.
.
IMAGE 
.
.
.
.
You do the same for the remaining EBS blocks using the commands below:
- xvdc block
  ```sudo gdisk /dev/xvdc ```
  xvdd block
  ```sudo gdisk /dev/xvdd```
confirm the partitions using lsblk

 ```lsblk```

   .
.
.
.
.
IMAGE 
.
.
.
.
note down the partition names: xvdb1, xvdc1 and xvdd1

3.  Create Physical Volumes: Converted the three attached EBS block partitions into physical volumes (PVs):
    ```  sudo pvcreate /dev/xvdb1 /dev/xvdc1 /dev/xvdd1 ```
4. Create Volume Group: Next, Created a volume group (VG) to aggregate the PVs:
 
```sudo vgcreate webdata-vg /dev/xvdb1 /dev/xvdc1 /dev/xvdd1```

5. Create Logical Volumes: Created logical volumes (LVs) for storing WordPress application data and logs. Here's how I did it:
```
sudo lvcreate -n app-lv -L 14G  webdata-vg
sudo lvcreate -n log-lv -L 14G  webdata-vg
lsblk
```
.
.
.
.
.
IMAGE 
.
.
.
.
To verify the entire setup - view the VG, PV and LV, you can run:
``` sudo vgdisplay -v ```

6. Create File system and Mount LVs: Create o a file system on the Logival volumes by running the follwoing commands:
```
sudo mkfs -t ext4 /dev/webdata-vg/app-lv
sudo mkfs -t ext4 /dev/webdata-vg/log-lv
```

.
.
.
.
.
IMAGE 
.
.
.
.
Then mounted them:

- Create the following locations:
```sudo mkdir -p /var/www/html /home/recovery/logs```
*note the -p is to ensure the parent directory are created if not existing.*

*The /home/recovery/logs directory is to backup the /var/log directory before mounting the /dev/webdata-vg/log-lv to the location as the mount process will wipe the location clean.*

- Backup the /var/log:

```sudo rsync -av /var/log/ /home/recovery/logs/```

The -av flag in the rsync command has the following meanings:

- a (archive): This option enables archive mode, which ensures that the data is copied recursively and preserves symbolic links, file permissions, timestamps, and ownership. It's a commonly used option when making backups or transfers where you want to keep the file attributes intact.

- v (verbose): This option enables verbose mode, meaning that rsync will display detailed information about what files are being transferred during the operation. It helps to see the progress and details of the copy process.

- Mount the LVs:
 ```
sudo mount /dev/webdata-vg/app-lv /var/www/html
sudo mount /dev/webdata-vg/log-lv /var/log
```
Restore the /var/log:
```
sudo rsync -av /home/recovery/logs/ /var/log/ 
```
7 Persistent Mounts: To ensure the volumes are automatically mounted at boot, updated /etc/fstab:
- Get the UUID of the LVs
```
sudo blkid
```
.
.
.
.
.
IMAGE 
.
.
.
.
- Update the /etc/fstab:
```
sudo nano /etc/fstab
```
- Paste the code below:
```
# mounts for wordpress webserver
UUID=c5e7c435-3141-4ea9-b410-62464d1cc478 /var/www/html ext4 defaults 0 0
UUID=88fb1766-b4d9-4931-87e7-99447eb6cc24 /var/log ext4 defaults 0 0
```
- Save and reload the daemon:
```
sudo systemctl daemon-reload
```
Test the mount:

```
sudo mount -a # if no output, the mount is successfully added to the fstab
```
# Configuring EBS and Logical Volume Manager (LVM) db-server:
Connect to the db-server instance via the ssh terminal to have access to the system.

## EBS Setup
1. Check Volume Visibility: After SSH'ing into the instances, I listed the attached block devices using:
   ```
   lsblk
   ```
   The new volumes appeared as /dev/xvdb, /dev/xvdc, and /dev/xvdd.
   .
.
.
.
.
IMAGE 
.
.
.
.
*note down the name of the EBB volumes as shown on the output of the lsblk command*

# LVM Configuration
1. Install LVM Tools: Since Red Hat Enterprise Linux 9.4 was being used, I ensured LVM was installed, I also installed nano (text editor) and wget (for downloading)
```
sudo dnf update
sudo dnf install lvm2 nano
```
you can check for available partition using lvmdiskscan command, however, since it is a fresh EBS volumes, there is no partition on it at the moment.

2. Create partitions on EBS volumes:

Using the gdisk utitlity to create a single partition on each EBS block as follows:

```sudo gdisk /dev/xvdb```

This will launch the partition utility interface as shown below:

.
.
.
.
.
IMAGE 
.
.
.
.
use the following commands to create and save the partition table to disk:

- n - To create a partition table, accept all defaults by pressing Enter key
- p - Print partition table informatoon
- w - Write changes to disk

.
.
.
.
.
IMAGE 
.
.
.
.
You do the same for the remaining EBS blocks using the commands below:

- xvdc block:
```
sudo gdisk /dev/xvdc
```
- xvdd block:

```
sudo gdisk /dev/xvdd
```
confirm the partitions using lsblk
```
lsblk
```

.
.
.
.
.
IMAGE 
.
.
.
.
*note down the partition names: xvdb1, xvdc1 and xvdd1*

3. Create Physical Volumes: Converted the three attached EBS block partitions into physical volumes (PVs):

  `sudo pvcreate /dev/xvdb1 /dev/xvdc1 /dev/xvdd1`
  
4. Create Volume Group: Next, Created a volume group (VG) to aggregate the PVs:

   `sudo vgcreate dbdata-vg /dev/xvdb1 /dev/xvdc1 /dev/xvdd1`
   
5. Create Logical Volumes: Created logical volumes (LVs) for storing WordPress application data and logs. Here's how I did it:

```
sudo lvcreate -n db-lv -L 14G  dbdata-vg
sudo lvcreate -n log-lv -L 14G  dbdata-vg
lsblk
```

.
.
.
.
IMAGE 
.
.
.
.

To verify the entire setup - view the VG, PV and LV, you can run:

```sudo vgdisplay -v```

6. Create File system and Mount LVs: Create o a file system on the Logival volumes by running the follwoing commands:


```
sudo mkfs -t ext4 /dev/dbdata-vg/db-lv
sudo mkfs -t ext4 /dev/dbdata-vg/log-lv
```
.
.
.
.
IMAGE 
.
.
.
.
Then mounted them:

- Create the following locations:
```sudo mkdir -p /db /home/recovery/logs```

*note the -p is to ensure the parent directory are created if not existing.*

*The /home/recovery/logs directory is to backup the /var/log directory before mounting the /dev/dbdata-vg/log-lv to the location as the mount process will wipe the location clean.*

- Backup the /var/log:
```
sudo rsync -av /var/log/ /home/recovery/logs/
```
*The -av flag in the rsync command has the following meanings:*

- a (archive): This option enables archive mode, which ensures that the data is copied recursively and preserves symbolic links, file permissions, timestamps, and ownership. It's a commonly used option when making backups or transfers where you want to keep the file attributes intact.

- v (verbose): This option enables verbose mode, meaning that rsync will display detailed information about what files are being transferred during the operation. It helps to see the progress and details of the copy process.

- Mount the LVs:
```
sudo mount /dev/dbdata-vg/db-lv /db
sudo mount /dev/dbdata-vg/log-lv /var/log
```
- Restore the /var/log:

```
sudo rsync -av /home/recovery/logs/ /var/log/
```
Persistent Mounts: To ensure the volumes are automatically mounted at boot, updated /etc/fstab:

- Get the UUID of the LVs:

```sudo blkid```
.
.
.
.
IMAGE 
.
.
.
.
- Update the /etc/fstab:
```sudo nano /etc/fstab```
- Paste the code below:
```
# mounts for wordpress webserver
UUID=c5e7c435-3141-4ea9-b410-62464d1cc478 /db ext4 defaults 0 0
UUID=88fb1766-b4d9-4931-87e7-99447eb6cc24 /var/log ext4 defaults 0 0
```
Save and reload the daemon:
```sudo systemctl daemon-reload```
Test the mount:
```sudo mount -a # if no output, the mount is successfully added to the fstab```

*Insight: Configuring LVM provided flexibility to scale storage without downtime. For WordPress, separate volumes for data and logs made it easier to manage and monitor disk usage.*

#AWS Security Group Configuration

Security groups were configured to control access between the WordPress and MySQL instances.

1. MySQL Security Group:

- Allowed traffic on port 3306 from the private IP of the web-server instance for database communication.
- Opened SSH (port 22) for administrative access.

2. WordPress Security Group (using the default Security group):

- Opened HTTP (port 80) for public access.
- Opened SSH (port 22) for administrative access.

# Install and Configure MYSQL Database server

While still inside the db-server, let setup the mysql database:
```
sudo dnf update
sudo dnf install mysql-server -y
```
# Initial Configuration

2. Started and enabled the MySQL service:
```
sudo systemctl start mysqld
sudo systemctl enable mysqld
```
Set the root password:
```
sudo mysql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Password.1';
FLUSH PRIVILEGES;
EXIT;
```
4. Ran the secure installation script:
```
sudo mysql_secure_installation
```
*Personal Note: I initially ran the mysql_secure_installation script without setting the root password first. This locked me out of the root account, leading to a valuable lesson on the importance of following the correct sequence of steps.*

5. Verified the installation:
```sudo systemctl status mysqld```

6. Create admin user for the wordpress application:
```sudo mysql -u root -p```
```
CREATE DATABASE wordpress;
CREATE USER 'myuser'@'172.31.3.201' IDENTIFIED WITH mysql_native_password BY 'Password.1';
GRANT ALL PRIVILEGES ON wordpress.* TO 'myuser'@'172.31.3.201';
FLUSH PRIVILEGES;
EXIT;
```
7. Test the remote connection to the newly created database via web-server instance: Connect to the web-server instance via the ssh interface and run this command:

- Install the MySQL client:
```sudo dnf install mysql```

- Connect to the remote mysql server:
```sudo mysql -h 172.31.3.113 -u myuser -p```
if you can connect into the mysql shell, then the setup was successful.
.
.
.
.
IMAGE 
.
.
.

# Installing Apache, PHP 8.3, and PHP Extensions on web-server instance
## Step 1: Verify RHEL Version
Before starting, ensure you are running RHEL 9.4. This can be done with the following command:

```cat /etc/redhat-release```
Expected output:

```Red Hat Enterprise Linux release 9.4 (Plow)```
## Step 2: Install Apache (httpd)
WordPress requires a web server to handle HTTP requests, and Apache is the most commonly used web server for WordPress installations.

1. Install Apache using the dnf package manager:

```sudo dnf install httpd```

2. Start and enable Apache to ensure it runs on boot:
```
sudo systemctl start httpd
sudo systemctl enable httpd
```

3. Check that Apache is running:
```
sudo systemctl status httpd
```

4. Access the web browser: http://your-server-ip

If everything is configured correctly, you should see the default redhat page.
