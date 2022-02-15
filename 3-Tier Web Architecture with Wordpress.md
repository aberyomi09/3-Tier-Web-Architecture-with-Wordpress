In this project I will prepare storage infrastructure on two Linux servers and
implement a basic web solution using
[WordPress](https://en.wikipedia.org/wiki/WordPress). WordPress is a free and
open-source content management system written in **PHP** and paired with
**MySQL** or **MariaDB** as its backend Relational Database Management System
(RDBMS).

This project consists of two parts:

1.  Configure storage subsystem for Web and Database servers based on RedHat
    Linux OS. This requires prerequisite knowledge of working with disks,
    partitions and volumes in Linux.

2.  Install WordPress and connect it to a remote MySQL database server. This
    part of the project will solidify your skills of deploying Web and DB tiers
    of Web solution.

As a DevOps engineer, your deep understanding of core components of web
solutions and ability to troubleshoot them will play essential role in your
further progress and development.

**Three-tier Architecture**

Generally, web, or mobile solutions are implemented based on what is called the
**Three-tier Architecture**.

**Three-tier Architecture** is a client-server software architecture pattern
that comprise of 3 separate layers.

\*\*3-tier image

1.  **Presentation Layer** (PL): This is the user interface such as the client
    server or browser on your laptop.

2.  **Business Layer** (BL): This is the backend program that implements
    business logic. Application or Webserver

3.  **Data Access or Management Layer** (DAL): This is the layer for computer
    data storage and data access. [Database
    Server](https://www.computerhope.com/jargon/d/database-server.htm) or File
    System Server such as [FTP
    server](https://titanftp.com/2018/09/11/what-is-an-ftp-server/), or [NFS
    Server](https://searchenterprisedesktop.techtarget.com/definition/Network-File-System)

In this project, you will have the hands-on experience that showcases
**Three-tier Architecture** while also ensuring that the disks used to store
files on the Linux servers are adequately partitioned and managed through
programs such as gdisk/fdisk and LVM respectively. I will be working working
with several storage and disk management concepts, so prerequisite knowledge of
Disk management in Linux is important.

##### **My 3-Tier Setup**

1.  A Laptop or PC to serve as a client

2.  An EC2 Linux Server as a web server (This is where I will install WordPress)

3.  An EC2 Linux server as a database (DB) server

LAUNCH AN **EC2 INSTANCE THAT WILL SERVE AS “WEB SERVER”.**

#### Step 1 — Prepare a Web Server

1.  Launch an EC2 instance that will serve as "Web Server". Create 3 volumes in
    the same AZ as your Web Server EC2, each of 10 GiB.

\*\* ATTACH VOLUME IMAGE

1.  Attach all three volumes one by one to your Web Server EC2 instance

2.  Open up the Linux terminal to begin configuration

3.  Use lsblk command to inspect what block devices are attached to the server.
    Notice names of your newly created devices. All devices in Linux reside in
    /dev/ directory. Inspect it with ls /dev/ and make sure you see all 3 newly
    created block devices there – their names will likely be xvdf, xvdh, xvdg.

LSBLK IMAGE

1.  Use df -h command to see all mounts and free space on your server

2.  Use gdisk utility to create a single partition on each of the 3 disks

*\#sudo fdisk /dev/xvdf*

*Fdisk Image*

1.  Use lsblk utility to view the newly configured partition on each of the 3
    disks.

Lsblk image

1.  Install [lvm2](https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux))
    package using *\#sudo yum install lvm2*. Run *\#sudo lvmdiskscan* command to
    check for available partitions.

Lvmdiskscan image

1.  Use [*\#pvcreate*](https://linux.die.net/man/8/pvcreate) utility to mark
    each of 3 disks as physical volumes (PVs) to be used by LVM

Pvcreate image

1.  Verify that your Physical volume has been created successfully by running
    *\#sudo pvs*

1.  Use [*\#vgcreate*](https://linux.die.net/man/8/vgcreate) utility to add all
    3 PVs to a volume group (VG). Name the VG **webdata-vg**

*\#sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1*

1.  Verify that your VG has been created successfully by running *\#sudo vgs*

1.  Use [*\#lvcreate*](https://linux.die.net/man/8/lvcreate) utility to create 2
    logical volumes. **apps-lv** (**Use half of the PV size**), and **logs-lv**
    **Use the remaining space of the PV size**. **NOTE**: apps-lv will be used
    to store data for the Website while, logs-lv will be used to store data for
    logs.

*\#sudo lvcreate -n apps-lv -l 50%FREE webdata-vg*

*\#sudo lvcreate -n logs-lv –l 100%FREE webdata-vg*

Lvcreate image

1.  Verify that your Logical Volume has been created successfully by running
    *\#sudo lvs*

2.  Use mkfs.ext4 to format the logical volumes with
    [ext4](https://en.wikipedia.org/wiki/Ext4) filesystem

*\#sudo mkfs -t ext4 /dev/webdata-vg/apps-lv*

*\#sudo mkfs -t ext4 /dev/webdata-vg/logs-lv*

Mkfs image

1.  Create **/var/www/html** directory to store website files

    *\#sudo mkdir -p /var/www/html*

2.  Create **/home/recovery/logs** to store backup of log data

    *\#sudo mkdir -p /home/recovery/logs*

3.  Mount **/var/www/html** on **apps-lv** logical volume

*\#sudo mount /dev/webdata-vg/apps-lv /var/www/html/*

1.  Use rsync utility to backup all the files in the log directory **/var/log**
    into **/home/recovery/logs** (*This is required before mounting the file
    system*)

*\#sudo rsync -av /var/log/. /home/recovery/logs/*

1.  Mount **/var/log** on **logs-lv** logical volume. (*Note that all the
    existing data on /var/log will be deleted. That is why step 15 above is very
    
    important*)

    *\#sudo mount /dev/webdata-vg/logs-lv /var/log*

2.  Restore log files back into **/var/log** directory

    *\#sudo rsync -av /home/recovery/logs/. /var/log*

3.  Update /etc/fstab file so that the mount configuration will persist after
    restart of the server.

**UPDATE THE /etc/fstab file**

The UUID of the device can be used to update the /etc/fstab file; alternatively
the device path or a label can be created to update the file.

*\#sudo vi /etc/fstab*

Fstab image

Test the configuration and reload the daemon

*\#sudo mount –a*

*\#sudo systemctl daemon-reload*

**Step 2 — Prepare the Database Server**

Launch a second RedHat EC2 instance that will have a role – ‘DB Server’

Repeat the same steps as for the Web Server, but instead of apps-lv create db-lv
ONLY (20gb i.e. 2 10gb LVs) and mount it to /db directory instead of
/var/www/html/.

#### Step 3 — Install WordPress on your Web Server EC2

1.  Update the repository

    *\#sudo yum -y update*

2.  Install wget, Apache and it’s dependencies

    *\#sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json*

3.  Start Apache

*\#sudo systemctl enable httpd*

*\#sudo systemctl start httpd*

*\#sudo systemctl status httpd*

Status httpd image

1.  To install PHP and it’s depemdencies

    *\#sudo yum install
    https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm*

    *\#sudo yum install yum-utils
    http://rpms.remirepo.net/enterprise/remi-release-8.rpm*

    *\#sudo yum module list php*

    *\#sudo yum module reset php*

    *\#sudo yum module enable php:remi-7.4*

    *\#sudo yum install php php-opcache php-gd php-curl php-mysqlnd*

    *\#sudo systemctl start php-fpm*

    *\#sudo systemctl enable php-fpm*

    *\#sudo systemctl status php-fpm*

    *\#setsebool -P httpd_execmem 1*

Status php image

1.  Restart Apache

    *\#sudo systemctl restart httpd*

2.  Download wordpress and copy wordpress to var/www/html

    *\#mkdir wordpress*

\#*cd wordpress*

*\# sudo wget http://wordpress.org/latest.tar.gz*

*\#sudo tar xzvf latest.tar.gz*

*\#sudo rm -rf latest.tar.gz*

*\#cp wordpress/wp-config-sample.php wordpress/wp-config.php*

*\# cp -R wordpress /var/www/html/*

1.  Configure SELinux Policies

    \#sudo chown -R apache:apache /var/www/html/wordpress

    \#sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R

    \#sudo setsebool -P httpd_can_network_connect=1

#### Step 4 — Install MySQL on your DB Server EC2

*\#sudo yum update*

*\#sudo yum install mysql-server*

Verify that the service is up and running by using *\#sudo systemctl status
mysqld*, if it is not running, restart the service and enable it so it will be
running even after reboot:

*\#sudo systemctl restart mysqld*

*\#sudo systemctl enable mysqld*

*\#sudo systemctl status mysqld*

*Status mysql image*

#### Step 5 — Configure DB to work with WordPress

*\#sudo mysql*

*CREATE DATABASE wordpress;*

*CREATE USER \`myuser\`@\`\<Web-Server-Private-IP-Address\>\` IDENTIFIED BY
'mypass';*

*GRANT ALL ON wordpress.\* TO 'myuser'@'\<Web-Server-Private-IP-Address\>';*

*FLUSH PRIVILEGES;*

*SHOW DATABASES;*

*exit*

#### Step 6 — Configure WordPress to connect to remote database.

**Hint:** Do not forget to open MySQL port 3306 on DB Server EC2. For extra
security, you should allow access to the DB server **ONLY** from your Web
Server’s IP address, so in the Inbound Rule configuration specify source as /32

1.  Install MySQL client and test that you can connect from your Web Server to
    your DB server by using mysql-client

*\#sudo yum install mysql*

*\#sudo mysql -u admin -p -h \<DB-Server-Private-IP-address\>*

*Mysql –u image*

1.  Verify if you can successfully execute SHOW DATABASES; command and see a
    list of existing databases.

2.  Change permissions and configuration so Apache could use WordPress:

3.  Enable TCP port 80 in Inbound Rules configuration for your Web Server EC2
    (enable from everywhere 0.0.0.0/0 or from your workstation’s IP)

4.  Try to access from your browser the link to your WordPress
    http://\<Web-Server-Public-IP-Address\>/wordpress/

Install wordpress image

Fill out your DB credentials:

Final Image

#### CONGRATULATIONS!

We have learned how to configure Linux storage susbystem and have also deployed
a full-scale Web Solution using WordPress CMS and MySQL RDBMS!

If you see this message – it means your WordPress has successfully connected to
your remote MySQL database
