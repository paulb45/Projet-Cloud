Installation d'un noeud de contrôle
============================

Cette partie retrace la mise en place du noeud de contrôle sur une VM CentOS Stream 9 avec SELinux en enforcing et le firewall activé.

Nous utilisons la version Epoxy d'OpenStack (2025-1).

Nous avons majoritairement suivi la documentation officielle d’OpenStack pour une installation minimale. https://docs.openstack.org/fr/install-guide/openstack-services.html

# Préparation initiale

Préparation de *dnf* et installation des paquets de base d'OpenStack.

```bash
dnf install dnf-plugins-core
dnf config-manager --set-enabled crb

dnf install centos-release-openstack-epoxy

dnf upgrade
dnf install python3-openstackclient
dnf install openstack-selinux
```

Installation de la base de données pour les services.

```bash
curl -sS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | sudo bash
rpm --import https://supplychain.mariadb.com/MariaDB-Server-GPG-KEY

dnf install mariadb-server
systemctl enable --now mariadb

mysql_secure_installation
```

# Installation de Keystone

Keystone est l'IDP d'OpenStack.

## Configuration de la BDD
```bash
mysql -u root -p
```
```sql
CREATE DATABASE keystone;

GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'azerty';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'azerty';
```

## Installation de Keystone
```bash
dnf install openstack-keystone httpd uwsgi-plugin-python3 mod_wsgi
```

Configuration du fichier */etc/keystone/keystone.conf* pour y définir :
```
[database]
connection = mysql+pymysql://keystone:azerty@controller/keystone

[token]
provider = fernet
```

Préparer la BDD
```bash
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

## Configuration de Keystone

```bash
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

keystone-manage bootstrap --bootstrap-password azerty \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

## Configuration de Apache

Editer le fichier */etc/httpd/conf/httpd.conf* pour y définir :
```
ServerName controller
```
```bash
ln -s /usr/share/keystone/uwsgi-keystone.conf /etc/httpd/conf.d/
```
```bash
systemctl enable --now httpd.service
firewall-cmd --zone=public --add-port=5000/tcp --permanent
firewall-cmd --reload
```

## Environnement

Pour la suite, ajouter dans *.bashrc* les variables suivantes :
```bash
export OS_USERNAME=admin
export OS_PASSWORD=azerty
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```


# Installation de Glance

Glance est l'outil de gestion des images d'OpenStack.

## Configuration de la BDD
```bash
mysql -u root -p
```
```sql
CREATE DATABASE glance;

GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'azerty';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'azerty';
```

## Préparation de Glance dans Keystone

```bash
openstack user create --domain default --password-prompt glance
openstack role add --project service --user glance admin
openstack service create --name glance --description "OpenStack Image" image

openstack endpoint create --region RegionOne image public http://controller:9292
openstack endpoint create --region RegionOne image internal http://controller:9292
openstack endpoint create --region RegionOne image admin http://controller:9292

openstack role add --user glance --user-domain Default --system all reader
```

## Installation de Glance

```bash
dnf install openstack-glance
```

## Configuration de Glance

Editer le fichier */etc/glance/glance-api.conf* pour y définir :
```
[DEFAULT]
enabled_backends=fs:file

[database]
connection = mysql+pymysql://glance:azerty@controller/glance

[glance_store]
default_backend = fs
filesystem_store_datadir = /var/lib/glance/images

[keystone_authtoken]
www_authenticate_uri  = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = azerty

[oslo_limit]
auth_url = http://controller:5000
auth_type = password
user_domain_id = default
username = glance
system_scope = all
password = azerty
endpoint_id = cd6c3fef11344985bb45eca7873a5d77
region_name = RegionOne

[paste_deploy]
flavor = keystone
```

Préparer la BDD.
```bash
su -s /bin/sh -c "glance-manage db_sync" glance
```

```bash
systemctl enable openstack-glance-api.service
systemctl start openstack-glance-api.service
```



# Installation de Placement

Placement est le service de gestion des ressources d'OpenStack.

## Configuration de la BDD
```bash
mysql -u root -p
```
```sql
CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'azerty';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'azerty';
```

## Préparation de Placement dans Keystone
```bash
openstack user create --domain default --password-prompt placement
openstack role add --project service --user placement admin

openstack service create --name placement --description "Placement API" placement
openstack endpoint create --region RegionOne placement public http://controller:8778
openstack endpoint create --region RegionOne placement internal http://controller:8778
openstack endpoint create --region RegionOne placement admin http://controller:8778
```

## Installation de Placement

```bash
dnf install openstack-placement-api
```

## Configuration de Placement

Editer le fichier */etc/placement/placement.conf* et définir :
```
[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = azerty

[placement_database]
connection = mysql+pymysql://placement:azerty@controller/placement
```

## Apache
```
systemctl restart httpd
```


# Installation de Nova

Nova sur le controller sert pour gérer les VMs entre les différents Compute (Nova).

## Configuration de la BDD
```bash
mysql -u root -p
```
```sql
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'azerty';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'azerty';

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'azerty';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'azerty';

GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'azerty';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'azerty';
```

## Configuration de Nova dans Keystone

```bash
openstack user create --domain default --password-prompt nova
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute

openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
```

## Installation de Nova

```bash
dnf install -y openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler
```

## Configuration de nova

Editer le fichier */etc/nova/noca.conf* et définir :
```
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:azerty@controller:5672/
my_ip = 10.1.5.11

[api]
auth_strategy=keystone

[api_database]
connection = mysql+pymysql://nova:azerty@controller/nova_api

[database]
connection = mysql+pymysql://nova:azerty@controller/nova

[glance]
api_servers = http://controller:9292

[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = azerty

[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = azerty
service_metadata_proxy = true
metadata_proxy_shared_secret = azerty

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = azerty

[service_user]
send_service_user_token = true
auth_url = http://controller:5000/v3
auth_strategy = keystone
auth_type = password
project_domain_name = Default
project_name = service
user_domain_name = Default
username = nova
password = azerty

[vnc]
enabled=true
server_listen=0.0.0.0
server_proxyclient_address=10.1.5.21
novncproxy_base_url = http://controller:6080/vnc_auto.html
```

Préparation de la BDD
```bash
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
su -s /bin/sh -c "nova-manage db sync" nova
```

```
systemctl enable \
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service

systemctl start \
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service
```

# Installation de Neutron

Neutron est le service réseau d'OpenStack.

## Configuration de la BDD

```bash
mysql -u root -p
```
```sql
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'azerty';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'azerty';
```

## Instalation de Neutron
dnf install openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch

## Configuration de Neutron

Editer le fichiet */etc/neutron/neutron.conf* et définir :
```
[DEFAULT]
core_plugin = ml2
service_plugins =
transport_url = rabbit://openstack:azerty@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[database]
connection = mysql+pymysql://neutron:azerty@controller/neutron

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = azerty

[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = nova
password = azerty

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

Neutron est installé suivant l'option 1 du guide pour simplifier l'installation.

### Configuration du plugin ML2

Editer le fichier */etc/neutron/plugins/ml2/ml2_conf.ini* et définir :
```
[ml2]
type_drivers = flat,vlan
tenant_network_types =
mechanism_drivers = openvswitch
extension_drivers = port_security

[ml2_type_flat]
flat_networks = provider
```

### Configuration de l'agent openvswith

Avant cela, il faut avoir configurer un brige ovs sur la VM.

```bash
ovs-vsctl add-br br-ex
ovs-vsctl add-port br-ex ensX
```

NetworkManager de fonctionne pas avec openvshitch donc il faut passer par les fichiers de /etc/sysconfig/network-scripts/ pour configurer notre bridge.

Une fois une IP valide sur le bridge, editer le fichier */etc/neutron/plugins/ml2/openvswitch_agent.ini* et définir :
```
[ovs]
bridge_mappings = provider:br-ex

[securitygroup]
enable_security_group = true
firewall_driver = openvswitch
```

### Configuration de l'agent DHCP

Configurer le fichier */etc/neutron/dhcp_agent.ini* et définir :
```
[DEFAULT]
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```


# Installation d'Horizon

Horizon est le dashboard pour la web admin d'OpenStack. Ce service n'est pas obligatoire car nous pouvons passer directement en CLI.

## Installation d'Horizon

```bash
dnf install openstack-dashboard
```

## Configuration d'Horizon

Editer le fichier */etc/openstack-dashboard/local_settings* et définir : 
```
OPENSTACK_HOST = "controller"
ALLOWED_HOSTS = ['*']

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
        'default': {
            'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
            'LOCATION': 'controller:11211',
        }
}

OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

OPENSTACK_API_VERSIONS = {
        "identity": 3,
        "image": 2,
        "volume": 3,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"

OPENSTACK_NEUTRON_NETWORK = {
        'enable_router': False,
        'enable_quotas': False,
        'enable_distributed_router': False,
        'enable_ha_router': False,
        'enable_fip_topology_check': False,
}
```

Ajouter cette ligne dans le fichier */etc/httpd/conf.d/openstack-dashboard.conf* :
```
WSGIApplicationGroup %{GLOBAL}
```

```
systemctl restart httpd.service memcached.service
```

# Installation de RabbitMQ

RabbitMQ est un service pour la communication entre les noeuds OpenStack.

## Installation de RabbitMQ

```bash
dnf install -y rabbitmq-server
```

## Configuration de RabbitMQ

```bash
systemctl enable --now rabbitmq-server
```
```bash
rabbitmqctl add_user openstack azerty
rabbitmqctl set_permissions -p / openstack ".*" ".*" ".*"
```
```bash
firewall-cmd --zone=public --add-port=8778/tcp --permanent
firewall-cmd --reload
```


# Déployer une instance

Avant de faire cette partie, il faut qu'un compute soit disponible. Cette partie peut être réalisée sur l'interface web.

```bash
openstack hypervisor list
```

## Créer le réseau (exemple)

```bash
openstack network create reseau-test \
    --provider-network-type flat \
    --provider-physical-network provider \ 
    --external

openstack subnet create subnet-test \
    --network reseau-test \
    --subnet-range 192.168.128.0/24 \
    --gateway 192.168.128.254 \
    --allocation-pool start=192.168.128.2,end=192.168.128.250 \
    --dns-nameserver 8.8.8.8
```

## Créer une image (exemple)

A partir d'un disque qcow2.
```bash
openstack image create "VM31" \
    --file /home/user/VM31.qcow2 \  
    --disk-format qcow2 \
    --container-format bare \  
    --public
```

## Créer un gabarit (exemple)

```bash
openstack flavor create \
    --ram 2048 \
    --disk 20 \
    --vcpus 2 \ 
    flavor-small
```

## Créer une instance (exemple)

```bash
openstack server create \
    --flavor flavor-small \  
    --image VM31 \
    --network reseau-test \   
    vm-test
```