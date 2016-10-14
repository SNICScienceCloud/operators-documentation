# Openstack Liberty Deployment at HPC2N

## General

Openstack Ansible was used for the Openstack deployment at HPC2N.

## System map

[![System map](img/ssc_hpc2n.png)](ssc_hpc2n.dia)

## Hardware setup

### Compute nodes x 10

- Dell PowerEdge R630
- 2 x 12 Core Intel(R) Xeon(R) CPU E5-2670 v3 @ 2.30GHz
- 128G RAM
- 2 x 300G SAS Disks in a RAID 1 mirror.

### Control nodes x 3

- Dell PowerEdge R630
- 1 x 8 Core Intel(R) Xeon(R) CPU E5-2630 v3 @ 2.40GHz
- 32G RAM
- 2 x 300G SAS Disks in a RAID 1 mirror.

### Storage nodes x 4

- HP ProLiant DL380e Gen8
- 2 x 4 Core Intel(R) Xeon(R) CPU E5-2407 0 @ 2.20GHz 
- 56G RAM
- 2 x 500G part of 4T SATA Disks in a RAID 1 mirror.
- 23T for Ceph storage*


Note*: Two storage nodes have 6 x 4T Disks, Two storage nodes have 5 x 4T Disks + 2 x 2T Disks. But the total amout of storage on the nodes are identical.

## Node setup

All nodes run ubuntu 14.04 and they are installed with Puppet, the HPC2N standard for server installation.

## Openstack Ansible

The storage node u-sn-o38 is also used as deployment-host for openstack-asible.
Kerberos is used for access to destination hosts. SSH-Keys are used for containers.

The rest of the setup follows openstack-ansible refrence manual for installation.

The following configuraton files were used for deployment.

- [openstack_user_config.yml](hpc2n_conf/openstack_user_config.yml)
- [user_variables.yml](hpc2n_conf/user_variables.yml)


## Network

Linux-bridges are used for the for the vlan and vxlan traffic.

There is a IPv6 network reserved but not yet configured.

### Compute node interfaces

/etc/network/interfaces.d/bridge.interfaces

    auto eth2
    iface eth2 inet manual
    
    # Container management VLAN interface
    iface eth2.10 inet manual
        vlan-raw-device eth2
    
    auto eth3
    iface eth3 inet manual
    
    # OpenStack Networking VXLAN (tunnel/overlay) VLAN interface
    iface eth3.20 inet manual
        vlan-raw-device eth3
    
    # Container management bridge
    auto br-mgmt
    iface br-mgmt inet static
        bridge_stp off
        bridge_waitport 0
        bridge_fd 0
        # Bridge port references tagged interface
        bridge_ports eth2.10
        address 172.16.2.3X
        netmask 255.255.255.0
    
    # OpenStack Networking VXLAN (tunnel/overlay) bridge
    auto br-vxlan
    iface br-vxlan inet static
        bridge_stp off
        bridge_waitport 0
        bridge_fd 0
        # Bridge port references tagged interface
        bridge_ports eth3.20
        address 10.10.0.3X
        netmask 255.255.0.0
    
    # OpenStack Networking VLAN bridge
    auto br-vlan
    iface br-vlan inet manual
        bridge_stp off
        bridge_waitport 0
        bridge_fd 0
        # Bridge port references untagged interface
        bridge_ports eth3
    
    auto br-storage
    iface br-storage inet static
        # Completely disable IPv6 router advertisements/autoconfig
        pre-up sysctl -q -e -w net/ipv6/conf/br-storage/accept_ra=0
        bridge_stp off
        bridge_waitport 0
        bridge_fd 0
        bridge_ports eth2
        address 172.16.1.3X
        netmask 255.255.255.0
        broadcast 172.16.1.255

- u-cn-o05 has IP suffix .30
- u-cn-o06 has IP suffix .31
- u-cn-o07 has IP suffix .32
- u-cn-o10 has IP suffix .33
- u-cn-o11 has IP suffix .34
- u-cn-o12 has IP suffix .35
- u-cn-o13 has IP suffix .36
- u-cn-o14 has IP suffix .37

### Management node interfaces

/etc/network/interfaces.d/bridge.interfaces

    # Container management VLAN interface
    iface bond0.10 inet manual
        vlan-raw-device bond0
    
    # OpenStack Networking VXLAN (tunnel/overlay) VLAN interface
    iface bond1.20 inet manual
        vlan-raw-device bond1
    
    # Container management bridge
    auto br-mgmt
    iface br-mgmt inet static
        bridge_stp off
        bridge_waitport 0
        bridge_fd 0
        # Bridge port references tagged interface
        bridge_ports bond0.10
        address 172.16.2.X
        netmask 255.255.255.0
    
    # OpenStack Networking VXLAN (tunnel/overlay) bridge
    auto br-vxlan
    iface br-vxlan inet manual
        bridge_stp off
        bridge_waitport 0
        bridge_fd 0
        # Bridge port references tagged interface
        bridge_ports bond1.20
    
    # OpenStack Networking VLAN bridge
    auto br-vlan
    iface br-vlan inet manual
        bridge_stp off
        bridge_waitport 0
        bridge_fd 0
        # Bridge port references untagged interface
        bridge_ports bond1
    
    auto br-storage
    iface br-storage inet static
        bridge_stp off
        bridge_waitport 0
        bridge_fd 0
        bridge_ports bond0
        address 172.16.1.X
        netmask 255.255.255.0

- u-mn-o24 has IP suffix .2
- u-mn-o25 has IP suffix .3
- u-mn-o26 has IP suffix .4


/etc/network/interfaces.d/bond0.interfaces

    auto eth2
    iface eth2 inet manual
        bond-master bond0
    
    auto eth4
    iface eth4 inet manual
        bond-master bond0
    
    auto bond0
    iface bond0 inet manual
        bond-slaves eth2 eth4
        bond-mode active-backup
        bond-miimon 100
        bond-downdelay 200
        bond-updelay 200

/etc/network/interfaces.d/bond1.interfaces

    auto eth3
    iface eth3 inet manual
        bond-master bond1
    
    auto eth5
    iface eth5 inet manual
        bond-master bond1
    
    auto bond1
    iface bond1 inet manual
        bond-slaves eth1 eth3
        bond-mode active-backup
        bond-miimon 100
        bond-downdelay 250
        bond-updelay 250


### Floating IPv4 Pool

The floating IPv4 network pool is a full class C network (130.239.81.0/24) with the gateway 130.239.81.254.

Created with:

    neutron net-create floating-130.239.81 --shared --router:external=True --provider:network_type vlan --provider:segmentation_id 3 --provider:physical_network vlan

    neutron subnet-create  --allocation-pool start=130.239.81.1,end=130.239.81.253 --gateway 130.239.81.254 --disable-dhcp --name floating-130.239.81 --ip-version 4 --dns-nameserver 130.239.1.90 *EXTERNAL_NETWORK_CIDR* 130.239.81.0/24

## Manual config changes required for region interation

**These are the settings for the test-cloud with self-signed cert on the keystone**

### Nova

#### Modifications for TLS on public endpoint to `/etc/haproxy/conf.d/nova_api_os_compute` on `haproxy_hosts`

Replace the * binding

    bind *:8774

With a dual bind using the load balancer internal VIP (without TLS) and the load balancer external VIP (with TLS enabled and SSLv3 disabled).
    
    bind 172.16.2.1:8774
    bind 130.239.46.130:8774 ssl crt /etc/ssl/private/haproxy.pem no-sslv3
        http-request set-header X-Forwarded-Proto https if { ssl_fc }

#### Modifications to `/etc/nova/nova.conf` on `compute_hosts`

Config settings for Nova to authenticate with Neutron credentials

    [neutron]
    insecure = True
    auth_plugin = password
    auth_url = https://130.238.29.249:35358
    auth_uri = https://130.238.29.249:5443
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = neutron-HPC2N
    password = **SECRET**
    region_name = UPPMAX

Config settings for Nova to communicate with the correct Cinder instance.

Locate the correct cinder endpoint with `openstack endpoint list` and the catalog_info format is "Service Type:Service Name:Interface URL Type".

    [cinder]
    catalog_info = volumev2:cinderv2-HPC2N:internalURL
    os_region_name = HPC2N

Config settings for Nova authentication

    [keystone_authtoken]
    insecure = True
    auth_plugin = password
    auth_url = https://130.238.29.249:35358
    auth_uri = https://130.238.29.249:5443
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = nova-HPC2N
    password = **SECRET**
    region_name = UPPMAX

#### Modifications to `/etc/nova/nova.conf` on `nova_api_os_compute_container`, `nova_api_metadata_container`, `nova_console_container`, `nova_cert_container` and `nova_conductor_container`

Config settings for Nova to communicate with the correct Cinder instance.

Locate the correct cinder endpoint with `openstack endpoint list` and the catalog_info format is "Service Type:Service Name:Interface URL Type".

    [cinder]
    catalog_info = volumev2:cinderv2-HPC2N:internalURL
    os_region_name = HPC2N

Config settings for Nova to authenticate with Neutron credentials

    [neutron]
    insecure = True
    auth_plugin = password
    auth_url = https://130.238.29.249:35358
    auth_uri = https://130.238.29.249:5443
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = neutron-HPC2N
    password = **SECRET**
    region_name = UPPMAX

Config settings for Nova authentication

    [keystone_authtoken]
    insecure = True
    auth_plugin = password
    auth_url = https://130.238.29.249:35358
    auth_uri = https://130.238.29.249:5443
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = nova-HPC2N
    password = **SECRET**
    region_name = UPPMAX

### Ceilometer

#### Modifications for TLS on public endpoint to `/etc/haproxy/conf.d/ceilometer_api` on `haproxy_hosts`

Replace the * binding

    bind *:8777

With a dual bind using the load balancer internal VIP (without TLS) and the load balancer external VIP (with TLS enabled and SSLv3 disabled).
    
    bind 172.16.2.1:8777
    bind 130.239.46.130:8777 ssl crt /etc/ssl/private/haproxy.pem no-sslv3
        http-request set-header X-Forwarded-Proto https if { ssl_fc }

#### Modifications to `/etc/ceilometer/ceilometer.conf` on `ceilometer_api_container` and `compute_hosts`

Config settings for Ceilometer authentication

    [keystone_authtoken]
    insecure = True
    auth_plugin = password
    signing_dir = /var/lib/glance/cache/api
    identity_uri = https://130.238.29.249:35358
    auth_url = https://130.238.29.249:35358
    auth_uri = https://130.238.29.249:5443
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = ceilometer-HPC2N
    password = **SECRET**
    region_name = UPPMAX

Config settings for Ceilometer service

    [service_credentials]
    os_auth_url = https://130.238.29.249:35358
    os_auth_uri = https://130.238.29.249:5443
    os_username = ceilometer-HPC2N
    os_tenant_name = service
    os_password = **SECRET**
    os_region_name = HPC2N

#### Modifications to `/etc/ceilometer/ceilometer.conf` on `ceilometer_collector_container`

Config settings for Ceilometer authentication

    [keystone_authtoken]
    insecure = True
    auth_plugin = password
    signing_dir = /var/lib/glance/cache/api
    identity_uri = https://130.238.29.249:35358
    auth_url = https://130.238.29.249:35358
    auth_uri = https://130.238.29.249:5443
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = ceilometer-HPC2N
    password = **SECRET**
    region_name = UPPMAX

### Cinder

#### Modifications for TLS on public endpoint to `/etc/haproxy/conf.d/cinder_api` on `haproxy_hosts`

Replace the * binding

    bind *:8776

With a dual bind using the load balancer internal VIP (without TLS) and the load balancer external VIP (with TLS enabled and SSLv3 disabled).

    bind 172.16.2.1:8776
    bind 130.239.46.130:8776 ssl crt /etc/ssl/private/haproxy.pem no-sslv3
        http-request set-header X-Forwarded-Proto https if { ssl_fc }

#### Modifications to `/etc/cinder/cinder.conf` on `cinder_api_container`, `cinder_scheduler_container` and `cinder_volumes_container`

Config settings for Cinder to communicate with the correct Nova instance.

Locate the correct nova endpoint with `openstack endpoint list` and the catalog_info format is "Service Type:Service Name:Interface URL Type".

    [DEFAULT]
    nova_catalog_info = compute:nova-HPC2N:internalURL
    nova_catalog_admin_info = compute:nova-HPC2N:adminURL

Config encryption_auth_url to use the correct keystone

    [keymgr]
    encryption_auth_url = https://130.238.29.249:5443/v3

Config settings for Cinder authentication

    [keystone_authtoken]
    insecure = True
    auth_plugin = password
    signing_dir = /var/cache/cinder
    auth_url = https://130.238.29.249:35358
    auth_uri = https://130.238.29.249:5443
    project_domain_id = default
    user_domain_id = default
    project_name = service
    region_name = UPPMAX
    username = cinder-HPC2N
    password = **SECRET**


### Neutron 

#### Modifications for TLS on public endpoint to `/etc/haproxy/conf.d/neutron_server` on `haproxy_hosts`

Replace the * binding

    bind *:9696

With a dual bind using the load balancer internal VIP (without TLS) and the load balancer external VIP (with TLS enabled and SSLv3 disabled).
    
    bind 172.16.2.1:9696
    bind 130.239.46.130:9696 ssl crt /etc/ssl/private/haproxy.pem no-sslv3
        http-request set-header X-Forwarded-Proto https if { ssl_fc }

#### Modifications to `/etc/neutron/metadata_agent.ini` on `neutron_agents_container`

Config settings for Neutron authentication

    [default]
    auth_plugin = password
    auth_url = https://130.238.29.249:35358
    auth_uri = https://130.238.29.249:5443
    auth_region = UPPMAX
    admin_tenant_name = service
    admin_user = neutron-HPC2N
    admin_password = **SECRET**
    endpoint_type = publicURL

#### Modifications to `/etc/neutron/neutron.conf` on `neutron_server_container`

Config settings for Neutron to authenticate with Nova credentials

    [nova]
    insecure = True
    project_name=services
    username=nova-HPC2N
    password=**SECRET**
    project_domain_id=default
    tenant_name=services
    user_domain_id=default
    auth_url=https://130.238.29.249:35358/
    region_name=UPPMAX

Config settings for Neutron authentication

    [keystone_authtoken]
    insecure = True
    auth_plugin = password
    auth_url = https://130.238.29.249:35358
    auth_uri = https://130.238.29.249:5443
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = neutron-HPC2N
    password = **SECRET**
    region_name = UPPMAX


### Glance

#### Modifications for TLS on public endpoint to `/etc/haproxy/conf.d/glance_api` on `haproxy_hosts`

Replace the * binding

    bind *:9292

With a dual bind using the load balancer internal VIP (without TLS) and the load balancer external VIP (with TLS enabled and SSLv3 disabled).

    bind 172.16.2.1:9292
    bind 130.239.46.130:9292 ssl crt /etc/ssl/private/haproxy.pem no-sslv3
        http-request set-header X-Forwarded-Proto https if { ssl_fc }

#### Modifications to `/etc/glance/glance-api.conf` and `/etc/glance/glance-registry.conf` on `glance_container`

Config settings for Glance authentication

    [keystone_authtoken]
    insecure = True
    auth_plugin = password
    signing_dir = /var/lib/glance/cache/api
    auth_url = https://130.238.29.249:35358
    auth_uri = https://130.238.29.249:5443
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = glance-HPC2N
    password = **SECRET**
    region_name = UPPMAX

#### Modifications to `/etc/glance/glance-cache.conf` on `glance_container`

    auth_url = https://130.238.29.249:35358
    auth_uri = https://130.238.29.249:5443
    admin_user = glance-HPC2N
    admin_password = **SECRET**

### Heat

#### Modifications for TLS on public endpoint to `/etc/haproxy/conf.d/heat_api` on `haproxy_hosts`

Replace the * binding

    bind *:8004

With a dual bind using the load balancer internal VIP (without TLS) and the load balancer external VIP (with TLS enabled and SSLv3 disabled).
    
    bind 172.16.2.1:8004
    bind 130.239.46.130:8004 ssl crt /etc/ssl/private/haproxy.pem no-sslv3
        http-request set-header X-Forwarded-Proto https if { ssl_fc }

#### Modifications for TLS on public endpoint to `/etc/haproxy/conf.d/heat_api_cfn` on `haproxy_hosts`

Replace the * binding

    bind *:8000

With a dual bind using the load balancer internal VIP (without TLS) and the load balancer external VIP (with TLS enabled and SSLv3 disabled).
    
    bind 172.16.2.1:8000
    bind 130.239.46.130:8000 ssl crt /etc/ssl/private/haproxy.pem no-sslv3
        http-request set-header X-Forwarded-Proto https if { ssl_fc }

#### Modifications to `/etc/heat/heat.conf` on `heat_apis_container` and `heat_engine_container`

Configure Client keystone to use the correct keystone and endpoint type

    [clients_keystone]
    insecure = True
    endpoint_type = publicURL
    auth_uri = https://130.238.29.249:5443

Configure EC2 Auth Token to use the correct keystone

    [ec2authtoken]
    auth_uri = https://130.238.29.249:5443/v3

Config settings for Heat authentication

    [keystone_authtoken]
    insecure = True
    auth_plugin = password
    signing_dir = /var/lib/heat/cache/heat
    auth_url = https://130.238.29.249:35358
    auth_uri = https://130.238.29.249:5443
    project_domain_id = default
    user_domain_id = default
    project_name = service
    region_name = UPPMAX
    username = heat-HPC2N
    password = **SECRET**

Config settings for Heat to trust Keystone

    [trustee]
    insecure = True
    auth_plugin = password
    signing_dir = /var/lib/heat/cache/heat
    auth_url = https://130.238.29.249:35358
    auth_uri = https://130.238.29.249:5443
    project_domain_id = default
    user_domain_id = default
    project_name = service
    region_name = UPPMAX
    username = heat-HPC2N
    password = **SECRET**


### Keystone

Disable autostart of keystone containers

    perl -pi -e 's/lxc.start.auto = 1/lxc.start.auto = 0/' /var/lib/lxc/*_keystone_container*/config

Disable keystone containers in haproxy

    mkdir /etc/haproxy/disabled_conf.d
    mv /etc/haproxy/conf.d/keystone_* /etc/haproxy/disabled_conf.d
 
### Horizon

Disable autostart of horizon containers

    perl -pi -e 's/lxc.start.auto = 1/lxc.start.auto = 0/' /var/lib/lxc/*_horizon_container*/config

Disable horizon containers in haproxy
    
    mkdir /etc/haproxy/disabled_conf.d
    mv /etc/haproxy/conf.d/horizon_* /etc/haproxy/disabled_conf.d
