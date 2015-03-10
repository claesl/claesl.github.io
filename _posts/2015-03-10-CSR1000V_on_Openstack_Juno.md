---
layout: post
title: Cisco CSR1000V on Openstack Juno
---

## Installation notes

Using the following guide: 
<http://www.cisco.com/c/en/us/td/docs/routers/csr1000/software/configuration/csr1000Vswcfg/installkvm.html#28196>

Create a new flavor as admin

Settings for the flavor can be found on the following link

 <http://www.cisco.com/c/en/us/td/docs/routers/csr1000/software/configuration/csr1000Vswcfg/installkvm.html#45562>

```
root@controller:~# source /home/tdc/admin-openrc.sh
root@controller:~# nova flavor-create CSR1K 6 4096 0 4
+----+-------+-----------+------+-----------+------+-------+-------------+-----------+
| ID | Name  | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
+----+-------+-----------+------+-----------+------+-------+-------------+-----------+
| 6  | CSR1K | 4096      | 0    | 0         |      | 4     | 1.0         | True      |
+----+-------+-----------+------+-----------+------+-------+-------------+-----------+
```

Import CSR1000V image to Glance repository.

Here we import it as the demo tenant.

```
root@controller:~# source /home/tdc/demo-openrc.sh
root@controller:~# glance image-create --name CSR1KV-3-14 --disk-format qcow2 --container-format bare --file csr1000v-universalk9.03.14.01.S.155-1.S1-std.qcow2 --property hw_vif_model=virtio --property hw_disk_bus=virtio --property hw_cdrom_bus=ide
+-------------------------+--------------------------------------+
| Property                | Value                                |
+-------------------------+--------------------------------------+
| Property 'hw_cdrom_bus' | ide                                  |
| Property 'hw_disk_bus'  | virtio                               |
| Property 'hw_vif_model' | virtio                               |
| checksum                | 053c559fd2fda865b5df5b0d3ff2f82a     |
| container_format        | bare                                 |
| created_at              | 2015-03-10T11:57:36                  |
| deleted                 | False                                |
| deleted_at              | None                                 |
| disk_format             | qcow2                                |
| id                      | 17bf0b8e-11cf-43d2-b1dc-9b72d7aa399e |
| is_public               | False                                |
| min_disk                | 0                                    |
| min_ram                 | 0                                    |
| name                    | CSR1KV-3-14                          |
| owner                   | be6dd690a9de4ca0adc065f2c86aea65     |
| protected               | False                                |
| size                    | 1212743680                           |
| status                  | active                               |
| updated_at              | 2015-03-10T11:57:42                  |
| virtual_size            | None                                 |
+-------------------------+--------------------------------------+
```

Create three networks

We will disable DHCP and Gateway on all the network and let the CSR1000V do those functions later.

Management network - 12.1.1.0/24

```
root@controller:~# source /home/tdc/demo-openrc.sh
root@controller:~# neutron net-create demo-mgmt
Created a new network:
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| admin_state_up  | True                                 |
| id              | 9ae4b71d-cdae-40e0-8bf0-eebffdbe42c2 |
| name            | demo-mgmt                            |
| router:external | False                                |
| shared          | False                                |
| status          | ACTIVE                               |
| subnets         |                                      |
| tenant_id       | be6dd690a9de4ca0adc065f2c86aea65     |
+-----------------+--------------------------------------+
root@controller:~# neutron subnet-create --name mgmt-12.1.1 --no-gateway --disable-dhcp  demo-mgmt 12.1.1.0/24
Created a new subnet:
+-------------------+--------------------------------------------+
| Field             | Value                                      |
+-------------------+--------------------------------------------+
| allocation_pools  | {"start": "12.1.1.1", "end": "12.1.1.254"} |
| cidr              | 12.1.1.0/24                                |
| dns_nameservers   |                                            |
| enable_dhcp       | False                                      |
| gateway_ip        |                                            |
| host_routes       |                                            |
| id                | 2067615c-1ecd-457c-a82f-24944727a952       |
| ip_version        | 4                                          |
| ipv6_address_mode |                                            |
| ipv6_ra_mode      |                                            |
| name              | mgmt-12.1.1                                |
| network_id        | 9ae4b71d-cdae-40e0-8bf0-eebffdbe42c2       |
| tenant_id         | be6dd690a9de4ca0adc065f2c86aea65           |
+-------------------+--------------------------------------------+
```

Internal network 12.1.2.0/24

```
root@controller:~# neutron net-create demo-internal
Created a new network:
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| admin_state_up  | True                                 |
| id              | dfad0409-7b05-468b-bdd1-d044a04def39 |
| name            | demo-internal                        |
| router:external | False                                |
| shared          | False                                |
| status          | ACTIVE                               |
| subnets         |                                      |
| tenant_id       | be6dd690a9de4ca0adc065f2c86aea65     |
+-----------------+--------------------------------------+
root@controller:~# neutron subnet-create --name internal-12.1.2 --no-gateway --disable-dhcp  demo-internal 12.1.2.0/24
Created a new subnet:
+-------------------+--------------------------------------------+
| Field             | Value                                      |
+-------------------+--------------------------------------------+
| allocation_pools  | {"start": "12.1.2.1", "end": "12.1.2.254"} |
| cidr              | 12.1.2.0/24                                |
| dns_nameservers   |                                            |
| enable_dhcp       | False                                      |
| gateway_ip        |                                            |
| host_routes       |                                            |
| id                | e5046828-8e73-437f-9422-78e9229c2089       |
| ip_version        | 4                                          |
| ipv6_address_mode |                                            |
| ipv6_ra_mode      |                                            |
| name              | internal-12.1.2                            |
| network_id        | dfad0409-7b05-468b-bdd1-d044a04def39       |
| tenant_id         | be6dd690a9de4ca0adc065f2c86aea65           |
+-------------------+--------------------------------------------+
```

External network 12.1.3.0/24

```
root@controller:~# neutron net-create demo-external
Created a new network:
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| admin_state_up  | True                                 |
| id              | 66168375-c7c5-4367-90a9-f92eb1d93cab |
| name            | demo-external                        |
| router:external | False                                |
| shared          | False                                |
| status          | ACTIVE                               |
| subnets         |                                      |
| tenant_id       | be6dd690a9de4ca0adc065f2c86aea65     |
+-----------------+--------------------------------------+
root@controller:~# neutron subnet-create --name external-12.1.3 demo-external 12.1.3.0/24
Created a new subnet:
+-------------------+--------------------------------------------+
| Field             | Value                                      |
+-------------------+--------------------------------------------+
| allocation_pools  | {"start": "12.1.3.2", "end": "12.1.3.254"} |
| cidr              | 12.1.3.0/24                                |
| dns_nameservers   |                                            |
| enable_dhcp       | True                                       |
| gateway_ip        | 12.1.3.1                                   |
| host_routes       |                                            |
| id                | f578f214-95c0-420f-95ed-c86885dc5e83       |
| ip_version        | 4                                          |
| ipv6_address_mode |                                            |
| ipv6_ra_mode      |                                            |
| name              | external-12.1.3                            |
| network_id        | 66168375-c7c5-4367-90a9-f92eb1d93cab       |
| tenant_id         | be6dd690a9de4ca0adc065f2c86aea65           |
+-------------------+--------------------------------------------+
```

Now we will create some ports for the CSR1000V on each of the networks.

```
root@controller:~# neutron port-create demo-mgmt --name "csr1kv01-mgmt" --fixed-ip ip_address=12.1.1.1
Created a new port:
+-----------------------+---------------------------------------------------------------------------------+
| Field                 | Value                                                                           |
+-----------------------+---------------------------------------------------------------------------------+
| admin_state_up        | True                                                                            |
| allowed_address_pairs |                                                                                 |
| binding:vnic_type     | normal                                                                          |
| device_id             |                                                                                 |
| device_owner          |                                                                                 |
| fixed_ips             | {"subnet_id": "2067615c-1ecd-457c-a82f-24944727a952", "ip_address": "12.1.1.1"} |
| id                    | b030f083-38db-4a66-a344-ca29f9eb495b                                            |
| mac_address           | fa:16:3e:dd:97:ac                                                               |
| name                  | csr1kv01-mgmt                                                                   |
| network_id            | 9ae4b71d-cdae-40e0-8bf0-eebffdbe42c2                                            |
| security_groups       | c05ece0b-508b-4fb3-bca2-217fcb0a85b6                                            |
| status                | DOWN                                                                            |
| tenant_id             | be6dd690a9de4ca0adc065f2c86aea65                                                |
+-----------------------+---------------------------------------------------------------------------------+
root@controller:~# neutron port-create demo-internal --name "csr1kv01-internal" --fixed-ip ip_address=12.1.2.1
Created a new port:
+-----------------------+---------------------------------------------------------------------------------+
| Field                 | Value                                                                           |
+-----------------------+---------------------------------------------------------------------------------+
| admin_state_up        | True                                                                            |
| allowed_address_pairs |                                                                                 |
| binding:vnic_type     | normal                                                                          |
| device_id             |                                                                                 |
| device_owner          |                                                                                 |
| fixed_ips             | {"subnet_id": "e5046828-8e73-437f-9422-78e9229c2089", "ip_address": "12.1.2.1"} |
| id                    | ff46af2c-296d-46b5-86ab-4676be3c81bc                                            |
| mac_address           | fa:16:3e:1b:ff:2a                                                               |
| name                  | csr1kv01-internal                                                               |
| network_id            | dfad0409-7b05-468b-bdd1-d044a04def39                                            |
| security_groups       | c05ece0b-508b-4fb3-bca2-217fcb0a85b6                                            |
| status                | DOWN                                                                            |
| tenant_id             | be6dd690a9de4ca0adc065f2c86aea65                                                |
+-----------------------+---------------------------------------------------------------------------------+
root@controller:~# neutron port-create demo-external --name "csr1kv01-external"
Created a new port:
+-----------------------+---------------------------------------------------------------------------------+
| Field                 | Value                                                                           |
+-----------------------+---------------------------------------------------------------------------------+
| admin_state_up        | True                                                                            |
| allowed_address_pairs |                                                                                 |
| binding:vnic_type     | normal                                                                          |
| device_id             |                                                                                 |
| device_owner          |                                                                                 |
| fixed_ips             | {"subnet_id": "f578f214-95c0-420f-95ed-c86885dc5e83", "ip_address": "12.1.3.2"} |
| id                    | 6c78d056-5165-47d8-9db5-f70e2a3ccf40                                            |
| mac_address           | fa:16:3e:b7:df:26                                                               |
| name                  | csr1kv01-external                                                               |
| network_id            | 66168375-c7c5-4367-90a9-f92eb1d93cab                                            |
| security_groups       | c05ece0b-508b-4fb3-bca2-217fcb0a85b6                                            |
| status                | DOWN                                                                            |
| tenant_id             | be6dd690a9de4ca0adc065f2c86aea65                                                |
+-----------------------+---------------------------------------------------------------------------------+
```

Virtual machines in Openstack can't connect directly to the external network. Due to this restriction the CSR1000V will have to connect to the demo-external network. On that network the router-1 will be default gateway doing the NAT and routing so the CSR1000V can reach Internet.

Let's create router-1 and connect it to demo-external and ext-net (our public network) network.

```
root@controller:~# neutron router-create router-1
Created a new router:
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| external_gateway_info |                                      |
| id                    | 91366764-e446-4944-800c-6dd71e19d53f |
| name                  | router-1                             |
| routes                |                                      |
| status                | ACTIVE                               |
| tenant_id             | be6dd690a9de4ca0adc065f2c86aea65     |
+-----------------------+--------------------------------------+
root@controller:~# neutron router-interface-add router-1 subnet=external-12.1.3
Added interface ef9105d4-8ce5-472f-847e-8a921e59448e to router router-1.
root@controller:~# neutron router-gateway-set router-1 ext-net
Set gateway for router router-1
```

And last we create the CSR1000V instance

```
root@controller:~# nova boot --image CSR1KV-3-14 --flavor CSR1K --nic port-id=b030f083-38db-4a66-a344-ca29f9eb495b --nic port-id=ff46af2c-296d-46b5-86ab-4676be3c81bc --nic port-id=6c78d056-5165-47d8-9db5-f70e2a3ccf40 csr1000v-01
+--------------------------------------+----------------------------------------------------+
| Property                             | Value                                              |
+--------------------------------------+----------------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                             |
| OS-EXT-AZ:availability_zone          | nova                                               |
| OS-EXT-STS:power_state               | 0                                                  |
| OS-EXT-STS:task_state                | scheduling                                         |
| OS-EXT-STS:vm_state                  | building                                           |
| OS-SRV-USG:launched_at               | -                                                  |
| OS-SRV-USG:terminated_at             | -                                                  |
| accessIPv4                           |                                                    |
| accessIPv6                           |                                                    |
| adminPass                            | Di2FKj3vah8E                                       |
| config_drive                         |                                                    |
| created                              | 2015-03-10T13:51:24Z                               |
| flavor                               | CSR1K (6)                                          |
| hostId                               |                                                    |
| id                                   | 02611e24-072a-4597-819d-ace27fdb54b6               |
| image                                | CSR1KV-3-14 (17bf0b8e-11cf-43d2-b1dc-9b72d7aa399e) |
| key_name                             | -                                                  |
| metadata                             | {}                                                 |
| name                                 | csr1000v-01                                        |
| os-extended-volumes:volumes_attached | []                                                 |
| progress                             | 0                                                  |
| security_groups                      | default                                            |
| status                               | BUILD                                              |
| tenant_id                            | be6dd690a9de4ca0adc065f2c86aea65                   |
| updated                              | 2015-03-10T13:51:24Z                               |
| user_id                              | ce1394e657534ea7973f22071db00047                   |
+--------------------------------------+----------------------------------------------------+
```

Verifying it started.

```
root@controller:~# nova list
+--------------------------------------+-------------+--------+------------+-------------+--------------------------------------------------------------------+
| ID                                   | Name        | Status | Task State | Power State | Networks                                                           |
+--------------------------------------+-------------+--------+------------+-------------+--------------------------------------------------------------------+
| e4902047-b5ef-4a40-82b7-2205852c8725 | csr1000v-01 | ACTIVE | -          | Running     | demo-mgmt=12.1.1.1; demo-external=12.1.3.1; demo-internal=12.1.2.1 |
+--------------------------------------+-------------+--------+------------+-------------+--------------------------------------------------------------------+
```

Lets create a host on the internal network.

```
root@controller:~# nova boot --image cirros-0.3.3-x86_64  --flavor m1.tiny  --nic net-id=dfad0409-7b05-468b-bdd1-d044a04def39 internal-vm01
+--------------------------------------+------------------------------------------------------------+
| Property                             | Value                                                      |
+--------------------------------------+------------------------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                                     |
| OS-EXT-AZ:availability_zone          | nova                                                       |
| OS-EXT-STS:power_state               | 0                                                          |
| OS-EXT-STS:task_state                | scheduling                                                 |
| OS-EXT-STS:vm_state                  | building                                                   |
| OS-SRV-USG:launched_at               | -                                                          |
| OS-SRV-USG:terminated_at             | -                                                          |
| accessIPv4                           |                                                            |
| accessIPv6                           |                                                            |
| adminPass                            | UNBD9R3qw8WG                                               |
| config_drive                         |                                                            |
| created                              | 2015-03-10T13:36:17Z                                       |
| flavor                               | m1.tiny (1)                                                |
| hostId                               |                                                            |
| id                                   | 2951ef6c-ee1a-4def-becb-dc3b7f7e567a                       |
| image                                | cirros-0.3.3-x86_64 (5354b66f-687e-43e9-b539-74a5f8a56626) |
| key_name                             | -                                                          |
| metadata                             | {}                                                         |
| name                                 | internal-vm01                                              |
| os-extended-volumes:volumes_attached | []                                                         |
| progress                             | 0                                                          |
| security_groups                      | default                                                    |
| status                               | BUILD                                                      |
| tenant_id                            | be6dd690a9de4ca0adc065f2c86aea65                           |
| updated                              | 2015-03-10T13:36:17Z                                       |
| user_id                              | ce1394e657534ea7973f22071db00047                           |
+--------------------------------------+------------------------------------------------------------+
```
### Tested versions
Openstack Juno
CSR1000V: csr1000v-universalk9.03.14.01.S.155-1.S1-std.qcow2
