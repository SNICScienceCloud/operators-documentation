# Openstack Liberty Deployment at HPC2N

## General

Openstack Ansible was used for the Openstack deployment at HPC2N.

## System map

![System map](img/ssc_hpc2n.png)

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

## Network

Linux-bridges are used for the for the vlan and vxlan traffic.

### Floating IPv4 Pool

The floating IPv4 network pool is a full class C network (130.239.81.0/24) with the gateway 130.239.81.254.

Created with:

    neutron net-create floating-130.239.81 --shared --router:external=True --provider:network_type vlan --provider:segmentation_id 3 --provider:physical_network vlan

    neutron subnet-create  --allocation-pool start=130.239.81.1,end=130.239.81.253 --gateway 130.239.81.254 --disable-dhcp --name floating-130.239.81 --ip-version 4 --dns-nameserver 130.239.1.90 *EXTERNAL_NETWORK_CIDR* 130.239.81.0/24

## Manual config changes required for region interation

**These are the settings for the test-cloud without TLS on the endpoints**

### Nova

#### Modifications to `/etc/nova/nova.conf` on `compute_hosts`

Config settings for Nova to authenticate with Neutron credentials

    [neutron]
    auth_plugin = password
    auth_url = http://130.238.29.249:35357
    auth_uri = http://130.238.29.249:5000
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = neutron-HPC2N
    password = **SECRET**
    region_name = UPPMAX

Config settings for Nova to authenticate with the correct Cinder instance.

Locate the correct cinder endpoint with `openstack endpoint list` and the catalog_info format is "Service Type:Service Name:Interface URL Type".

    [cinder]
    catalog_info = volumev2:cinderv2-HPC2N:internalURL
    os_region_name = HPC2N

Config settings for Nova authentication

    [keystone_authtoken]
    auth_plugin = password
    auth_url = http://130.238.29.249:35357
    auth_uri = http://130.238.29.249:5000
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = nova-HPC2N
    password = **SECRET**
    region_name = UPPMAX

#### Modifications to `/etc/nova/nova.conf` on `nova_api_os_compute_container`, `nova_api_metadata_container`, `nova_console_container`, `nova_cert_container` and `nova_conductor_container`

Config settings for Nova to authenticate with the correct Cinder instance.

Locate the correct cinder endpoint with `openstack endpoint list` and the catalog_info format is "Service Type:Service Name:Interface URL Type".

    [cinder]
    catalog_info = volumev2:cinderv2-HPC2N:internalURL
    os_region_name = HPC2N

Config settings for Nova to authenticate with Neutron credentials

    [neutron]
    auth_plugin = password
    auth_url = http://130.238.29.249:35357
    auth_uri = http://130.238.29.249:5000
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = neutron-HPC2N
    password = **SECRET**
    region_name = UPPMAX

Config settings for Nova authentication

    [keystone_authtoken]
    insecure = False
    auth_plugin = password
    auth_url = http://130.238.29.249:35357
    auth_uri = http://130.238.29.249:5000
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = nova-HPC2N
    password = **SECRET**
    region_name = UPPMAX

### Ceilometer

#### Modifications to `/etc/ceilometer/ceilometer.conf` on `ceilometer_api_container`

Config settings for Ceilometer authentication

    [keystone_authtoken]
    insecure = False
    auth_plugin = password
    signing_dir = /var/lib/glance/cache/api
    identity_uri = http://130.238.29.249:35357
    auth_url = http://130.238.29.249:35357
    auth_uri = http://130.238.29.249:5000
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = ceilometer-HPC2N
    password = **SECRET**
    region_name = UPPMAX

Config settings for Ceilometer service

    [service_credentials]
    os_auth_url = http://130.238.29.249:35357
    os_auth_uri = http://130.238.29.249:5000
    os_username = ceilometer-HPC2N
    os_tenant_name = service
    os_password = **SECRET**
    os_region_name = HPC2N

#### Modifications to `/etc/ceilometer/ceilometer.conf` on `ceilometer_collector_container`

Config settings for Ceilometer authentication

    [keystone_authtoken]
    insecure = False
    auth_plugin = password
    signing_dir = /var/lib/glance/cache/api
    identity_uri = http://130.238.29.249:35357
    auth_url = http://130.238.29.249:35357
    auth_uri = http://130.238.29.249:5000
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = ceilometer-HPC2N
    password = **SECRET**
    region_name = UPPMAX

### Cinder

#### Modifications to `/etc/cinder/cinder.conf` on `cinder_api_container`, `cinder_scheduler_container` and `cinder_volumes_container`

Config settings for Cinder authentication

    [keystone_authtoken]
    insecure = False
    auth_plugin = password
    signing_dir = /var/cache/cinder
    auth_url = http://130.238.29.249:35357
    auth_uri = http://130.238.29.249:5000
    project_domain_id = default
    user_domain_id = default
    project_name = service
    region_name = UPPMAX
    username = cinder-HPC2N
    password = **SECRET**


### Neutron 

#### Modifications to `/etc/neutron/metadata_agent.ini` on `neutron_agents_container`

Config settings for Neutron authentication

    [default]
    auth_plugin = password
    auth_url = http://130.238.29.249:35357
    auth_uri = http://130.238.29.249:5000
    auth_region = UPPMAX
    admin_tenant_name = service
    admin_user = neutron-HPC2N
    admin_password = **SECRET**
    endpoint_type = publicURL

#### Modifications to `/etc/neutron/neutron.conf` on `neutron_server_container`

Config settings for Neutron to authenticate with Nova credentials

    [nova]
    project_name=services
    username=nova-HPC2N
    password=**SECRET**
    project_domain_id=default
    tenant_name=services
    user_domain_id=default
    auth_url=http://130.238.29.249:35357/
    region_name=UPPMAX

Config settings for Neutron authentication

    [keystone_authtoken]
    insecure = False
    auth_plugin = password
    auth_url = http://130.238.29.249:35357
    auth_uri = http://130.238.29.249:5000
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = neutron-HPC2N
    password = **SECRET**
    region_name = UPPMAX


### Glance

#### Modifications to `/etc/glance/glance-api.conf` and `/etc/glance/glance-registry.conf` on `glance_container`

Config settings for Glance authentication

    [keystone_authtoken]
    insecure = False
    auth_plugin = password
    signing_dir = /var/lib/glance/cache/api
    auth_url = http://130.238.29.249:35357
    auth_uri = http://130.238.29.249:5000
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = glance-HPC2N
    password = **SECRET**
    region_name = UPPMAX

#### Modifications to `/etc/glance/glance-cache.conf` on `glance_container`

    auth_url = http://130.238.29.249:35357
    auth_uri = http://130.238.29.249:5000
    admin_user = glance-HPC2N
    admin_password = **SECRET**

### Heat

#### Modifications to `/etc/heat/heat.conf` on `heat_apis_container` and `heat_engine_container`

Config settings for Heat authentication

    [keystone_authtoken]
    insecure = False
    auth_plugin = password
    signing_dir = /var/lib/heat/cache/heat
    auth_url = http://130.238.29.249:35357
    auth_uri = http://130.238.29.249:5000
    project_domain_id = default
    user_domain_id = default
    project_name = service
    region_name = UPPMAX
    username = heat-HPC2N
    password = **SECRET**

Config settings for Heat to trust Keystone

    [trustee]
    insecure = False
    auth_plugin = password
    signing_dir = /var/lib/heat/cache/heat
    auth_url = http://130.238.29.249:35357
    auth_uri = http://130.238.29.249:5000
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
