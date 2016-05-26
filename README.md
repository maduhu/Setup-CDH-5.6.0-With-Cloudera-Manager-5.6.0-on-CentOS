# Setup-CDH-5.6.0-With-Cloudera-Manager-5.6.0-on-CentOS

###STEP - 1
Prerequisites

Ensure environment is properly configured to resolve hostname to ip addresses using /etc/hosts
Ensure that the hostname you have set is also same in the file /etc/sysconfig/network
Passwordless SSH 
Install Java version 1.7 or greater 
Disable IPtables and IP6tables  
Disable swappiness
Set ulimit
Install NTP
Disable SELinux
Disable Transparent Huge Pages 
Upgrade openSSL
Extend disk capacity on hosts

###STEP - 2
Download and Install the Cloudera Manager server and Repos

// Download the Cloudera Manager repo file (Version 5) to /etc/yum.repos.d on the host that will have the Cloudera Manager Server installed. 

wget http://archive.cloudera.com/cm5/redhat/6/x86_64/cm/cloudera-manager.repo 

// Install the Cloudera Manager Server packages

sudo  yum install cloudera-manager-daemons cloudera-manager-server

#STEP - 3
Installing External Database : MySQL 

Install MySQL

// Install and configure the server 

yum -y install mysql-server
service mysqld start
mysql_secure_installation 

the above step would require more configuration 

Enter current password for root (enter for none): press enter

Set root password? [Y/n] y

// set new password and reenter new password
 
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] n
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y

// This is for creating the user for ambari this is same for hive and oozie

CREATE USER 'ambari'@'localhost' IDENTIFIED BY 'ambari';
GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'localhost' with grant option;
CREATE USER 'ambari'@'%' IDENTIFIED BY 'ambari';
GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'%' with grant option;
CREATE USER 'ambari'@'node1.cloudwick.com' IDENTIFIED BY 'ambari';
GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'node1.cloudwick.com' with grant option;

###VERY IMPORTANT

// change the Configuring MySQL > 

go to > /etc/my.cnf

// To make sure mysql instance is accessible from outside the instance, locate [mysqld] section and add/correct as follows so that 
mysqld can reached remotely

bind-address=[IP_ADDRESS_OF_MYSQL_SERVER]

// Optimizing settings: you need to optimize mysql server otherwise it is going to eat all your CPU and other resources

// put this is below the [mysqld]

[mysqld]
//binding address
bind-address=[IP_ADDRESS_OF_MYSQL_SERVER]
  
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
 
With the READ-COMMITTED isolation level, the phenomenon of dirty read is avoided, 
because any uncommitted changes is not visible to any other transaction, until the 
change is committed.
transaction-isolation=READ-COMMITTED
 
 
//Disabling symbolic-links is recommended to prevent assorted security risks;
to do so, uncomment this line:
symbolic-links=0
 
key_buffer              = 16M
key_buffer_size         = 32M
max_allowed_packet      = 16M
thread_stack            = 256K
thread_cache_size       = 64
query_cache_limit       = 8M
query_cache_size        = 64M
query_cache_type        = 1
 
 
//Important: see Configuring the Databases and Setting max_connections
max_connections         = 550
 
 
//Important: log-bin should be on a disk with enough free space
Enable binary logging. The server logs all statements that change data to the 
binary log, which is used for backup and replication.
log-bin=/var/lib/mysql/logs/binary/mysql_binary_log
 
 
//For MySQL version 5.1.8 or later. Comment out binlog_format for older versions.
binlog_format = mixed
read_buffer_size = 2M
read_rnd_buffer_size = 16M
sort_buffer_size = 8M
join_buffer_size = 8M
 
 
//InnoDB settings
default-storage_engine = InnoDB
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit  = 2
innodb_log_buffer_size          = 64M
innodb_buffer_pool_size         = 4G
innodb_thread_concurrency       = 8
innodb_flush_method             = O_DIRECT
innodb_log_file_size = 512M
 
 
[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

// then save it

// Ensure the mysqld server starts at boot 

sudo /sbin/chkconfig mysqld on
sudo /sbin/chkconfig --list mysqld

// Create the database and user for Oozie, Activity Monitor,Reports Manager, Hive Metastore, Sentry Server, Cloudera Navigator Audit Server in MySQL: 

//Oozie
	create database oozie;
	grant all on oozie.* TO 'oozie'@'%' IDENTIFIED BY 'oozie';

//Activity Monitor
	create database amon DEFAULT CHARACTER SET utf8;
grant all on amon.* TO 'amon'@'%' IDENTIFIED BY 'amon';

//Reports Manager
	create database rman DEFAULT CHARACTER SET utf8;
	grant all on rman.* TO 'rman'@'%' IDENTIFIED BY 'rman';

//Hive Metastore
	create database hive DEFAULT CHARACTER SET utf8;
	grant all on hive.* TO 'hive'@'%' IDENTIFIED BY 'hive';

//Sentry Server
	create database sentry DEFAULT CHARACTER SET utf8;
	grant all on sentry.* TO 'sentry'@'%' IDENTIFIED BY 'sentry';

//Cloudera Navigator Audit Server
	create database nav DEFAULT CHARACTER SET utf8;
	grant all on nav.* TO 'nav'@'%' IDENTIFIED BY 'nav';

//Cloudera Navigator Metadata Server
create database navms DEFAULT CHARACTER SET utf8;
grant all on navms.* TO 'navms'@'%' IDENTIFIED BY 'navms';

// Cloudera Manager also requires a database, which is configured using the scm_prepare_database.sh on the host where the 
Cloudera Manager Server package is installed using following syntax:

scm_prepare_database.sh database-type [options] database-name username password

NOTE : 
// To run the script when MySQL and the Cloudera Manager Server will be installed on different hosts, first create a temporary user in MySQL who can connect to MySQL remotely
grant all on *.* to 'temp'@'%' identified by 'temp' with grant option; 

// Then run the script on the Cloudera Manager host with the options -h, -u, -p, and --scm-host, where -h is the host where MySQL is installed, -u for the username for the database, -p for the password for the database, and --scm-host where the Cloudera Manager will be installed: 

sudo /usr/share/cmf/schema/scm_prepare_database.sh mysql -h myhost1.sf.cloudera.com -utemp -ptemp --scm-host myhost2.sf.cloudera.com scm scm scm 


// After the script has run successfully, there should be a user and database in MySQL called “scm”. Finally, delete the temporary user: 

drop user 'temp'@'%';

###Step 4
Start Cloudera Manager Server and login to Cloudera Manager Admin Console

//Run the following command in the terminal on the Cloudera Manager Server host 

sudo service cloudera-scm-server start

// Wait several minutes for the Cloudera Manager Server to start. To observe the startup process, view the log for the Cloudera Manager server by running 

tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log

In a web browser, enter http://Server host:7180, where Server host is the FQDN or IP address of the host where the 
Cloudera Manager Server is running. and check whether you can see the login page. The default username and password is admin. 

then setup the configurations as you need.




