Installation d'un noeud compute
============================

Cette partie retrace la mise en place du noeud de contrôle sur une VM CentOS Stream 9 avec SELinux en enforcing et le firewall activé. La VM à un CPU de type "**host**".

Nous utilisons la version Epoxy d'OpenStack (2025-1).

Nous avons majoritairement suivi la documentation officielle d’OpenStack pour une installation minimale. https://docs.openstack.org/nova/2025.1/install/compute-install-rdo.html


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

# Installation de Nova

## Instalation de Nova

```bash
dnf install openstack-nova-compute libvirt qemu-kvm
```

## Configuration de Nova

Editer le fichier */etc/nova/nova.conf* et définir :
```
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:azerty@controller
my_ip = 10.1.5.21
enabled = True
compute_driver=libvirt.LibvirtDriver
instances_path = /var/lib/nova/instances

[api]
auth_strategy=keystone

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

[libvirt]
compute_uuid_file=/var/lib/nova/compute_id
virt_type = kvm
image_type = default

[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = azerty

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
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = 10.1.5.21
novncproxy_base_url = http://controller:6080/vnc_auto.html
```

```bash
systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service
```

Ouverture pour le VNC.
```bash
firewall-cmd --zone=public --add-port=5900-5999/tcp --permanent
firewall-cmd --reload
```


# Installation de Neutron

## Installation de Netron

```bash
dnf install -y openstack-neutron-openvswitch openvswitch
```

## Préparation bridge

```bash
systemctl enable --now openvswitch
```

```bash
ovs-vsctl add-br br-ex
ovs-vsctl add-port br-ex ens18
dhclient br-ex
```

## Configuration de neutron

Editer le fichier */etc/neutron/neutron.conf* et définir :
```
[DEFAULT]
transport_url = rabbit://openstack:azerty@controller

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = azerty

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

```bash
systemctl enable neutron-openvswitch-agent.service
systemctl start neutron-openvswitch-agent.service
```