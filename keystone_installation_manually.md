Keystone Installation Manually
==============================

## Prerequisites

* Environment Requirements Installation

   ```
   # yum -y install gcc gcc-c++ libffi-devel libxml2-devel libxslt-devel libyaml-devel openldap-devel openssl-devel postgresql postgresql-devel python-devel sqlite-devel zip MySQL-python mariadb-devel python-pip

   # yum install centos-release-openstack-queens
   # yum -y install openstack-selinux
   ```

   Once you find "No packages named python-pip", you can install epel-release to resolve it.

* Database(Mariadb) Installation and Configuration

   1. Install the packages:

      ```
      # yum install mariadb mariadb-server python2-PyMySQL
      ```
   2. Config mariadb

      Create and edit the /etc/my.cnf.d/openstack.cnf file (backup existing configuration files in /etc/my.cnf.d/ if needed) and complete the following actions:

         * Create a [mysqld] section, and set the bind-address key to the management IP address of the controller node to enable access by other nodes via the management network. Set additional keys to enable useful options and the UTF-8 character set:

            ```ini
            [mysqld]
            bind-address = 0.0.0.0

            default-storage-engine = innodb
            innodb_file_per_table = on
            max_connections = 4096
            collation-server = utf8_general_ci
            character-set-server = utf8
            ```

   3. Finalize installation

      + Start the database service and configure it to start when the system boots:

         ```
         # systemctl enable mariadb.service
         # systemctl start mariadb.service
         ```

      + Secure the database service by running the mysql_secure_installation script. In particular, choose a suitable password for the database root account:

         ```
         # mysql_secure_installation
         ```

## Install Keystone from source code


* Installation by using pip. *Assume that your keystone source location path is at "~/keystone"*.

   ```
   # pip --no-cache-dir install --upgrade -c https://git.openstack.org/cgit/openstack/requirements/plain/upper-constraints.txt?h=stable/queens -e ~/keystone/.[ldap,memcache,mongodb] ~/keystone
   ```

* Config keystone database

   1. Use the database access client to connect to the database server as the root user:

      ```
      $ mysql -u root -p
      ```

   2. Create the keystone database:

      ```
      MariaDB [(none)]> CREATE DATABASE keystone;
      ```

   3. Grant proper access to the keystone database:

      ```
      MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
      IDENTIFIED BY 'KEYSTONE_DBPASS';
      MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
      IDENTIFIED BY 'KEYSTONE_DBPASS';
      ```

* Config /etc/keyston.conf

   ```
   # mkdir /etc/keystone
   # cp ~/keystone/etc/keystone.conf.sample /etc/keystone/keystone.conf
   # cp ~/keystone/etc/policy.v3cloudsample.json /etc/keystone/policy.json
   # cp ~/keystone/etc/keystone-paste.ini /etc/keystone/
   # cp ~/keystone/etc/sso_callback_template.html /etc/keystone/
   ```

   ```ini
   [database]
   #...
   connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@current_host_ip/keystone
   ```

* Populate the identity service database.

   1. create user 'keystone'

      ```
      # groupadd keystone
      # useradd -g keystone keystone
      ```

   2. db sync

      ```
      # su -s /bin/sh -c "keystone-manage db_sync" keystone
      ```

* Initialize Fernet key repositories:

   ```
   # keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
   # keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
   ```

* Bootstrap the identity service:

   ```
   # keystone-manage bootstrap --bootstrap-password ADMIN_PASSWORD \
     --bootstrap-admin-url http://current_host_ip:35357/v3/ \
     --bootstrap-internal-url http://current_host_ip:5000/v3/ \
     --bootstrap-public-url http://current_host_ip:5000/v3/ \
     --bootstrap-region-id RegionOne
   ```

## Configure the Apache HTTP server

   1. Install apache http server

      ```
      # yum install httpd mod_wsgi
      ```

   2. config httpd wsgi-keystone.conf

      ```
      # cp ~/keystone/httpd/wsgi-keystone.conf /etc/httpd/conf.d/
      ```

      Modify /usr/local/bin to /usr/bin in /etc/httpd/conf.d/wsgi-keystone.conf

   3. finalize the installation

      ```
      # systemctl enable httpd.service
      # systemctl start httpd.service
      ```


## Verify the Installation

* create openrc.v3.domain:

   ```
   export OS_USERNAME=admin
   export OS_PASSWORD=ADMIN_PASS
   export OS_USER_DOMAIN_NAME=Default
   export OS_DOMAIN_NAME=Default
   export OS_AUTH_URL=http://current_host_ip:35357/v3
   export OS_IDENTITY_API_VERSION=3
   ```

* verify:

   Use admin domain scope openrc.
   ```
   # source /path/to/operc.v3.domain
   ```

   Install openstackclient in another python virtual environment.
   ```
   # install openstackclient
   # virtualenv osc
   # source osc/bin/activate
   # pip install python-openstack
   ```

   ```
   # openstack user list

   +----------------------------------+-------+
   | ID                               | Name  |
   +----------------------------------+-------+
   | 0af810c9d972427db0274ecc9360f224 | admin |
   +----------------------------------+-------+
   ```

## Reference

+ [Documentation of Openstack Keystone](https://docs.openstack.org/keystone/queens/install/keystone-install-rdo.html)
+ [Documentation of Openstack Installation Guide](https://docs.openstack.org/install-guide/environment.html)
+ [Reference Guide of pip](https://pip.pypa.io/en/stable/reference/pip_install/#requirements-file-format)