# install openstack network node 

Disclaimer : The following wasn't generated using AI. I just prefer to use english

the following describes how to set up a network node on a CentOS9 machine.
Please note that a network node is distinct from a neutron node. the neutron node hosts the neutron server whereas a network node only manages the network connection for openstack. in a minimal installation the controller manages the neutron server. Hence there is no need for neutron node. 
However for security reasons it's a good practice to manage the network from a different node than the controller.
which is why we use a network node.

for this installation we require two network interfaces with an internet connextion
one network interface will be dubbed the gateway interface 
the other is the local interface

## updating centOS 

before proceeding further ensure your system is up to date 

    dnf install -y https://www.rdoproject.org/repos/epoxy/rdo-release-epoxy.rpm
    
    dnf clean all
    
    dnf makecache

    
    dnf install dnf-plugins-core

    dnf upgrade 

## instaling the necessary packages

    dnf install -y centos-release-openstack-epoxy
    
    dnf install -y python3-openstackclient

    dnf install -y openstack-selinux

    dnf install -y openstack-neutron

    dnf install -y ebtables

    dnf install -y ipset

    dnf install -y https://www.rdoproject.org/repos/epoxy/rdo-release-epoxy.rpm

    dnf install -y openstack-neutron openstack-neutron-ml2

    dnf install -y bridge-utils

    dnf install -y openvswitch ovn ovn-central ovn-host openstack-neutron-ovn-metadata-agent

    dnf install -y dhclient

## network config 1

    bash -c 'echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf'
    bash -c 'echo "net.ipv4.conf.all.rp_filter=0" >> /etc/sysctl.conf'
    bash -c 'echo "net.ipv4.conf.default.rp_filter=0" >> /etc/sysctl.conf'
    sysctl -p

    systemctl enable --now openvswitch
    systemctl enable --now ovn-controller

    ovs-vsctl add-br br-provider
    ovs-vsctl add-port br-provider ens19 

            # ens19 is the gateway interface

# modifier les fichiers de conf

inside /etc/neutron/plugins/ml2/ml2_conf.ini # make a .bak beforehand to feel safe

    [ml2]
    type_drivers = geneve,flat,vlan
    tenant_network_types = geneve
    mechanism_drivers = ovn
    extension_drivers = port_security

    [ml2_type_flat]
    flat_networks = provider

    [ml2_type_geneve]
    vni_ranges = 1:65536

    [ovn]
    ovn_nb_connection = tcp:CONTROLLER_IP:6641
    ovn_sb_connection = tcp:CONTROLLER_IP:6642
    ovn_l3_scheduler = leastloaded


inside /etc/neutron/neutron.conf # same as before .bak just in case

btw 10.1.5.11 is controller ip. make sure the firewall as the necesarry ports open on the controller

    [DEFAULT]
    transport_url = rabbit://openstack:azerty@10.1.5.11
    core_plugin = ml2

    [keystone_authtoken]
    www_authenticate_uri = http://10.1.5.11:5000/
    auth_url = http://10.1.5.11:5000/
    memcached_servers = 10.1.5.11:11211
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = neutron
    password = azerty

    [oslo_concurrency]
    lock_path = /var/lib/neutron/tmp

inside /etc/neutron/metadata_agent.ini 

    [DEFAULT]
    auth_url = http://10.1.5.11:5000
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    region_name = RegionOne
    project_name = service
    username = neutron
    password = azerty
    nova_metadata_host = 10.1.5.11
    metadata_proxy_shared_secret = azerty

now lets start tthe services

    ovs-vsctl set Open_vSwitch . external_ids:ovn-encap-type=geneve
    ovs-vsctl set Open_vSwitch . external_ids:ovn-encap-ip=10.1.5.154

    ovs-vsctl set Open_vSwitch . external_ids:ovn-remote=tcp:10.1.5.11:6642

    systemctl restart ovn-controller

    systemctl status ovn-controller

    ovn-sbctl show


# the bridge

try turning on the bridge

    dhclient br-provider

if you have an IP address and network connectivity that's it ! 

else !

check the gateway interface is tied to the bridge 

check the routes ensure the bridge is the default

check you still have network connectivity

might have to modify etc/resolv.conf 








