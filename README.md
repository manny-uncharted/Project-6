# WEB SOLUTION WITH WORDPRESS

In this project we are tasked to prepare storage infrastructure on two Linux servers and implement a basic web solution using WordPress. WordPress is a free and open-source content management system written in PHP and paired with MySQL or MariaDB as its backend Relational Database Management System (RDBMS). 

This project consists of two parts:

- Configure storage subsystem for Web and Database servers based on Linux OS. The focus of this part is to give you practical experience of working with disks, partitions and volumes in Linux.

- Install WordPress and connect it to a remote MySQL database server. This part of the project will solidify your skills of deploying Web and DB tiers of Web solution.

As a DevOps engineer, a deep understanding of core components of web solutions and the ability to troubleshoot them will play essential role in your further progress and development.

## Three-tier Architecture
Generally, web, or mobile solutions are implemented based on what is called the Three-tier Architecture.

Three-tier Architecture is a client-server software architecture pattern that comprise of 3 separate layers. They are:
- <b>Presentation Layer (PL)</b>: This is the user interface such as the client server or browser on your laptop.
- <b>Business Layer (BL)</b>: This is the backend program that implements business logic. Application or Webserver
- <b>Data Access or Management Layer (DAL)</b>: This is the layer for computer data storage and data access. Database Server or File System Server such as FTP server, or NFS Server.

With this project, you will have the hands-on experience that showcases Three-tier Architecture while also ensuring that the disks used to store files on the Linux servers are adequately partitioned and managed through programs such as gdisk and LVM respectively.

Requirements:
1. <b>Your 3-Tier Setup</b>
    - A Laptop or PC to serve as a client
    - An EC2 Linux Server as a web server (This is where you will install WordPress)
    - An EC2 Linux server as a database (DB) server

<b>Note:</b> We are using RedHat OS for this project, you should be able to spin up an EC2 instance on your own. Also when connecting to RedHat you will need to use ec2-user user. Connection string will look like ec2-user@public-ip-address.


## Creating and mounting Volumes
- Create and attach a new volume to your Linux server.

    Results:
    ![](img/volume.png)
    Note: Ensure that the availability zone of your volume must be the same as your Linux server.

- Connect to your linux server and check if the volume is attached using this command:
    ```
    lsblk
    ```
    Results:
    ![](img/lsblk.png)



- Use df -h command to see all mounts and free space on your server
    ```
    sudo df -h
    ```
    Results:
    ![](img/df-h.png)

- Use gdisk utility to create a single partition on each of the 3 disks
    ```
    sudo gdisk /dev/xvdf
    ```
    and then use the w command and enter "y" to create a single partition.

    Also repeat the same for the other two disks
    ```
    sudo gdisk /dev/xvdg
    sudo gdisk /dev/xvdh
    ```
    Results:
    ![](img/gdisk1.png)

- Use lsblk utility to view the newly configured partition on each of the 3 disks.
    ```
    sudo lsblk
    ```
    Results:
    ![](img/lsblk2.png)

- Install lvm2 package using sudo yum install lvm2. Run sudo lvmdiskscan command to check for available partitions.
    ```
    sudo yum install lvm2
    sudo lvmdiskscan
    ```
    Results:
    ![](img/lvm2.png)
    ![](img/lvmdiskscan.png)
    Note: Unlike ubuntu that uses apt, for redhat the package manager is yum.

- Use pvcreate utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM
    ```
    sudo pvcreate /dev/xvdf
    sudo pvcreate /dev/xvdg
    sudo pvcreate /dev/xvdh
    ```
    Results:
    ![](img/pvcreate.png)

- Verify that your Physical volume has been created successfully by running sudo pvs
    ```
    sudo pvs
    ```
    Results:
    ![](img/pvs.png)

- Use vgcreate utility to add all 3 PVs to a volume group (VG). Name the VG webdata-vg
    ```
    sudo vgcreate webdata-vg /dev/xvdf2 /dev/xvdg2 /dev/xvdh2
    ```
    Results:
    ![](img/vgcreate.png)

- Use lvcreate utility to create 2 logical volumes. apps-lv (Use half of the PV size), and logs-lv Use the remaining space of the PV size. NOTE: apps-lv will be used to store data for the Website while, logs-lv will be used to store data for logs.
    ```
    sudo lvcreate -n apps-lv -L 14G webdata-vg
    sudo lvcreate -n logs-lv -L 14G webdata-vg
    ```
    Results:
    ![](img/lvcreate.png)

- Verify that your Logical Volume has been created successfully by running sudo lvs
    ```
    sudo lvs
    ```
    Results:
    ![](img/lvs.png)

- Verify the entire setup
    ```
    sudo vgdisplay -v #view complete setup - VG, PV, and LV
    sudo lsblk 
    ```
    Results:
    ![](img/vgdisplay.png)
    ![](img/lsblk3.png)

- Use mkfs.ext4 to format the logical volumes with ext4 filesystem
    ```
    sudo mkfs.ext4 /dev/webdata-vg/apps-lv
    sudo mkfs.ext4 /dev/webdata-vg/logs-lv
    ```
    Results:
    ![](img/mkfs2.png)

## Creating a directory structure.
- Create /var/www/html directory to store website files
    ```
    sudo mkdir -p /var/www/html
    ```
    Results:
    ![](img/mkdir-html.png)

- Create /home/recovery/logs to store backup of log data
    ```
    sudo mkdir -p /home/recovery/logs
    ```
    Results:
    ![](img/mkdir-logs.png)

- Mount /var/www/html on apps-lv logical volume
    ```
    sudo mount /dev/webdata-vg/apps-lv /var/www/html/
    ```
    Results:
    ![](img/mount-html.png)

- Use rsync utility to backup all the files in the log directory /var/log into /home/recovery/logs (This is required before mounting the file system)
    ```
    sudo rsync -av /var/log/. /home/recovery/logs/
    ```
    Results:
    ![](img/rsync.png)

- Mount /var/log on logs-lv logical volume. (Note that all the existing data on /var/log will be deleted. That is why step of creating /var/www/html directory to store website files)
    ```
    sudo mount /dev/webdata-vg/logs-lv /var/log
    ```
    Results:
    ![](img/mount-log.png)

- Restore log files back into /var/log directory
    ```
    sudo rsync -av /home/recovery/logs/. /var/log
    ```
    Results:
    ![](img/rsync2.png)


## UPDATING THE `/ETC/FSTAB` FILE
- Update /etc/fstab file so that the mount configuration will persist after restart of the server.
The UUID of the device will be used to update the /etc/fstab file;
    ```
    sudo blkid
    ```
    Results:
    ![](img/blkid.png)

- Update /etc/fstab in this format using your own UUID and rememeber to remove the leading and ending quotes.
    ```
    sudo nano /etc/fstab
    ```
    and add this
    ```
    UUID=<uuid of your webdata-vg-apps> /var/www/html ext4 defaults 0 0
    UUID=<uuid of your webdata-vg-logs> /var/log ext4 defaults 0 0
    ```
    Results:
    ![](img/fstab2.png)

- Test the configuration and reload the daemon
    ```
    sudo mount -a
    sudo systemctl daemon-reload
    ```
    Results:
    ![](img/mount-a2.png)

- Verify your setup by running df -h, output must look like this:
    ```
    sudo df -h
    ```
    Results:
    ![](img/df-h2.png)


## Preparing the Database Server
Launch a second RedHat EC2 instance that will have a role – ‘DB Server’
Repeat the same steps as for the Web Server, but instead of apps-lv create db-lv and mount it to /db directory instead of /var/www/html/.

- ssh into the instance you just created
    ```
    ssh -i <your_key_file> ec2-user@<public_ip_address>
    ```
    Results:
    ![](img/ssh-db.png)

- Create and attach 3 Logical Volumes to the database server instance.
    ![](img/volume.png)

- Connect to your linux server and check if the volume is attached using this command:
    ```
    sudo lsblk
    ```
    Results:
    ![](img/lsblk-db.png)

- Use df -h command to see all mounts and free space on your server
    ```
    sudo df -h
    ```
    Results:
    ![](img/df-h-db.png)

- Using gdisk utility create a single partition on each of the 3 disks
    ```
    sudo gdisk /dev/xvdf
    ```
    and then create a new partition with the "n" command, inputting 1 (to create a single partition) and then the "w" command and enter "y" to create a single partition.

    Also repeat the same for the other two disks
    ```
    sudo gdisk /dev/xvdg
    sudo gdisk /dev/xvdh
    ```
    Results:
    ![](img/gdisk-db.png)

- Use lsblk utility to view the newly configured partition on each of the 3 disks.
    ```
    sudo lsblk
    ```
    Results:
    ![](img/lsblk2-db.png)

- Install lvm2 package using sudo yum install lvm2. Run sudo lvmdiskscan command to check for available partitions.
    ```
    sudo yum install lvm2
    sudo lvmdiskscan
    ```
    Results:
    ![](img/lvm-db.png)
    ![](img/lvmdiskscan-db.png)
    Note: Unlike ubuntu that uses apt, for redhat the package manager is yum.

- Use pvcreate utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM
    ```
    sudo pvcreate /dev/xvdf2 /dev/xvdg2 /dev/xvdh2
    ```
    Results:
    ![](img/pvcreate-db.png)

- Verify that your Physical volume has been created successfully by running sudo pvs
    ```
    sudo pvs
    ```
    Results:
    ![](img/pvs-db.png)

- Use vgcreate utility to add all 3 PVs to a volume group (VG). Name the VG webdata-vg
    ```
    sudo vgcreate dbdata-vg /dev/xvdf2 /dev/xvdg2 /dev/xvdh2
    ```
    Results:
    ![](img/vgcreate-db.png)

- Use lvcreate utility to create 2 logical volumes. apps-lv (Use half of the PV size), and logs-lv Use the remaining space of the PV size. NOTE: apps-lv will be used to store data for the Website while, logs-lv will be used to store data for logs.
    ```
    sudo lvcreate -n db-lv -L 14G dbdata-vg
    sudo lvcreate -n logs-lv -L 14G dbdata-vg
    ```
    Results:
    ![](img/lvcreate-db.png)

- Verify that your Logical Volume has been created successfully by running sudo lvs
    ```
    sudo lvs
    ```
    Results:
    ![](img/lvs-db.png)

- We need to verify all we have done so far on the database server instance so far with these commands.
    ```
    sudo vgdisplay -v #view complete setup - VG, PV, and LV
    sudo lsblk 
    ```
    Results:
    ![](img/vgdisplay-db.png)
    ![](img/lsblk3-db.png)

- Use mkfs.ext4 to format the logical volumes with ext4 filesystem
    ```
    sudo mkfs.ext4 /dev/dbdata-vg/db-lv && sudo mkfs.ext4 /dev/dbdata-vg/logs-lv
    ```
    Results:
    ![](img/mkfs-db.png)

Now that we are done configuring the database logical volumes, we would be moving on with creating the mount points for the logical volumes and the required directories.

- Create /db directory to store website files
    ```
    sudo mkdir -p /db
    ```
    Results:
    ![](img/mkdir-db.png)

- Create /home/recovery/logs to store backup of log data
    ```
    sudo mkdir -p /home/recovery/logs
    ```
    Results:
    ![](img/mkdir-logs-db.png)

- Mount /db on apps-lv logical volume
    ```
    sudo mount /dev/dbdata-vg/db-lv /db
    ```
    Results:
    ![](img/mount-db.png)

- Like we did for the webserver instance use rsync utility to backup all the files in the log directory /var/log into /home/recovery/logs (This is required before mounting the file system)
    ```
    sudo rsync -av /var/log/. /home/recovery/logs/
    ```
    Results:
    ![](img/rsync-db.png)

- Mount /var/log on logs-lv logical volume. (Note that all the existing data on /var/log will be deleted. That is why step of creating /db directory to store database files)
    ```
    sudo mount /dev/dbdata-vg/logs-lv /var/log
    ```
    Results:
    ![](img/mount-log-db.png)

- Restore log files back into /var/log directory
    ```
    sudo rsync -av /home/recovery/logs/. /var/log
    ```
    Results:
    ![](img/rsync2-db.png)

Now we need to update the /etc/fstab file to ensure that the configurations we made is persistent across reboots.

- Update /etc/fstab file so that the mount configuration will persist after restart of the server.
The UUID of the device will be used to update the /etc/fstab file;
    ```
    sudo blkid
    ```
    Results:
    ![](img/blkid-db.png)

- Update /etc/fstab in this format using your own UUID and rememeber to remove the leading and ending quotes.
    ```
    sudo nano /etc/fstab
    ```
    and add this
    ```
    UUID=<uuid of your webdata-vg-apps> /var/www/html ext4 defaults 0 0
    UUID=<uuid of your webdata-vg-logs> /var/log ext4 defaults 0 0
    ```
    Results:
    ![](img/fstab2-db.png)

- Test the configuration and reload the daemon
    ```
    sudo mount -a
    sudo systemctl daemon-reload
    ```
    Results:
    ![](img/mount-a2-db.png)

- Verify your setup by running df -h, output must look like this:
    ```
    sudo df -h
    ```
    Results:
    ![](img/df-h2-db.png)

Now your db server is ready to go and make other configurations as required.

## Install WordPress on your Web Server EC2 Instance
- Update the repository
    ```
    sudo yum update
    ```
    Results:
    ![](img/yum-update.png)

- Install wget, Apache and it’s dependencies
    ```
    sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
    ```
    Results:
    ![](img/yum-install.png)