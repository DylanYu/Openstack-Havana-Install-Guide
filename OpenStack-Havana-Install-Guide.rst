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


