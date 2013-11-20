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

1.1. Keystone
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

* Define an authorization token to use as a shared secret between the Identity Service and other OpenStack services.

  Use **openssl** to generate a random token and store it in the file *admin_token*::

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

  We just created two tenants *admin* and *service*, a user *admin*, a role *admin*.

  The user *admin* is in *admin* tenant with *admin* role.

* Define services and API endpoints. Replace *the_service_id_above* with the actual service id created in first step::

   # keystone service-create --name=keystone --type=identity --description="Keystone Identity Service"
   # keystone endpoint-create \
     --service-id=the_service_id_above \
     --publicurl=http://controller:5000/v2.0 \
     --internalurl=http://controller:5000/v2.0 \
     --adminurl=http://controller:35357/v2.0

* Verify the Identity Service installation::

   # unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT
   # keystone --os-username=admin \
     --os-password=ADMIN_PASS \
     --os-auth-url=http://controller:35357/v2.0 token-get
   # keystone --os-username=admin \
     --os-password=ADMIN_PASS \
     --os-tenant-name=admin \
     --os-auth-url=http://controller:35357/v2.0 token-get

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

1.2. Glance
-----------------------------

This part assumes you set the appropriate environment variables to your credentials.

If not, just use **source keystonerc**.

* Install the Image Service on the controller node::

   # apt-get install glance

* Edit /etc/glance/glance-api.conf and /etc/glance/glance-registry.conf and change the [DEFAULT] section::

   ...
   [DEFAULT]
   ...
   # SQLAlchemy connection string for the reference implementation
   # registry server. Any valid SQLAlchemy connection string is fine.
   # See: http://www.sqlalchemy.org/docs/05/reference/sqlalchemy/connections.html#sqlalchemy.create_engine
   sql_connection = mysql://glance:GLANCE_DBPASS@controller/glance
   ...

* Use the password you created to log in as root and create a glance database user::

   # mysql -u root -p
   mysql> CREATE DATABASE glance;
   mysql> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
   mysql> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';

* Create the database tables for the Image Service::

   # glance-manage db_sync

* Create a glance user that the Image Service can use to authenticate with the Identity Service.

  Choose a password and specify an email address for the glance user.

  Use the service tenant and give the user the admin role::

   # keystone user-create --name=glance --pass=GLANCE_PASS --email=glance@example.com
   # keystone user-role-add --user=glance --tenant=service --role=admin

* Edit /etc/glance/glance-api.conf and /etc/glance/glance-registry.conf and change the [keystone_authtoken] section::

   ...
   [keystone_authtoken]
   auth_host = controller
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = GLANCE_PASS
   ...

* Edit /etc/glance/glance-api-paste.ini and /etc/glance/glance-registry-paste.ini to set the following options in the [filter:authtoken] section::

   [filter:authtoken]
   paste.filter_factory=keystoneclient.middleware.auth_token:filter_factory
   auth_host=controller
   admin_user=glance
   admin_tenant_name=service
   admin_password=GLANCE_PASS

* Register the Image Service with the Identity Service so that other OpenStack services can locate it.

  Register the service and create the endpoint::

   # keystone service-create --name=glance --type=image --description="Glance Image Service"

* Use the id property returned for the service to create the endpoint::

   # keystone endpoint-create \
     --service-id=the_service_id_above \
     --publicurl=http://controller:9292 \
     --internalurl=http://controller:9292 \
     --adminurl=http://controller:9292

* Restart the glance service with its new settings::

   # service glance-registry restart
   # service glance-api restart

Then we try to verify the Image Service Installation.

* Download the image into a dedicated directory::

   $ mkdir images
   $ cd images/
   $ wget http://cdn.download.cirros-cloud.net/0.3.1/cirros-0.3.1-x86_64-disk.img

* Upload the image to the Image Service::

   # glance image-create --name="CirrOS 0.3.1" --disk-format=qcow2 \
     --container-format=bare --is-public=true < cirros-0.3.1-x86_64-disk.img

   +------------------+--------------------------------------+
   | Property         | Value                                |
   +------------------+--------------------------------------+
   | checksum         | d972013792949d0d3ba628fbe8685bce     |
   | container_format | bare                                 |
   | created_at       | 2013-11-20T05:03:30                  |
   | deleted          | False                                |
   | deleted_at       | None                                 |
   | disk_format      | qcow2                                |
   | id               | 0d192c86-1a92-4ac5-97da-f3d95f74e811 |
   | is_public        | True                                 |
   | min_disk         | 0                                    |
   | min_ram          | 0                                    |
   | name             | CirrOS 0.3.1                         |
   | owner            | None                                 |
   | protected        | False                                |
   | size             | 13147648                             |
   | status           | active                               |
   | updated_at       | 2013-11-20T05:03:30                  |
   +------------------+--------------------------------------+

* Confirm that the image was uploaded and display its attributes::

   # glance image-list

   +--------------------------------------+--------------+-------------+------------------+----------+--------+
   | ID                                   | Name         | Disk Format | Container Format | Size     | Status |
   +--------------------------------------+--------------+-------------+------------------+----------+--------+
   | 0d192c86-1a92-4ac5-97da-f3d95f74e811 | CirrOS 0.3.1 | qcow2       | bare             | 13147648 | active |
   +--------------------------------------+--------------+-------------+------------------+----------+--------+

1.3. Horizon
------------

* Install the dashboard on controller node::

   # apt-get install memcached libapache2-mod-wsgi openstack-dashboard

* You can now access the dashboard at http://controller/horizon .

  Login with credentials for any user that you created with the OpenStack Identity Service.

