


# Step 0: Prerequisites

__1.__ Establish 2 EC2 Instance of t2.medium family type and Red Hat was lunched in the region of my choosing using the AWS console.

__2.__ The security group for both instances was configured with the following inbound rules:

- Allow traffic on port 80 (HTTP) with source from anywhere on the internet.
- Allow traffic on port 22 (SSH) with source from any IP address. This is opened by default.
  
  ![](/images/Screenshot%20(507).png)

# Step 1: Prepare a Web Server

## Create and Attach EBS Volumes

#### Create EBS Volumes
__1.__  Go to the **Elastic Block Store > Volumes** section in the AWS Management Console.

__2.__ Click **Create Volume**.
Configure the volume:
  - Volume type: General Purpose SSD (gp2)
  - Size: 10 GB
 - Availability Zone: Same as the  EC2 instances we created.
- Click **Create Volume**.
  
   ![ebs_volume](/images/Screenshot%20(508).png)

**NOTE:** Repeat the above steps to create 2 more volumes.

#### Attach EBS Volumes to Web Server Instance
1. Go to the **Elastic Block Store > Volumes** section.
2. Select the first volume, click **Actions** > **Attach Volume**.
3. Select the server instance from the list and specify a device name (e.g., `/dev/sdb`).
4. Click **Attach**.
5. Repeat the above steps to attach the other two volumes to the server instance using device names `/dev/sdc` and `/dev/sdd`.

![Attach_Volume](/images/Screenshot%20(509).png)
![attach_volume_outcome](/images/Screenshot%20(510).png)

# Step 2: Connect to the Instances and Prepare Volumes


__1.__ Open your terminal and connect to the server instance:
```bash
ssh -i "keypair" ec2-user@ec2-34-224-95-90.compute-1.amazonaws.com
```
__2.__ Inspect what block devices are attached to the server. 
```bash
lsblk
```
**NOTE:** Notice names of your newly created devices.

![lsblk](/images/Screenshot%20(511).png)

__3.__  See all mounts and free space on your server by running the following command:
```bash
df -h
```
![](/images/Screenshot%20(512).png)

__4.__ Use `gdisk` to create a single partition on each of the 3 disks

```bash
sudo gdisk /dev/xvda4
```
**NOTE:** To create a partition, follow these steps within gdisk:

- Type n to create a new partition.
- Press Enter to accept the default partition number.
- Press Enter to accept the default first sector.
- Press Enter to accept the default last sector, using the entire disk.
- Type w to write the changes and exit.
![partition1](/images/Screenshot%20(514).png)


```bash
sudo gdisk /dev/xvda3
```
![partition2](/images/Screenshot%20(515).png)


```bash
sudo gdisk /dev/nvme3n1
```
![partition3](/images/Screenshot%20(516).png)

__5.__ Use `lsblk` utility to view the newly configured partition on each of the 3 disks.

```bash
lsblk
```
![veiw_all_partition](/images/Screenshot%20(517).png)

__6.__ Install `lvm2` package. Lvm2 is used for managing disk drives and other storage devices

```bash
sudo yum install lvm2
```
![install lvn2](/images/Screenshot%20(518).png)

__7.__ Use the `pvcreate` utility tool to mark each of the volumes as physical volumes to be used by LVM

```bash
sudo pvcreate /dev/xvdb
sudo pvcreate /dev/xvdf
sudo pvcreate /dev/xvdbf
```

![pvcreate_physical_volume](/images/Screenshot%20(519).png)

__8.__ Verify that the physical volume has been created by running 
```bash
sudo pvs
```
![](/images/Screenshot%20(520).png)

__9.__ Use `vgcreate` utility to add all 3 PVs to a volume group. Name the VG `webdata-vg`
```bash
sudo vgcreate webdata-vg /dev/xvdb  /dev/xvdf  /dev/xvdbf
```
__10.__ Verify that your `VG` has been created sucessfully by running 
```bash
sudo vgs
```
![](/images/Screenshot%20(521).png)

__11.__ Create 2 logical volumes, name one app-lv and the other logs-lv. For app-lv, use half of the disk size, then use the remaining part fpor the logs-lv
```bash
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
```
__12.__ Verify that the logical volumes has been created

![logical_volume](/images/Screenshot%20(522).png)

__13.__ Verify the entire setup to be sure all has been configured properly
```bash
sudo vgdisplay -v #view complete setup - VG, PV, and LV 
sudo lsblk 
```
![vgdisplay_](/images/Screenshot%20(523).png)

__14.__ Use `mkfs.ext4` to format the logical format with `ext4`filesystem.

```bash
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
 sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```
![](/images/Screenshot%20(524).png)

__15.__ Create `/var/www/html` directory to store website files
```bash
sudo mkdir -p /var/www/html
```

__16.__ Create `/home/recovery/logs` directory to store backup of log data
```bash
sudo mkdir -p /home/recovery/logs
```
![directories-](/images/Screenshot%20(525).png)

__17.__ Mount `/var/www/html` on `apps-lv`logical volume
```bash
sudo mount /dev/webdata-vg/apps-lv /var/www/html/
```
__18.__ Use `rsync `Utility to backup all the files in the log directory `/var/log `  into  `/home/recovery/logs`(This is requitred before mounting the file system).

```bash
sudo rsync -av /var/log/ /home/recovery/logs/
```
![M_B](/images/Screenshot%20(526).png)

__19.__ Mount the .var/logs on the log-lv
```bash
sudo mount /dev/webdata-vg/logs-lv   /var/log/
``` 
__20.__  Restore log files back ito `/var/log `  directory

```bash
sudo rsync -av /home/recovery/logs/ /var/log
```
![M&R](/images/Screenshot%20(527).png)

__21.__ Update `/etc/fstab` file so that the mount configuration will persist after restart of the server. The UUID of the device will be used to update the `/etc/fstab` file.
```bash
sudo blkid
```
![update_fstab](/images/Screenshot%20(528).png)

```bash
sudo vi /etc/fstab
```
Update /etc/fstab in this format using your own UUID and remember to remove the leading and ending quotes.

__22.__ Test the configuration and reload the daemon
```bash
sudo mount -a
sudo systemctl daemon-reload
```

__22.__ Verify your setup by running `df h`, output must look like this:
```bash
df -h
```
![](/images/Screenshot%20(529).png)

### STEP 2: Prepare the Database Server
Now, we're ready to install and configure the MySQL server, which will act as the database for our website on the server instance. To accomplish this, we'll replicate the process we used for the server instance, including creating an EC2 instance, attaching the three EBS volumes, and SSHing into the instance to create partitions.

Create logical volumes using the same process as we did in the server instance. The logical volume should be named `db-lv` instead of `apps-lv`. Additionally, create another logical volume named `logs-lv`. Mount the `db-lv` to the `/db/` directory.

### STEP 3: Install WordPress on your Web Server EC2
__1.__ Update the repositiory
```bash
sudo yum -y update
```
![](/images/Screenshot%20(530).png)

__2.__ Install wget, Apache and it's dependencies
```bash
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
```
![installApache](/images/Screenshot%20(531).png)

__3.__ Enable and start Apache
```bash
sudo systemctl enable httpd 
sudo systemctl start httpd
```

![start_apache](/images/Screenshot%20(532).png)

__4.__ Install PHP and it's dependencies.

```bash
  sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
    sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
    sudo yum module list php sudo yum module reset php
    sudo yum module enable php:remi-7.4
    sudo yum install php php-opcache php-gd php-curl php-mysqlnd
    sudo systemctl start php-fpm
    sudo systemctl enable php-fpm setsebool -P httpd_execmem 1
```
![](/images/Screenshot%20(533).png)

__5.__ Restart Apache
```bash
sudo systemctl restart httpd
```

__6.__ Create an info.php page to test if your configuration is correct
```bash
sudo vi /var/www/html/info.php
```

Write the following code to chek the php configuration
```bash
 <?php
  phpinfo();
  ?>
```
__7.__ Vist your IPaddress/info.php

![](/img/homepage_php.png)

__8.__ Download and Copy wordpress to the /var/www/html directory
```bash

sudo wget http://wordpress.org/latest.tar.gz

sudo tar xzvf latest.tar.gz

sudo rm -rf latest.tar.gz

sudo cp wordpress/wp-config-sample.php wordpress/wp-config.php
```

__9.__ Configure SElinux policies

```bash
sudo chown -R apache:apache /var/www/html/wordpress

sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R

sudo setsebool -P httpd_can_network_connect=1
```
![](/img/php_wordpress.jpg)

![](/img/apache_install.jpg)

__10.__ Install and configure mysql server on your DB ec2 instance

```bash
sudo yum -y update
sudo yum install mysql-server
```
![](/img/db_mysql_server.jpg)

__11.__ Verify that the service is up and running

```bash
sudo systemctl status mysqld
sudo systemctl restart mysqld
sudo systemctl enable mysqld
```
__12.__ Configure the DB to work with wordpress , you can do this by creating a DB user that is from the web server IP address.

First , log in as root user

```bash
sudo mysql
```
__13.__ Then create a new db

```bash
CREATE DATABASE wordpress;

CREATE USER 'myuser'@'my server ipaddress' IDENTIFIED BY 'choose your password' ;

GRANT ALL ON wordpress2.* TO 'citatech'@'yourwebserver ip address' ;

FLUSH PRIVILEGES;

SHOW DATABASES;
```
![](/img/wordpress_created.png)

Test your db connection by logging in to your db from your webserver, before that, ensure you allowed port 3306 (which is the default port for mysql) in your mysql instance inbound rules, configure the connection to your-webserver-IP-address/32

__14.__ Then access your webserver instance and also install mysql client

```bash
sudo yum install -y mysql-client
```
__15.__ Log in to the mysql server remotely from your webserver.

```bash
sudo mysql -u myuser -p -h (your mysql server ip address)
```
__16.__ If logged successfully, you should get the same result below. P.S, the image below shows both my webserver which is on the left and mysql server which is on the right.

![db_remote_connection](/img/DB_Remotely_connected.png)

**Now that you have successfully setup and configured mysql and connected to it remotely from your webserver, it is essential we set up wordpress to do the same.**

__17.__  Visit your-ip-address/wordpress in your web browser and you should get the same result as below;

![](/img/wordpress_installed.png)

## STEP FIVE

__1.__ Log into your webserver ec2 instance and access your wordpress directory.

```bash
    sudo cd /var/html/wordpress
```

__2.__ Configure wordpress to use your mysql database we created earlier, to do this , we have to access the wp-config file , this is where the configuration of wordpress is done.

```bash
sudo vi wp-config.php
```
- Replace the variables of DB user, DB host, DB password .
  
- DB host - this should be your mysql server ip address.
  
- DB password ; your database password.
  
- DB user ; the database user we created earlier

- Save and exit

- Visit your webserver IP 
  
![](/img/WP_Install.png)
```bash
address/wordpress directory
your-ip-address/wordpress
  ```

  and you should get the installation page below :

![](/img/connected%20_wordpress%20db.png)

- Follow the installation process and click on install, wait a few minutes and wordpress would have been successfully installed using your remote database on the mysql server
  
Site title for the site.
- Username: admin
- Password: admin_password
- Your Email: admin_email
  
Discourage search engines from indexing this site:

![wp_setup_complete
](/img/WP_Setup_Complete.png)

**In this documentation, we've mastered the process of creating and linking EBS volumes to our instance, partitioning and establishing logical volumes to house our WordPress website. Furthermore, we've successfully crafted a WordPress site, stored its files on our Apache web server, and hosted the database on a separate server, allowing us to remotely access it.**