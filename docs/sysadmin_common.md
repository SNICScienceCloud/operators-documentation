# System administration

## Project quota management

Check a projects quota

    openstack quota show "SNIC 2017/13-7"

Check default quota

    openstack quota show --default

Changing a projects quota

    openstack quota set --instances=200 "SNIC 2017/13-7"

## Infrastructure OS Pathing

### Controller host

Disable services in haproxy

    root@controller_host: #

        echo "show stat" | nc -U /var/run/haproxy.stat | \
        grep `hostname -s` | cut -d, -f1,2 | tr , / | \
        xargs -i -n1 sh -c "echo disable server {} | nc -U /var/run/haproxy.stat"

Check that they are disabled

    root@controller_host: #

        echo "show stat" | nc -U /var/run/haproxy.stat | grep  `hostname -s`

Make sure that mysql is properly stopped

    root@controller_host: #

        lxc-attach -n `lxc-ls -1 |grep _galera_container` systemctl stop mysql

Upgrade packages in all containers

    root@controller_host: #

        lxc-ls -1 |xargs -i -n1 lxc-attach -n {} -- sh -c "apt-get update; apt-get -y dist-upgrade"

Update packages on the controller base os

    root@controller_host: #

        apt-get update
        apt-get -y dist-upgrade
        apt -y autoremove

Then reboot the controller node

    root@controller_host: #

        reboot

Make sure that the controller is up and in sync with galera before you continue with the next controller

    root@ansible_deploy_host: #

        cd /opt/openstack-ansible/playbooks
        ansible galera_container -s -m shell -a 'mysql -e "SHOW STATUS LIKE \"wsrep_cluster%\""'

In the output, **wsrep_cluster_size** should be **3** and the **wsrep_cluster** variables should be **the same in all** three databases.

    xxxxxx_galera_container | SUCCESS | rc=0 >>
    Variable_name   Value
    wsrep_cluster_conf_id   51
    wsrep_cluster_size          3
    wsrep_cluster_state_uuid        2b482fcd-ee11-11e6-9629-b7460b5b7501
    wsrep_cluster_status        Primary
    
    yyyyyy_galera_container | SUCCESS | rc=0 >>
    Variable_name   Value
    wsrep_cluster_conf_id   51
    wsrep_cluster_size          3
    wsrep_cluster_state_uuid        2b482fcd-ee11-11e6-9629-b7460b5b7501
    wsrep_cluster_status        Primary
    
    zzzzzz_galera_container | SUCCESS | rc=0 >>
    Variable_name   Value
    wsrep_cluster_conf_id   51
    wsrep_cluster_size          3
    wsrep_cluster_state_uuid        2b482fcd-ee11-11e6-9629-b7460b5b7501
    wsrep_cluster_status        Primary

### Compute hosts

Start live migrate on the utility container. Then set host in maintenance mode

    root@utility_container: # 
    
        compute_host={COMPUTE_HOST};
        nova service-disable --reason maintenance $compute_host nova-compute
        nova host-evacuate-live $compute_host; 
        watch sh -c "'nova migration-list |grep $compute_host |grep -v completed |grep `date +%F`'"

Watch status on the {COMPUTE_HOST} compute node

    root@compute_host: #

        watch virsh list

Wait until empty, then update pagackes of  {COMPUTE_HOST}

    root@compute_host: #

        apt-get update
        apt-get -y dist-upgrade
        apt -y autoremove

Make sure that no vm:s are running on {COMPUTE_HOST}

    root@compute_host: #

        virsh list

Then reboot  {COMPUTE_HOST}

    root@compute_host: #

        reboot

When the compute_host is rebooted, enable it again in the utility container

    root@utility_container: # 

        nova service-enable $compute_host nova-compute

### Ceph Storage hosts

Make sure that the health is OK

    root@ceph_storage_host: #

        ceph health

    HEALTH_OK

Set the noout flag on the OSD to prevent Ceph from start rebuilding OSDs while you reboot.

    root@ceph_storage_host: #

        ceph osd set noout

Patch the os and reboot

    root@ceph_storage_host: #

        apt-get update
        apt-get -y dist-upgrade
        apt -y autoremove
        reboot

Wait for the host to reboot and resync.. When this is complete the only warning should be that the noout flag is set.

    root@ceph_storage_host: #

        watch ceph health

    HEALTH_WARN noout flag(s) set


Then you can proceed with the next Ceph Storage host.

After all storage hosts are patched you can unset the OSD noout flag to resume normal operations.

    root@ceph_storage_host: #
        ceph osd unset noout


