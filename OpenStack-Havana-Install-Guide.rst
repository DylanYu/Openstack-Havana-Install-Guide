===================================
OpenStack Havana Installation Guide
===================================

Table of Contents
=================

::

  0. Basic Configuration
  1. Controller Node
  2. Networking Node
  3. Compute Node


0. Basic Configuration
======================

0.1. Networking
---------------

:Node Name:  NICs
:Networking: eth0 (192.168.0.11), eth1 (114.212.189.131)
:Controller: eth0 (192.168.0.12), eth1 (114.212.189.132)
:Compute:    eth0 (192.168.0.13), eth2 (114.212.189.133)

* Configure host names by manually editing the /etc/hosts file on each system::

   127.0.0.1       localhost
   192.168.0.11    networking
   192.168.0.12    controller
   192.168.0.13    compute

0.2. NTP
--------

* Install the ntp package on each system running OpenStack services::

   # apt-get install ntp

* On any nodes besides controller node, synchronize their time from the controller node rather than from outside of your LAN. To do so, edit /etc/ntp.conf and change the server directive to use the controller node as internet time source::

   server controller
   #server 0.ubuntu.pool.ntp.org
   #server 1.ubuntu.pool.ntp.org
   #server 2.ubuntu.pool.ntp.org
   #server 3.ubuntu.pool.ntp.org

   #server ntp.ubuntu.com

0.3. MySQL database
-------------------

* On the controller node, install the MySQL database, and the MySQL Python library::

   # apt-get install python-mysqldb mysql-server

* Edit /etc/mysql/my.cnf and set the bind-address to the internal IP address of the controller, to allow access from outside the controller node::

   bind-address = 192.168.0.12

* Restart the MySQL service to apply the changes::

   # service mysql restart

* Delete the anonymous users with **mysql_secure_installation** command. Respond **yes** to all prompts besides *change the root password*::

   # mysql_secure_installation

* On any nodes besides the controller node, just install MySQL Python library::

   # apt-get install python-mysqldb

0.4. OpenStack packages
-----------------------

On each node,

* Install the Ubuntu Cloud Archive for Havana::

   # apt-get install python-software-properties
   # add-apt-repository cloud-archive:havana

* Upgrade the system::

   # apt-get update && apt-get dist-upgrade

0.5. Messaging server
---------------------

* On the controller node, install the messaging queue server::

   # apt-get install rabbitmq-server


1. Controller Node
==================

1.1 Identity service
--------------------

* Install the Identity Service::

   # apt-get install keystone

* Edit /etc/keystone/keystone.conf and change the [sql] section::

   [sql]
   # The SQLAlchemy connection string used to connect to the database
   connection = mysql://keystone:KEYSTONE_DBPASS@controller/keystone

* Use the password that you set previously to log in as root. Create a keystone database user::

   # mysql -u root -p
   mysql> CREATE DATABASE keystone;
   mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
   mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';

* Start the keystone service and create its tables::

   # keystone-manage db_sync
   # service keystone restart

* Define an authorization token to use as a shared secret between the Identity Service and other OpenStack services. Use **openssl** to generate a random token and store it in the file *admin_token*::

   # openssl rand -out admin_token -hex 10

* Edit /etc/keystone/keystone.conf and change the [DEFAULT] section, replacing ADMIN_TOKEN with the results of the command::

   [DEFAULT]
   # A "shared secret" between keystone and other openstack services
   admin_token = ADMIN_TOKEN
   ...

* Define users, tenants, and roles. Replace ADMIN_TOKEN with the actual token created above::

   # export OS_SERVICE_TOKEN=ADMIN_TOKEN
   # export OS_SERVICE_ENDPOINT=http://controller:35357/v2.0
   # keystone tenant-create --name=admin --description="Admin Tenant"
   # keystone tenant-create --name=service --description="Service Tenant"
   # keystone user-create --name=admin --pass=ADMIN_PASS --email=admin@example.com
   # keystone role-create --name=admin
   # keystone user-role-add --user=admin --tenant=admin --role=admin

  We just created two tenants *admin* and *service*, a user *admin*, a role *admin*. The user *admin* is in *admin* tenant with *admin* role.

* Define services and API endpoints. Replace *the_service_id_above* with the actual service id created in first step::

   # keystone service-create --name=keystone --type=identity --description="Keystone Identity Service"
   # keystone endpoint-create \
     --service-id=the_service_id_above \
     --publicurl=http://controller:5000/v2.0 \
     --internalurl=http://controller:5000/v2.0 \
     --adminurl=http://controller:35357/v2.0

* Verify the Identity Service installation::

   # unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT
   # keystone --os-username=admin --os-password=ADMIN_PASS --os-auth-url=http://controller:35357/v2.0 token-get
   # keystone --os-username=admin --os-password=ADMIN_PASS --os-tenant-name=admin --os-auth-url=http://controller:35357/v2.0 token-get

  You should receive tokens in response.

* Set up a keystonerc file with the admin credentials and admin endpoint to simplify command-line usage::

   export OS_USERNAME=admin
   export OS_PASSWORD=ADMIN_PASS
   export OS_TENANT_NAME=admin
   export OS_AUTH_URL=http://controller:35357/v2.0

  You can source this file to read in the environment variable::

   # source keystonerc

  Verify that your *keystonerc* is configured correctly by performing the same command as above, but without the --os-* arguments::

   # keystone token-get

  The command returns a token and the ID of the specified tenant. This verifies that you have configured your environment variables correctly.

* Finally, verify that your admin account has authorization to perform administrative commands::

   # keystone user-list
   +----------------------------------+-------+---------+-------------------+
   |                id                |  name | enabled |       email       |
   +----------------------------------+-------+---------+-------------------+
   | 1a466d433c7441ff986bb64536bd434b | admin |   True  | admin@example.com |
   +----------------------------------+-------+---------+-------------------+


