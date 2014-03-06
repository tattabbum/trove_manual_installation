=================================
Manual Trove Installation - Havana
=================================

Scenario
========

1) The Openstack Havana environment has a Keystone service accepting only HTTPS connection.

2) The VM running Trove starts with a freshly Ubuntu 12.04.04 server.


Base configuration
==================

On the VM that will host the Trove service, execute a base configuration like the other Openstack services(http://docs.openstack.org/trunk/install-guide/install/apt/content/ch_basics.html) installing only the NTP protocol and set the password for Trove.

Installation
============

--------------------
Install dependencies
--------------------
Loggin as root user and execute the following commands:

* Install required packages::

	# apt-get install build-essential libxslt1-dev qemu-utils mysql-client git python-dev python-pexpect python-mysqldb libmysqlclient-dev

* Some packages in Ubuntu repo are outdated, so install their latest version from sources::

	# cd ~
	# wget https://pypi.python.org/packages/source/s/setuptools/setuptools-0.9.8.tar.gz
	# tar xfvz setuptools-0.9.8.tar.gz
	# cd setuptools-0.9.8
	# python setup.py install

	# cd ~
	# wget https://pypi.python.org/packages/source/p/pip/pip-1.4.1.tar.gz
	# tar xfvz pip-1.4.1.tar.gz
	# cd pip-1.4.1
	# python setup.py install

	# cd ~

You should use the last versions on https://pypi.python.org/packages/source/s/setuptools/ and https://pypi.python.org/packages/source/p/pip/ 

------------
Obtain Trove
------------

* Get Trove's sources from git::

	# git clone https://github.com/openstack/trove.git
	# git clone https://github.com/openstack/python-troveclient.git

-------------
Install Trove
-------------

* First install required python packages::

	# cd ~/trove
	# pip install -r requirements.txt

* Install Trove itself::

	# python setup.py develop

* Install Trove CLI::

	# cd ~/python-troveclient
	# python setup.py develop
	# cd ~
	
* We'll need glance client as well::

	# pip install python-glanceclient


Prepare OpenStack
-----------------
* Create a tenant "trove" and user "trove" with password "trove" to be used with Trove::

	Create tenant
	# keystone --os-username <OpenStackAdminUsername> --os-password <OpenStackAdminPassword>  \
           --os-tenant-name <OpenStackAdminTenant> \
           --os-auth-url http://<KeystoneIp>:35357/v2.0 \
           tenant-create --name trove

	Creat a user:
	# keystone --os-username <OpenStackAdminUsername> --os-password <OpenStackAdminPassword>  \
           --os-tenant-name <OpenStackAdminTenant> \
           --os-auth-url http://<KeystoneIp>:35357/v2.0 \
           user-create --name trove --pass trove --tenant trove

	Role, admin and user association:
	# keystone --os-username <OpenStackAdminUsername> --os-password <OpenStackAdminPassword>   \ 
           --os-tenant-name <OpenStackAdminTenant>  \
           --os-auth-url http://<KeystoneIp>:35357/v2.0 \
           user-role-add --user trove --tenant trove --role admin

	Adding the user trove to the tenant service (see the redstack function of devstack):
	# keystone --os-username <OpenStackAdminUsername> --os-password <OpenStackAdminPassword>   \ 
           --os-tenant-name <OpenStackAdminTenant>  \
           --os-auth-url http://<KeystoneIp>:35357/v2.0 \
           user-role-add --user trove --tenant service --role admin


* Create service trove::

	# key'stone --os-username <OpenStackAdminUsername> --os-password <OpenStackAdminPassword> \ 
           --os-tenant-name <OpenStackAdminTenant> \
           --os-auth-url http://<KeystoneIp>:35357/v2.0 \
           service-create --name trove --type database

* Create endpoint that points to trove. Pay attention to the use of quotes (')::

	# keystone --os-username <OpenStackAdminUsername> --os-password <OpenStackAdminPassword> \
           --os-tenant-name <OpenStackAdminTenant> \
           --os-auth-url http://<KeystoneIp>:35357/v2.0 endpoint-create \
           --service-id trove_service_id   \
           --publicurl 'http://IP_trove:8779/v1.0/$(tenant_id)s'   \
           --adminurl 'http://IP_trove:8779/v1.0/$(tenant_id)s'    \
           --internalurl 'http://IP_trove:8779/v1.0/$(tenant_id)s'
 
Prepare Trove configuration files
---------------------------------

There are several configuration files for Trove:

* api-paste.ini and trove.conf.sample - for trove-api
* trove-taskmanager.conf.sample - for trove-taskmanager
* trove-guestagent.conf.sample - for trove-guestagent
* <service_type>.cloudinit - cloudinit scripts for different service types. For now only mysql and percona are recognized as valid service types. NOTE: file names must exactly follow the pattern, e.g. 'mysql.cloudinit'

Samples of the above are available in $TROVE/trove/etc/trove/ as *.conf.sample files.
If a vanilla Ubuntu image used as a source image for Trove instances, then it is cloudinit script�s responsibility to install and run Trove guestagent in the instance.
As an alternative one may consider creating a custom image with pre-installed and pre-configured Trove in it.

* Edit the trove.conf.sample and trove-taskmanager.conf.sample files, adding the Rabbit Hostname for AMQP::

	# AMQP Connection info
	rabbit_password = PASSWORD_RABBIT
	rabbit_host = HOST_RABBIT

* Edit the api-paste.ini  file in order to set the CA path::

	...
	[filter:authtoken]
	# signing_dir is configurable, but the default behavior of the authtoken
	# middleware should be sufficient.  It will create a temporary directory
	# in the home directory for the user the trove process is running as.
	signing_dir = path_to_signing_dir (i.e. /root/trove/etc/trove)

* Edit all the trove configuration files iaccording to the rows in the Devstack�s Trove installation (see the redstack function of devstack).

* If Keystone accepts only HTTPS connections, in order to validate CA_file.pem of Keystone (SSL_504 error) you sholud modify:

	* the $TROVE_PATH/trove/trove/common/remote.py file in the rows 45 and 65, adding the cacert="/path/to/your/file.pem" ad last parameter in the .Client() function.

	* the /usr/local/lib/python2.7/dist-packages/keystoneclient/middleware/auth_token.py in the rows 720 and 725::
	
	720: print('#####self.ssl_ca_file', self.ssl_ca_file) 
	725: kwargs['verify'] = '/path/to/your/file.pem' 

Prepare image
-------------

* As the source image for trove instances, we will use a cloudinit-enabled vanilla Ubuntu image::

	# wget http://cloud-images.ubuntu.com/precise/current/precise-server-cloudimg-amd64-disk1.img

* Convert the downloaded image into uncompressed qcow2::

	# qemu-img convert -O qcow2 precise-server-cloudimg-amd64-disk1.img precise.qcow2

* Upload the converted image into Glance (using the Horizon interface)::

	# glance --os-username trove --os-password trove --os-tenant-name trove \
         --os-auth-url http://<KeystoneIp>:35357/v2.0 \
         image-create --name ubuntu_mysql --public --container-format ovf 
          --disk-format qcow2 
          --owner trove < precise.qcow2

Prepare database
----------------

* Create the datatabse.
	In the VM in which I have create the Trove's database (see the Havana's services configuration)::
		# mysql -u root -p
		mysql> CREATE DATABASE trove;
		mysql> GRANT ALL PRIVILEGES ON trove.* TO trove@'localhost' \
		IDENTIFIED BY 'TROVE_DBPASS';
		mysql> GRANT ALL PRIVILEGES ON trove.* TO trove@'%' \
		IDENTIFIED BY 'TROVE_DBPASS';

* Inizialize the database (see the redstack function of devstack)::
	
	# trove-manage --config-file=<PathToTroveConf> db_wipe mysql

	As an alternative, you can use:

	# trove-manage --config-file=<PathToTroveConf> db_sync

* Access to Trove's database and insert the following rows (see the redstack function of devstack)::

	mysql> INSERT INTO datastores VALUES ('a00000a0-00a0-0a00-00a0-000a000000aa', 'mysql', 
	'b00000b0-00b0-0b00-00b0-000b000000bb'); 

	mysql> INSERT INTO datastores values ('e00000e0-00e0-0e00-00e0-000e000000ee', 'Test_Datastore_1', '');

	mysql> INSERT INTO datastore_versions VALUES ('b00000b0-00b0-0b00-00b0-000b000000bb', 
  	'a00000a0-00a0-0a00-00a0-000a000000aa', 'mysql-5.5', 'c00000c0-00c0-0c00-00c0-000c000000cc', 
	'mysql-server-5.5', 1, 'mysql'); 

	mysql> INSERT INTO datastore_versions VALUES ('d00000d0-00d0-0d00-00d0-000d000000dd', 
	'a00000a0-00a0-0a00-00a0-000a000000aa', 'mysql_inactive_version', '', '', 0, 'manager1');


* Setup trove to use the uploaded image:

	Retrive id_image from nova::

	# nova --os-username trove --os-password trove --os-tenant-name trove --os-auth-url http://keystone_IP:5000/v2.0 image-list | awk '/ubuntu_mysql/ {print $2}'

	Update  datastore (see the redstack function of devstack)::

	# trove-manage --config-file=<PathToTroveConf> datastore_update mysql "" 

	# trove-manage --config-file=<PathToTroveConf> datastore_version_update mysql mysql-5.5 mysql image_id mysql-server-5.5 1

	# trove-manage --config-file=<PathToTroveConf> datastore_version_update mysql mysql_inactive_version manager1 image_id "" 0

	# trove-manage --config-file=<PathToTroveConf> datastore_update mysql mysql-5.5

	# trove-manage --config-file=<PathToTroveConf> datastore_update Test_Datastore_1 "" 


Run Trove
----------
Run the following commands::

	# trove-api --config-file=<PathToTroveConf> &

	# trove-taskmanager --config-file=<PathToTroveTaskmanager> &

	# trove-conductor --config-file=<PathToTroveConductor> &

Troubleshooting
---------------
No instance IPs in the output of "trove-cli instance get"

If Trove instance is created properly, is in the state ACTIVE, and is known for sure to be working, but there are no IP addresses for the instance in the output of �trove-cli instance get <id>�, then make sure the following lines are added to trove.conf:

add_addresses = True
network_label_regex = ^NETWORK_NAME$
where NETWORK_NAME should be replaced with real name of the nova network to which the instance is connected to.

One possible way to find the nova network name is to execute the �nova list� command. The output will list all Openstack instances for the tenant, including network information. Look for

NETWORK_NAME=IP_ADDRESS

Suggestions
-----------
