# System administration

## Project quota management

Check a projects quota

    openstack quota show "SNIC 2017/13-7"

Check default quota

    openstack quota show --default

Changing a projects quota

```openstack quota set --instances=200 "SNIC 2017/13-7"```
