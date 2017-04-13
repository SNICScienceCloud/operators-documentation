# System administration

## Project quota management

Check a projects quota

    openstack quota show "SNIC 2017/13-7"

Check default quota

    openstack quota show --default

Changing a projects quota

```openstack quota set --instances=200 "SNIC 2017/13-7"```

## Infrastructure OS Pathing

### Controller host

Disable services in haproxy

    root@controller_host: #
    	echo "show stat" | nc -U /var/run/haproxy.stat | \
        grep `hostname -s` | cut -d, -f1,2 | tr , / | \
        xargs -i -n1 sh -c "echo disable compute_host {} | nc -U /var/run/haproxy.stat"

Check that they are disabled

    root@controller_host: #
        echo "show stat" | nc -U /var/run/haproxy.stat | grep  `hostname -s`

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

### Compute hosts

Start live migrate on the utility container, and set host in maintenance mode

    root@utility_container: # 
        compute_host={COMPUTE_HOST};
        nova host-evacuate-live $compute_host; 
        watch sh -c "'nova migration-list |grep $compute_host |grep -v completed |grep `date +%F`'"
        nova host-update --maintenance enable $compute_host

Watch status on the {COMPUTE_HOST} compute node

    root@compute_host: #
    	watch virsh list

Wait until empty, then update pagackes of  {COMPUTE_HOST}

    root@compute_host: #
        apt-get update; apt-get -y dist-upgrade; apt -y autoremove

Make sure that no vm:s are running on {COMPUTE_HOST}

    root@compute_host: #
        virsh list

Then reboot  {COMPUTE_HOST}

    root@compute_host: #
        reboot

When the compute_host is rebooted, enable it again in the utility container

    root@utility_container: # 
        nova host-update --maintenance disable $compute_host
