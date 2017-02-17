# Openstack Newton Deployment at HPC2N

## General

Openstack Ansible was used for the Openstack deployment at HPC2N.

    git clone https://git.openstack.org/openstack/openstack-ansible
    git checkout stable/newton

Some region and federation bugs were reported to openstack-ansible during the install.
Make sure that you have the fixes for bug#1660322, bug#1660344, bug#1660626 and bug#1661197 before starting the deply.

If your openstack-ansible does not include fixes for them then you can download them from here

- [Fix for bug #1660322 and #1660344](bugfix/bug1660322_and_bug1660344_fix.tgz)
- [Fix for bug #1660626](bugfix/bug1660626_fix.tgz)
- [Fix for bug #1661197](bugfix/bug1661197_fix.tgz)

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

All nodes run Ubuntu 16.04 and they are installed with Puppet, the HPC2N standard for server installation.

### Compute node post puppet suff

All stuff is not yet fixed with puppet so after the initial installation of the host has completed we need to run some commands and then configure the networks.

    mkdir /openstack
    mkdir /var/lib/nova
    
    mkfs.xfs /dev/rootvg/openstacklv; then
    mkfs.xfs /dev/rootvg/novalv
    
    cat - >> /etc/fstab <<EOF
    /dev/rootvg/openstacklv	/openstack	xfs	nodev	0 2
    /dev/rootvg/novalv	/var/lib/nova	xfs	nosuid,nodev	0 2
    EOF
    
    mount -a
    
    cat - > /etc/apt/sources.list <<EOF
    ###### Ubuntu Main Repos
    deb http://archive.ubuntu.com/ubuntu/ xenial main restricted universe multiverse
    ###### Ubunt:u Update Repos
    deb http://archive.ubuntu.com/ubuntu/ xenial-security main restricted universe multiverse
    deb http://archive.ubuntu.com/ubuntu/ xenial-updates main restricted universe multiverse
    EOF
    
    
    echo 'bonding' >> /etc/modules
    echo '8021q' >> /etc/modules
    
    apt-get install -y bridge-utils debootstrap ifenslave ifenslave-2.6 lsof lvm2 ntp openssh-server sudo tcpdump vlan
    
    perl -pi -e 's,(.*?/varlv.*?\s+)(nosuid\,nodev),$1 defaults,' /etc/fstab
    perl -pi -e 's,/home/syslog,/var/spool/rsyslog,' /etc/passwd

    mount -o remount /var

### Management node post puppet suff

All stuff is not yet fixed with puppet so after the initial installation of the host has completed we need to run some commands and then configure the networks.

    mkdir /openstack
    
    lvcreate -n openstacklv -L20G lxc
    
    mkfs.xfs /dev/lxc/openstacklv
    cat - >> /etc/fstab <<EOF
    /dev/lxc/openstacklv	/openstack	xfs	nodev	0 2
    EOF
    
    mount -a
    
    cat - > /etc/apt/sources.list <<EOF
    ###### Ubuntu Main Repos
    deb http://archive.ubuntu.com/ubuntu/ xenial main restricted universe multiverse
    ###### Ubunt:u Update Repos
    deb http://archive.ubuntu.com/ubuntu/ xenial-security main restricted universe multiverse
    deb http://archive.ubuntu.com/ubuntu/ xenial-updates main restricted universe multiverse
    EOF
    
    
    echo 'bonding' >> /etc/modules
    echo '8021q' >> /etc/modules
    
    apt-get install -y bridge-utils debootstrap ifenslave ifenslave-2.6 lsof lvm2 ntp openssh-server sudo tcpdump vlan
    
    perl -pi -e 's,(.*?/varlv.*?\s+)(nosuid\,nodev),$1 defaults,' /etc/fstab
    perl -pi -e 's,/home/syslog,/var/spool/rsyslog,' /etc/passwd

    mount -o remount /var

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

    auto eno1
    iface eno1 inet manual
    
    # Container management VLAN interface
    auto eno1.10
    iface eno1.10 inet manual
        vlan-raw-device eno1
    
    auto eno2
    iface eno2 inet manual
    
    # OpenStack Networking VXLAN (tunnel/overlay) VLAN interface
    auto eno2.20
    iface eno2.20 inet manual
        vlan-raw-device eno2
    
    # Container management bridge
    auto br-mgmt
    iface br-mgmt inet static
        bridge_stp off
        bridge_waitport 0
        bridge_fd 0
        # Bridge port references tagged interface
        bridge_ports eno1.10
        address 172.16.2.3X
        netmask 255.255.255.0
    
    # OpenStack Networking VXLAN (tunnel/overlay) bridge
    auto br-vxlan
    iface br-vxlan inet static
        bridge_stp off
        bridge_waitport 0
        bridge_fd 0
        # Bridge port references tagged interface
        bridge_ports eno2.20
        address 10.10.0.3X
        netmask 255.255.0.0
    
    # OpenStack Networking VLAN bridge
    auto br-vlan
    iface br-vlan inet manual
        bridge_stp off
        bridge_waitport 0
        bridge_fd 0
        # Bridge port references untagged interface
        bridge_ports eno2
    
    auto br-storage
    iface br-storage inet static
        # Completely disable IPv6 router advertisements/autoconfig
        pre-up sysctl -q -e -w net/ipv6/conf/br-storage/accept_ra=0
        bridge_stp off
        bridge_waitport 0
        bridge_fd 0
        bridge_ports eno1
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

    auto eno1
    iface eno1 inet manual
        bond-master bond0
    
    auto enp3s0f0
    iface enp3s0f0 inet manual
        bond-master bond0
    
    auto bond0
    iface bond0 inet manual
        bond-slaves eno1 enp3s0f0
        bond-mode active-backup
        bond-miimon 100
        bond-downdelay 200
        bond-updelay 200

/etc/network/interfaces.d/bond1.interfaces

    auto eno2
    iface eno2 inet manual
        bond-master bond1
    
    auto enp3s0f1
    iface enp3s0f1 inet manual
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

    neutron net-create public --shared --router:external=True --provider:network_type vlan --provider:segmentation_id 3 --provider:physical_network vlan

    neutron subnet-create  --allocation-pool start=130.239.81.1,end=130.239.81.253 --gateway 130.239.81.254 --disable-dhcp --name public-81 --ip-version 4 --dns-nameserver 130.239.1.90 *ID_OF_PUBLIC_NETWORK* 130.239.81.0/24

## Galera

### Galera backup

Create a backup script in all galera containers


    cat - <<EOF > /etc/cron.hourly/mysqldump
    #!/bin/sh
    /usr/bin/mysqldump --single-transaction --all-databases | /bin/gzip -c > /var/backup/sqldump.`date +"%H"`.sql.gz
    EOF

    chmod +x /etc/cron.hourly/mysqldump


## Ceilometer

### Setup mogoDB

MongoDB is not setup by the openstack ansible scripts

The mongoDB dabase is installed on the controller node u-mn-24 using the following commands

    apt-get install mongodb-server mongodb-clients python-pymongo
    sed -i 's/127.0.0.1/172.16.2.2/g' /etc/mongodb.conf
    echo smallfiles = true >> /etc/mongodb.conf
    service mongodb restart

Get **CEILOMETER_DBPASS** from `ceilometer_container_db_password` in `/etc/openstack_deploy/user_secrets.yml` on deploy host `u-sn-o38`

    mongo --host 172.16.2.2 --eval '
    db = db.getSiblingDB("ceilometer");
    db.addUser({user: "ceilometer",
    pwd: "**CEILOMETER_DBPASS**",
    roles: [ "readWrite", "dbAdmin" ]})'

### Database cleanup

Configure how long the samples and events should be stored in the database.

We will only keep them for 7 days.

On the deploy host edit /etc/ansible/roles/os_ceilometer/templates/ceilometer.conf.j2. Add metering_time_to_live and event_time_to_live

    --- ceilometer.conf.j2.old	2017-02-14 11:25:45.470152855 +0100
    +++ ceilometer.conf.j2.new	2017-02-14 11:25:54.481702171 +0100
    @@ -106,6 +106,8 @@
     [database]
     metering_connection = {{ ceilometer_connection_string }}
     event_connection = {{ ceilometer_connection_string }}
    +metering_time_to_live = {{ ceilometer_metering_time_to_live }}
    +event_time_to_live = {{ ceilometer_event_time_to_live }}
     
     {% if ceilometer_gnocchi_enabled | bool %}
      [dispatcher_gnocchi]

Edit your user_variables.yml and add the variables

    ceilometer_metering_time_to_live: 604800
    ceilometer_event_time_to_live: 604800

## Horizon

### "Drop down" patch for regions

To get a drop down menu for all regions that works with saml2 we need to replace a file in all horizon containsers.

Run the following commands in all horizon containers.

    cp /openstack/venvs/horizon-14.0.7/lib/python2.7/site-packages/horizon/templates/horizon/common/_region_selector.html /openstack/venvs/horizon-14.0.7/lib/python2.7/site-packages/horizon/templates/horizon/common/_region_selector.html.orig
    
    cat - <<EOF > /openstack/venvs/horizon-14.0.7/lib/python2.7/site-packages/horizon/templates/horizon/common/_region_selector.html
      <a href="#" class="dropdown-toggle" data-toggle="dropdown" role="button" aria-expanded="false">
        <span class="region-title">HPC2N</span>
        <span class="fa fa-caret-down"></span>
      </a>
      <ul id="region_list" class="dropdown-menu dropdown-menu-right">
           <li>
             <a href="https://c3se.cloud.snic.se/project">
               C3SE
             </a>
           </li>
           <li>
             <a href="https://uppmax.cloud.snic.se/project">
               UPPMAX
             </a>
           </li>
      </ul>
    EOF

### Fix for broken POLICY_CHECK_FUNCTION check

The version of horizon checked out by openstack-ansible has a bug causing the Admin panel to show for non admin users.

This bug is solved in later versions of horizon but you can do a quickfix by editing /openstack/venvs/horizon-14.0.7/lib/python2.7/site-packages/openstack_dashboard/dashboards/admin/dashboard.py

Replace

    class Admin(horizon.Dashboard):
        name = _("Admin")
        slug = "admin"

        if getattr(settings, 'POLICY_CHECK_FUNCTION', None):
            policy_rules = (('identity', 'admin_required'),
                            ('image', 'context_is_admin'),
                            ('volume', 'context_is_admin'),
                            ('compute', 'context_is_admin'),
                            ('network', 'context_is_admin'),
                            ('orchestration', 'context_is_admin'),)
        else:
            permissions = (tuple(utils.get_admin_permissions()),)

With

    class Admin(horizon.Dashboard):
        name = _("Admin")
        slug = "admin"

        permissions = (tuple(utils.get_admin_permissions()),)

## Glance

### Flavors

Flavors are sorted by the ID, so set them in the correct order.

   openstack flavor create --ram 512 --disk 1 --vcpus 1 --id 8c704ef9-74dc-495e-9e2b-baebc6775b16 --public ssc.tiny
   openstack flavor create --ram 2048 --disk 20 --vcpus 1 --id 8d704ef9-74dc-495e-9e2b-baebc6775b16 --public ssc.small
   openstack flavor create --ram 4096 --disk 40 --vcpus 2 --id 8e704ef9-74dc-495e-9e2b-baebc6775b16 --public ssc.medium
   openstack flavor create --ram 8192 --disk 80 --vcpus 4 --id 8f704ef9-74dc-495e-9e2b-baebc6775b16 --public ssc.large
   openstack flavor create --ram 16384 --disk 160 --vcpus 8 --id 8g704ef9-74dc-495e-9e2b-baebc6775b16 --public ssc.xlarge

### Images

These are the ones used at installation time, make sure to use the latest available and update description.
Do not just copy the block because it will fail to fecth some of the images.

    wget https://cloud-images.ubuntu.com/xenial/20170128/xenial-server-cloudimg-amd64-disk1.img
    glance image-create --progress --file xenial-server-cloudimg-amd64-disk1.img --visibility public --name "Ubuntu 16.04 LTS (Xenial Xerus) Daily Build [20170128]" --disk-format qcow2 --container-format bare --min-disk 3
    rm xenial-server-cloudimg-amd64-disk1.img
    
    wget http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-1611.qcow2
    glance image-create --progress --file CentOS-7-x86_64-GenericCloud-1611.qcow2 --visibility public --name "Centos 7 [20170117]" --disk-format qcow2 --container-format bare --min-disk 8
    rm CentOS-7-x86_64-GenericCloud-1611.qcow2
    
    wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
    glance image-create --progress --file cirros-0.3.4-x86_64-disk.img --visibility public --name "Cirros 0.3.4 [20150507]" --disk-format qcow2 --container-format bare --min-disk 1
    rm cirros-0.3.4-x86_64-disk.img
    
    wget https://cloud-images.ubuntu.com/trusty/20170202/trusty-server-cloudimg-amd64-disk1.img
    glance image-create --progress --file trusty-server-cloudimg-amd64-disk1.img --visibility public --name "Ubuntu 14.04 LTS (Trusty Tahr) Daily Build [20170202]" --disk-format qcow2 --container-format bare --min-disk 3
    rm trusty-server-cloudimg-amd64-disk1.img
    
    wget https://stable.release.core-os.net/amd64-usr/1235.9.0/coreos_production_openstack_image.img.bz2
    bzip2 -d coreos_production_openstack_image.img.bz2
    glance image-create --progress --file coreos_production_openstack_image.img --visibility public --name "CoreOS 1235.9.0 [20170202]" --disk-format qcow2 --container-format bare --min-disk 9
    rm coreos_production_openstack_image.img


