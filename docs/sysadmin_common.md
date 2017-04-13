# System administration

## Project quota management

Check a projects quota

    openstack quota show "SNIC 2017/13-7"

Check default quota

    openstack quota show --default

Changing a projects quota

```openstack quota set --instances=200 "SNIC 2017/13-7"```

## Infrastructure OS Pathing

### Controller nodes

Disable services in haproxy

    echo "show stat" | nc -U /var/run/haproxy.stat | grep `hostname -s` | cut -d, -f1,2 | tr , / | xargs -i -n1 sh -c "echo disable server {} | nc -U /var/run/haproxy.stat"

Check that they are disabled

    echo "show stat" | nc -U /var/run/haproxy.stat | grep  `hostname -s`

Upgrade packages in all containers

    lxc-ls -1 |xargs -i -n1 lxc-attach -n {} -- sh -c "apt-get update; apt-get -y dist-upgrade"

Update packages on the bas os

    apt-get update; apt-get -y dist-upgrade; apt -y autoremove

Then reboot the controller node

    reboot

### Compute nodes

Start live migrate on the utility container, and set host in maintenance mode

    server={SERVER_NAME}; nova host-evacuate-live $server; watch sh -c "'nova migration-list |grep $server |grep -v completed |grep `date +%F`'"
    nova host-update --maintenance enable $server

Watch status on the {SERVER_NAME} compute node

    watch virsh list

Wait until empty, then update pagackes of  {SERVER_NAME}

    apt-get update; apt-get -y dist-upgrade; apt -y autoremove

Make sure that no vm:s are running on {SERVER_NAME}

    virsh list

Then reboot  {SERVER_NAME}

    reboot

When the server is rebooted, enable it again in the utility container

    nova host-update --maintenance disable $server
