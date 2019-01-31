# ceph-admin-notes

These notes were taken as an exercise to study Red Hat Ceph Storage 3.x deployment and administration, are not intended to replace or substitute the content present in the official Ceph [documentation](http://docs.ceph.com/).

 ~ [mrbarge](https://github.com/mrbarge/)

## Installation and Deployment

### Satellite Repositories
- `rhel-7-server-rpms`
- `rhel-7-server-extras-rpms`
- `rhel-7-server-rhceph-3-mon-rpms`
- `rhel-7-server-rhceph-3-osd-rpms`
- `rhel-7-server-rhceph-3-tools-rpms`

### Networking
#### Firewall ports
| service | ports                        |
| ------- | ---------------------------- |
| mon     | 6789/tcp                     |
| mgr     | 7000/tcp, 8003/tcp, 9283/tcp |
| osd     | 6800-7300/tcp                |
| radosgw | 7480/tcp                     |

### Ansible

An Ansible user must have:

* password-less ssh to all hsots
* password-less sudo access
  
  ```user ALL=(root) NOPASSWD:ALL```

The Ansible installation package `ceph-ansible` must be installed. Subsequently, configure:

- `group_vars/all.yml`
- `group_vars/osds.yml`
- `group_vars/clients.yml`
- `group_vars/site.yml`

#### OSD Configuration

The OSD group_vars defines the devices used for storage on the OSDs.

```
devices:
  - /dev/sda
  - /dev/sdb

osd_scenario: "lvm"
```

> Note that 'collocated' is deprecated as of _mimic_ and _lvm_ is the recommended choice.

#### Site configuration

There is an `all.sample.yml` within `/usr/share/ceph-ansible/group_vars/` that can be used as a suitable base to grow from - uncomment and customize.

Some key variables will be:
- monitor_interface
- public_network
- global.osd_pool_default_size
- global.osd_pool_default_min_size

#### Client configuration

If centering your management on particular host(s), define the `copy_admin_key` variable to `true` in the `client.yml` groupvars.

#### Inventory

Create an inventory:

```
[mons]
...
[mgrs]
...
[osds]
...
[clients]
```

Run the playbook:

```
ansible-playbook -i /path/to/inventory site.yml
```

Validate the health of the cluster afterwards by logging on to one of the mons.

```
[user@mon1 ~]$ ceph -s
cluster:
  id:   X
  health: HEALTH_OK
  ...

services:
  osds: X osds: X up, X in  
  ...
```  

## Cluster Administration

### Scale-out / Adding more OSDs

Update the inventory used for installation and add the new OSDs:

```
[osds]
...
newhost
```

### Adding more disks to existing OSDs 

Update the Ansible OSDs groupvars:

- `group_vars/osds.yml`

```
devices:
  - /dev/vda
  - /dev/vdb
  ...
```

Run the playbook:

```
ansible-playbook -i /path/to/inventory site.yml
```

## Storage Management

### Creating a replicated storage pool

```
ceph osd pool create <name> <# pgs> <# effective pgs>
```

> *Relevant Configuration Parameters*
> * `osd_pool_default_size` (number of replicas)
> * `osd_pool_default_min_size` (number of object copies)

### Creating an erasure-coded pool

Use the `erasure` argument at the end of the pool creation command:
```
ceph osd pool create <name> <# pgs> <# effective pgs> erasure <profile>
```

Erasure code profiles can be viewed and customized:

```
ceph osd erasure-code-profile ls
ceph osd erasure-code-profile get <profile>
ceph osd erasure-code-profile set <parameter> <value>
```

Example:
```
ceph osd erasure-code-profile set mynewprofile k=5 m=3
ceph osd pool create newpool 12 12 erasure mynewprofile
```

### Setting pool applications

```
ceph osd pool application enable <pool> <app name>
```

Valid application choices are:

| application name | desc                        |
|----------------- | --------------------------- |
| cephfs           | Ceph file system            |
| rbd              | Ceph block device           |
| rgw              | Object gateway              |

## Monitoring and Management

### System configuration

Useful files:

* `/usr/share/ceph-ansible/group_vars/all.yml` (on Ansible deploy host)
* `/etc/sysconfig/ceph`
* `/etc/ceph/ceph.conf`

Update the `ceph_conf_overrides` section of the Ansible groupvars to set config items.

Pulling live config:

```
ceph daemon <type>.<id> config show
```

### Pools

#### Configuration changes

View:
```
ceph osd pool get <pool> all
```

Set:
```
ceph osd pool set <pool> <config item> <value>
```


#### Statistics

**Displaying pools**: ```ceph osd lspools```
**Disk free**: ```ceph df```
**IO Stats**: ```ceph osd pool stats```

#### Quota

```
ceph osd pool set-quota <pool> max_objects <number>
```
> 0 = unlimited

#### Snapshots

Snapshots can be used to retrieve or restore a historic version of object from the pool at the time the snapshot was taken:

```
ceph osd pool <command>
```
* `mksnap`
* `rmsnap`

```
rados -p <pool> -s <snapshot> get <object> <file>
rados -p <pool> rollback <object> <snapshot>
```

