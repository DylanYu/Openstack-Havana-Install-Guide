===================================
OpenStack Havana Installation Guide
===================================

Table of Contents
=================

::

  0. Basic Configuration
  1. Controller
  2. Networking
  3. Compute


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

* On any nodes besides the controller node, just install the MySQL Python library::

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


1. Controller
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

* Define services and API endpoints. Replace *the_service_id_above* with the actual service id created in first step (similarly hereinafter)::

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


2. Networking
==================

2.1. Basic setup
----------------

This part creates required OpenStack components: user, service, database, and endpoint, on the **controller node**.

* Create a neutron database::

   # mysql -u root -p
   mysql> CREATE DATABASE neutron;
   mysql> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
   IDENTIFIED BY 'NEUTRON_DBPASS';
   mysql> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
   IDENTIFIED BY 'NEUTRON_DBPASS';

* Create the required user, service, and endpoint so that Networking can interface with the Identity Service::

   # keystone user-create --name=neutron --pass=NEUTRON_PASS --email=neutron@example.com
   # keystone user-role-add --user=neutron --tenant=service --role=admin
   # keystone service-create --name=neutron --type=network \
     --description="OpenStack Networking Service"
   # keystone endpoint-create \
        --service-id the_service_id_above \
        --publicurl http://controller:9696 \
        --adminurl http://controller:9696 \
        --internalurl http://controller:9696

2.2. Install Networking services on a dedicated network node
------------------------------------------------------------

* Install the OpenStack Networking service on the network node::

   # apt-get install neutron-server neutron-dhcp-agent neutron-plugin-openvswitch-agent neutron-l3-agent

* Edit the /etc/sysctl.conf file, as follows::

   net.ipv4.ip_forward=1
   net.ipv4.conf.all.rp_filter=0
   net.ipv4.conf.default.rp_filter=0

  This step enables packet forwarding and disables packet destination filtering so that the network node can coordinate traffic for the VMs.

  To activate changes in the /etc/sysctl.conf file, run the following command::

   # sysctl -p

* Edit the /etc/neutron/neutron.conf file and add these lines to the keystone_authtoken section::

   [keystone_authtoken]
   auth_host = controller
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = neutron
   admin_password = NEUTRON_PASS
   auth_url=http://controller:35357/v2.0
   auth_strategy=keystone

  **Note:** You MUST configure *auth_url* to point to keystone endpoint.

* Configure the RabbitMQ access. Edit the /etc/neutron/neutron.conf file to modify the following parameters in the DEFAULT section::

   rabbit_host = controller
   rabbit_userid = guest
   rabbit_password = guest

  If you've changed you RabbitMQ password, remeber to modify the value of *rabbit_password*.

* Edit the [database] section in the same file, as follows::

   [database]
   connection = mysql://neutron:NEUTRON_DBPASS@controller/neutron

* Edit the /etc/neutron/api-paste.ini file and add these lines to the [filter:authtoken] section::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host=controller
   auth_uri=http://controller:5000
   admin_user=neutron
   admin_tenant_name=service
   admin_password=NEUTRON_PASS

Then we start to install the Open vSwitch (OVS) plug-in. Good luck.

* Install the Open vSwitch plug-in and its dependencies::

   # apt-get install neutron-plugin-openvswitch-agent openvswitch-switch

* Start Open vSwitch::

   # service openvswitch-switch start

* Installl the Open vSwitch datapath module and make sure the Open vSwitch module is loaded correctly::

   # apt-get install openvswitch-datapath-source
   # module-assistant auto-install openvswitch-datapath
   # modinfo openvswitch
   
   filename:       /lib/modules/3.2.0-56-generic/updates/openvswitch/openvswitch.ko
   version:        1.10.2
   license:        GPL
   description:    Open vSwitch switching datapath
   srcversion:     EBF7178BF66BA8C40E397CB
   depends:        
   vermagic:       3.2.0-56-generic SMP mod_unload modversions 

* Add integration and external bridges::

   # ovs-vsctl add-br br-int
   # ovs-vsctl add-br br-ex

* Add a port (connection) from the EXTERNAL_INTERFACE interface (114.212.189.131 in this guide) to br-ex interface::

   # ovs-vsctl add-port br-ex EXTERNAL_INTERFACE

* Configure the EXTERNAL_INTERFACE without an IP address and in promiscuous mode. Additionally, you must set the newly created br-ex interface to have the IP address that formerly belonged to EXTERNAL_INTERFACE. Edit file /etc/network/interfaces as follows::

   auto lo
   iface lo inet loopback

   auto eth0
   iface eth0 inet static
       address 192.168.0.11
       netmask 255.255.255.0

   auto eth1
   iface eth1 inet manual
   up ifconfig $IFACE 0.0.0.0 up
   up ip link set $IFACE promisc on
   down ip link set $IFACE promisc off
   down ifconfig $IFACE down

   auto br-ex
   iface br-ex inet static
       address 114.212.189.131
       netmask 255.255.255.0
       gateway 114.212.189.1

  Restart your network service to apply changes.

* Edit the /etc/neutron/l3_agent.ini and /etc/neutron/dhcp_agent.ini files, respectively::

   interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
   use_namespaces = False

* Edit the /etc/neutron/neutron.conf file::

   core_plugin = neutron.plugins.openvswitch.ovs_neutron_plugin.OVSNeutronPluginV2

* Configure the OVS plug-in to use GRE tunneling, the br-int integration bridge, the br-tun tunneling bridge, and a local IP for the DATA_INTERFACE tunnel IP (192.168.0.11 in this guide). Edit the /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini file::

   [ovs]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   enable_tunneling = True
   integration_bridge = br-int
   tunnel_bridge = br-tun
   local_ip = DATA_INTERFACE

* Configure a firewall plug-in. Edit the /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini file::

   [securitygroup]
   # Firewall driver for realizing neutron security group function.
   firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

* Configure the database connection. Edit the [database] section in the above file (This step is missing in the offical installation guide)::

   [database]
   sql_connection = mysql://neutron:NEUTRON_DBPASS@controller:3306/neutron

* Restart the OVS plug-in and make sure it starts on boot::

   # service neutron-plugin-openvswitch-agent restart

* List your virtual bridges::

   # ovs-vsctl show

  You should see br-ex, br-int which are created by yourself manually, and br-tun which is created by openvswitch automatically.

Now you've installed and configured a plug-in.

* Use the Dnsmasq plug-in to perform DHCP on the software-defined networks. Edit the /etc/neutron/dhcp_agent.ini file::

   dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq

* Restart Networking::

   # service neutron-dhcp-agent restart
   # service neutron-l3-agent restart

  If you check the dhcp-agent.log and l3-agent.log in /var/log/neutron, you will see error messages *Skipping unknown group key: firewall_driver* and *Router id is required if not using namespaces*. These may be bugs and you should not worry about them temporarily.

2.3. Install networking support on a dedicated compute node
-----------------------------------------------------------

This section details set up for any node that runs the nova-compute component but does not run the full network stack.

* Disable packet destination filtering (route verification) to let the networking services route traffic to the VMs. Edit the /etc/sysctl.conf file::

   net.ipv4.conf.all.rp_filter=0
   net.ipv4.conf.default.rp_filter=0

* Install the Open vSwitch plug-in and its dependencies::

   # apt-get install neutron-plugin-openvswitch-agent openvswitch-switch openvswitch-datapath-dkms

* Add the br-int integration bridge, which connects to the VMs::

   # ovs-vsctl add-br br-int

* Restart Open vSwitch::

   # service openvswitch-switch restart

* You must configure Networking core to use OVS. Edit the /etc/neutron/neutron.conf file::

   core_plugin = neutron.plugins.openvswitch.ovs_neutron_plugin.OVSNeutronPluginV2

* Tell the OVS plug-in to use GRE tunneling with a br-int integration bridge, a br-tun tunneling bridge, and a local IP for the tunnel of DATA_INTERFACE's IP (192.168.0.13 in this guide). Edit the /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini file::

   [ovs]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   enable_tunneling = True
   integration_bridge = br-int
   tunnel_bridge = br-tun
   local_ip = DATA_INTERFACE_IP

* Configure a firewall plug-in. Edit the /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini file::

   [securitygroup]
   # Firewall driver for realizing neutron security group function.
   firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

* Configure the database connection. Edit the [database] section in the above file::

   [database]
   sql_connection = mysql://neutron:NEUTRON_DBPASS@controller:3306/neutron

* Configure the core components of Neutron. Edit the /etc/neutron/neutron.conf file::

   [DEFAULT]
   rpc_backend = neutron.openstack.common.rpc.impl_kombu
   rabbit_host = controller
   # Change the following settings if you're not using the default RabbitMQ configuration
   #rabbit_port = 5672
   #rabbit_userid = guest
   #rabbit_password = guest
   [keystone_authtoken]
   auth_host = controller
   admin_tenant_name = service
   admin_user = neutron
   admin_password = NEUTRON_PASS
   auth_url = http://controller:35357/v2.0
   auth_strategy = keystone

* Edit the database URL under the [database] section in the above file, to tell Neutron how to connect to the database::

   [database]
   sql_connection = mysql://neutron:NEUTRON_DBPASS@controller/neutron

* Edit the /etc/neutron/api-paste.ini file and add these lines to the [filter:authtoken] section::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host=controller
   admin_user=neutron
   admin_tenant_name=service
   admin_password=NEUTRON_PASS

* Restart the Neutron Open vSwitch agent::

   # service neutron-plugin-openvswitch-agent restart

* Verify your configuration with the following command::

   # ovs-vsctl show

   e233a600-2486-4273-9f7a-a62b11ddd0d2
       Bridge br-tun
           Port "gre-1"
               Interface "gre-1"
                   type: gre
                   options: {in_key=flow, local_ip="192.168.0.13", out_key=flow, remote_ip="192.168.0.11"}
           Port br-tun
               Interface br-tun
                   type: internal
           Port patch-int
               Interface patch-int
                   type: patch
                   options: {peer=patch-tun}
       Bridge br-int
           Port br-int
               Interface br-int
                   type: internal
           Port patch-tun
               Interface patch-tun
                   type: patch
                   options: {peer=patch-int}
       ovs_version: "1.10.2"

  As you will see, by the information of br-tun, our compute node is now able to sense the networking node through the tunnel.

2.4. Install networking support on a dedicated controller node
--------------------------------------------------------------

This is for a node which runs the control components of Neutron, but does not run any of the components that provide the underlying functionality (such as the plug-in agent or the L3 agent).

* Install the main Neutron server, Neutron libraries for Python, and the Neutron command-line interface (CLI)::

   # apt-get install neutron-server python-neutronclient

* Configure the core components of Neutron. Edit the /etc/neutron/neutron.conf file::

   [DEFAULT]
   rpc_backend = neutron.openstack.common.rpc.impl_kombu
   [keystone_authtoken]
   auth_host = controller
   admin_tenant_name = service
   admin_user = neutron
   admin_password = NEUTRON_PASS
   auth_url = http://controller:35357/v2.0
   auth_strategy = keystone

* Edit the database URL under the [database] section in the above file, to tell Neutron how to connect to the database::

   [database]
   connection = mysql://neutron:NEUTRON_DBPASS@controller/neutron

* Configure the Neutron copy of the api-paste.ini at /etc/neutron/api-paste.ini file::

   [filter:authtoken]
   EXISTING_STUFF_HERE
   admin_tenant_name = service
   admin_user = neutron
   admin_password = NEUTRON_PASS

* Install the Open vSwitch plug-in::

   # apt-get install neutron-plugin-openvswitch

  Nothing new to install here. We could just skip this step.

  **Note:** The dedicated controller node does not need to run Open vSwitch or the Open vSwitch agent.

* You must set some common configuration options no matter which networking technology you choose to use with Open vSwitch. You must configure Networking core to use OVS. Edit the /etc/neutron/neutron.conf file::

   core_plugin = neutron.plugins.openvswitch.ovs_neutron_plugin.OVSNeutronPluginV2

* Tell the OVS plug-in to use GRE tunneling. Edit the /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini file::

   [ovs]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   enable_tunneling = True

* **TO DE DETERMINED**::

  Whether to add firewall and sql connection config in ovs_neutron_plugin.ini.

* Tell Nova about Neutron. Specifically, you must tell Nova that Neutron will be handling networking and the firewall. Edit the /etc/nova/nova.conf file::

   network_api_class=nova.network.neutronv2.api.API
   neutron_url=http://controller:9696
   neutron_auth_strategy=keystone
   neutron_admin_tenant_name=service
   neutron_admin_username=neutron
   neutron_admin_password=NEUTRON_PASS
   neutron_admin_auth_url=http://controller:35357/v2.0
   firewall_driver=nova.virt.firewall.NoopFirewallDriver
   security_group_api=nova

  Regardless of which firewall driver you chose when you configure the network and compute nodes, set this driver as the No-Op firewall. The difference is that this is a *Nova* firewall, and because Neutron handles the Firewall, you must tell Nova not to use one.

  **Note:** As we haven't install the Compute service (later in Part 3), the /etc/nova/nova.conf file does not exist. We could install the Compute service in advance to create the file, with the following command::

   # apt-get install nova-novncproxy novnc nova-api \
     nova-ajax-console-proxy nova-cert nova-conductor \
     nova-consoleauth nova-doc nova-scheduler \
     python-novaclient

* Restart neutron-server::

   # service neutron-server restart


3. Compute
==============

In this guide, most Compute services run on the controller node and the service that launches virtual machines runs on a dedicated compute node. 

3.1.  Install the Compute controller services
---------------------------------------------

On the controller node,

* Install these Compute packages, which provide the Compute services that run on the controller node::

   # apt-get install nova-novncproxy novnc nova-api \
     nova-ajax-console-proxy nova-cert nova-conductor \
     nova-consoleauth nova-doc nova-scheduler \
     python-novaclient

* Create a nova database user::

   # mysql -u root -p
   mysql> CREATE DATABASE nova;
   mysql> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
   IDENTIFIED BY 'NOVA_DBPASS';
   mysql> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
   IDENTIFIED BY 'NOVA_DBPASS';

* Edit the /etc/nova/nova.conf file and add these lines to the [database] section::

   ...
   [database]
   # The SQLAlchemy connection string used to connect to the database
   connection = mysql://nova:NOVA_DBPASS@controller/nova

* Create the tables for the Compute service::

   # nova-manage db sync

* Edit the /etc/nova/nova.conf file and add these lines to the [DEFAULT] section::

   ...
   [DEFAULT]
   ...
   my_ip=192.168.0.12
   vncserver_listen=192.168.0.12
   vncserver_proxyclient_address=192.168.0.12

  The IP address used here is the INTERNAL IP for controller node.

* Create a nova user that Compute uses to authenticate with the Identity Service. Use the *service* tenant and give the user the *admin* role::

   # keystone user-create --name=nova --pass=NOVA_PASS --email=nova@example.com
   # keystone user-role-add --user=nova --tenant=service --role=admin

* Edit the /etc/nova/nova.conf file and add these lines to the [DEFAULT] section::

   ...
   [DEFAULT]
   ...
   auth_strategy=keystone

* Add the credentials to the /etc/nova/api-paste.ini file. Add these options to the [filter:authtoken] section::

   [filter:authtoken]
   paste.filter_factory=keystoneclient.middleware.auth_token:filter_factory
   auth_host=controller
   auth_port=5000
   auth_protocol=http
   auth_uri=http://controller:5000/v2.0
   admin_tenant_name=service
   admin_user=nova
   admin_password=NOVA_PASS

* Register Compute with the Identity Service::

   # keystone service-create --name=nova --type=compute \
     --description="Nova Compute service"

* Use the id property that is returned to create the endpoint::

   # keystone endpoint-create \
     --service-id=the_service_id_above \
     --publicurl=http://controller:8774/v2/%\(tenant_id\)s \
     --internalurl=http://controller:8774/v2/%\(tenant_id\)s \
     --adminurl=http://controller:8774/v2/%\(tenant_id\)s

* Set these configuration keys to configure Compute to use the RabbitMQ message broker. Add them to the DEFAULT configuration group in the /etc/nova/nova.conf file::

   rpc_backend = nova.rpc.impl_kombu
   rabbit_host = controller
   rabbit_password = guest

* Restart Compute services::

   # service nova-api restart
   # service nova-cert restart
   # service nova-consoleauth restart
   # service nova-scheduler restart
   # service nova-conductor restart
   # service nova-novncproxy restart

* To verify your configuration, list available images::

   # nova image-list
   +--------------------------------------+--------------+--------+--------+
   | ID                                   | Name         | Status | Server |
   +--------------------------------------+--------------+--------+--------+
   | 0d192c86-1a92-4ac5-97da-f3d95f74e811 | CirrOS 0.3.1 | ACTIVE |        |
   +--------------------------------------+--------------+--------+--------+

3.2. Configure a Compute node
------------------------------

* Install the appropriate packages for the Compute service::

   # apt-get install nova-compute-kvm python-guestfs

  When prompted to create a supermin appliance, respond **yes**.

* Relax the restriction of guestfs::

   # chmod 0644 /boot/vmlinuz*

* Edit the /etc/nova/nova.conf configuration file and add these lines to the appropriate sections::

   ...
   [DEFAULT]
   ...
   auth_strategy=keystone

   rpc_backend = nova.rpc.impl_kombu
   rabbit_host = controller
   rabbit_password = guest

   my_ip=192.168.0.13
   vncserver_listen=0.0.0.0
   vncserver_proxyclient_address=192.168.0.13

   glance_host=controller

   network_api_class=nova.network.neutronv2.api.API
   neutron_url=http://controller:9696
   neutron_auth_strategy=keystone
   neutron_admin_tenant_name=service
   neutron_admin_username=neutron
   neutron_admin_password=NEUTRON_PASS
   neutron_admin_auth_url=http://controller:35357/v2.0
   firewall_driver=nova.virt.firewall.NoopFirewallDriver
   security_group_api=nova
   ...
   [database]
   # The SQLAlchemy connection string used to connect to the database
   connection = mysql://nova:NOVA_DBPASS@controller/nova

* Edit the /etc/nova/api-paste.ini file to add the credentials to the [filter:authtoken] section::

   [filter:authtoken]
   paste.filter_factory=keystoneclient.middleware.auth_token:filter_factory
   auth_host=controller
   auth_port = 35357
   auth_protocol = http
   admin_user=nova
   admin_tenant_name=service
   admin_password=NOVA_PASS

* Restart the Compute service::

   # service nova-compute restart

3.3. Launch an instance
------------------------

* Generate a keypair that consists of a private and public key to be able to launch instances on OpenStack. These keys are injected into the instances to make password-less SSH access to the instance::

   $ ssh-keygen
   $ cd .ssh
   $ nova keypair-add --pub_key id_rsa.pub mykey

  To view available keypairs::

   $ nova keypair-list
   +--------+-------------------------------------------------+
   |  Name  |                   Fingerprint                   |
   +--------+-------------------------------------------------+
   | mykey  | 7a:80:3a:c9:53:00:14:91:94:83:70:b3:2a:6d:da:0b |
   +--------+-------------------------------------------------+

* To see a list of the available flavors::

   $ nova flavor-list
   +----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
   | ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
   +----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
   | 1  | m1.tiny   | 512       | 1    | 0         |      | 1     | 1.0         | True      |
   | 2  | m1.small  | 2048      | 20   | 0         |      | 1     | 1.0         | True      |
   | 3  | m1.medium | 4096      | 40   | 0         |      | 2     | 1.0         | True      |
   | 4  | m1.large  | 8192      | 80   | 0         |      | 4     | 1.0         | True      |
   | 5  | m1.xlarge | 16384     | 160  | 0         |      | 8     | 1.0         | True      |
   +----+-----------+-----------+------+-----------+------+-------+-------------+-----------+

* Get the ID of the image to use for the instance::

   $ nova image-list
   +--------------------------------------+--------------+--------+--------+
   | ID                                   | Name         | Status | Server |
   +--------------------------------------+--------------+--------+--------+
   | 0d192c86-1a92-4ac5-97da-f3d95f74e811 | CirrOS 0.3.1 | ACTIVE |        |
   +--------------------------------------+--------------+--------+--------+

* To use SSH and ping, you must configure security group rules::

   # nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
   # nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0

* Launch the instance::

   nova boot --flavor 1 --key_name mykey --image 0d192c86-1a92-4ac5-97da-f3d95f74e811 --security_group default cirrOS


