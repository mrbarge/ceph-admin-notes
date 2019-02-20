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

### Authentication and Authorization

#### Viewing or interrogating user configuration

Listing users:
```
ceph auth list
client.openstack
  key: abc123
  caps: [osd] allow *
```

Showing a user:
```
ceph auth get client.openstack
```

Dumping a user config to file:
```
ceph auth export client.openstack > client.openstack.cfg
```

#### Creating users
A user and their roles are created as follows:

```
ceph auth get-or-create <user> mon '<perms>' osd '<perms> -o /path/to/keyring'
```

* Permissions can specify specific read/write permissions, or defer to a profile. (`allow rw`)
* Permissions can be restricted to specific pools or namespaces. (`allow rw pool=mypool namespace=blah`) (_Specify pool before namespace_) 
* Permissions can be restricted to specific object name prefixes. (`allow rw object_prefix hello`)
* Permissions can be restricted to CephFS paths (`allow rw path=/data`)
* Permissions can be restricted to specific monitor commands (`allow command "osd lspools"`)

Example:

```
ceph auth get-or-create client.application \
   osd 'allow rw pool=app' mon 'profile appprofile' \
   -o /etc/ceph/ceph.client.application.keyring
```

Permissions can be updated after the fact with `ceph auth caps`:

```
ceph auth caps client.application mon 'allow r' osd 'allow rw pool=blah object_prefix hello'
```

The user can be supplied to ceph commands using the `--id` argument:

```
ceph --id client.application health
rados --id application -p mypool ls
```

#### Deleting users

```
ceph auth del client.openstack
rm /etc/ceph/ceph.client.openstack.keyring
```

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
[ceph@host ~$] ceph osd erasure-code-profile set newprofile k=5 m=3
[ceph@host ~$] ceph osd erasure-code-profile ls
default
ceph125
newprofile
[ceph@host ~$] ceph osd erasure-code-profile get newprofile
k=5
m=3
...
[ceph@host ~$] ceph osd pool create mypool 32 erasure newprofile
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

#### Pulling config parameters

```
ceph daemon mon.server config get <parameter>
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

## Rados Block Device / Block Storage

### Pre-requisites

Create a pool and enable rbd

```
ceph osd pool create mypool <parms>
ceph osd pool application enable rbd
rbd pool init mypool
```

### Image management

#### Mapping and unmapping as a service device:

```
[root@server ~] rbd map rbd/imagename
/dev/rbd0

[root@server ~] rbd showmapped
id pool image snap device
0  rbd  imagename - /dev/rbd0

[root@server ~] rbd unmap /dev/rbd0
```

#### Customizing:

```
rbd feature enable rbd/imagename <feature>
```

#### Snapshotting:

```
rbd snap create mypool/imagename@snapname
rbd snap protect mypool/imagename@snapname
rbd clone mypool/imagename@snapname mypool/imageclonename
rbd flatten mypool/imageclonename
```

#### Exporting and Importing

```
rbd export <image/snapshot> <path>
rbd import <path> <image>
```

Detecting changes between exports:
```
rbd-export-diff --from-snap <snap> <image>
rbd-export-diff  <image> <path>
```

Using it as an image transfer channel:
```
cat <image> | ssh <server> rbd import - <image>
```

### Caching

Caching is configured in the `[client]` section of `/etc/ceph/ceph.conf`:

| parameter | meaning |
|---------- | ------- |
| rbd_cache | Enable caching (true/false) |
| rbd_cache_size | cache size in bytes |
| rbd_cache_writethough_until_flush | Write-though caching until first flush (True/false) |

### iSCSI 

#### Gateway Setup

Enable via Ansible:
```
[iscsi-gws]
host1
host2
```

Set the `group_vars/iscsi-gws.yml`:

```
---
gateway_iqn: "<shared GW IQN>"
gateway_ip_list: "<comma-separated-IPs>"

# override if cluster name is not default
cluster_name: prod

# must be set for the first play run
perform_system_checks: True

# Images to export via iSCSI
rbd_devices:
  - { pool: '<pool name>', image: '<image name>', size: '<image size>', host: '<iSCSI GW host>', state: 'present' }
  
client_connections:
  - { client: '<client iqn>', image_list: '<poolname>.<imagename>'', chap: '<user>/<pass>', status: 'present' }
```

After deployment, enable and start the service on the gateway hosts, `rbd-target-api` and `rbd-target-gw`

Setup health monitoring with:
```
yum install ceph-iscsi-tools
systemctl enable pmcd
/var/lib/pcp/pmdas/ilo/Install
gwtop
```

#### Initiator Setup

* Install the `iscsi-initiator-utils` package.
* Install the `device-mapper-multipath` package.
* Enable multipath

```
mpathconf --enable --with_multipathd y
cat << EOF >> /etc/multipath.conf
devices {
  device {
    vendor "LIO-ORG"
    hardware_handler "1 alua"
    path_grouping_policy "failover"
    path_selector "queue-length 0"
    failback 60
    path_checker tur
    prio alua
    prio_args exclusive_pref_bit
    fast_io_fail_tmo 25
    no_path_retry queue
  }
}
systemctl restart multipathd
```
* Update auth credentials in `/etc/iscsi/iscsid.conf`
* iSCSI discovery and attachment:

```
iscsiadm -m discovery -t -p <agteway ip>
iscsiadm -m node -T <gateway iqn> -l
```

## Multi-cluster Administration

### RBD Mirroring

> Each cluster must have a unique name.  For example, if both clusters are named _ceph_ this won't work.

Steps:
* (For both clusters) Create a dedicated mirroring user with appropriate permissions to access the pool to be mirrored.
* (For both clusters) Distribute the cluster config (`/etc/ceph/<clustername>.conf`) to its peer.
* (For both clusters) Distribute the mirror user's keyring to its peer.

On the passive cluster:
* `yum install rbd-mirror` 
* `systemctl enable ceph-rbd-mirror.target --now`
* `rbd mirror pool enable rbd pool --cluster <passive cluster>`
* `rbd mirror pool enable rbd pool --cluster <active cluster>`
* `rbd mirror pool peer add rbd <mirror user>@<active cluster> --cluster <passive cluster>`

## RADOS Gateway / Object Storage

### Installation

Define hosts in the Ansible inventory:

```
[rgws]
rgwhost1
...
```

Define `rgws` group_vars:
```
# Distributes the admin key to the RGWs
copy_admin_key: true
```

Define global group_vars if required:

| variable | definition |
| -------- | ---------- |
| fetch_directory | cluster dir |
| rados_civetweb_port | Civetweb port (default 7480) |
| radosgw_interface | Host network interface to listen on |
| radosgw_frontend | `civetweb` for civetweb-fronted interface |
| ceph_conf_overrides::client.rgw.<host>::rgw_dns_name | fqdn used for url |

### User Setup

Creation:

```
radosgw-admin user create --uid=<user> --display-name="User" \
  [ --access-key=<AWS key> --secret=<AWS secret> ]
```

Quotas:

* `radosgw-admin user info --uid=<user uid>`
* `radosgw-admin user stats --uid=<user uid>`
* `radosgw-admin usage show --show-log-entries=false`

Subusers:

```
radosgw-admin subuser create --uid=<user> --subuser=<user>:<subuser> --access=full
```

### Driving S3 API

Setup `dnsmasq` on cclient machine in order to do correct S3 object name resolution:

`/etc/dnsmasq.d/ceph.conf`
```
address=/.HOST/IP
```

Pre-configure the S3 
Use `s3cmd` for bucket/object actions:

```
yum install s3cmd
s3cmd --configure
# edit host_base and host_bucket in the generated $HOME/.s3cfg
```

* Creation of bucket: `s3cmd mb s3://bucketname`
* Putting object (public): `s3cmd put --acl-public myfile s3://bucketname/objectname`

Set policy with `s3cmd setpolicy`:

* `s3cmd setpolicy mypolicy s3://bucketname`

### Integrating OpenStack

OpenStack swift users map from `<tenant>:<user>` to Ceph's `<user>:<subuser>`.

In addition to creating a dedicated subuser, create a key with the `--keytype` of `swift`:

```
radosgw-admin key create --subuser=<suer>:swift --key-type=swift
```

The `rgw_swift_versioning_enabled` option must be set in `/etc/ceph.conf` to enable Swift's object versioning.

The user can be validated by installing the `python-swiftclient` package:

```
yum install python-swiftclient
swift -V 1.0 -A http://$SERVER/auth/v1 -U <user>:<subuser> -K <secret> stat
```

### Multi-zoned

Primary:
```
radosgw-admin realm create --default --rgw-realm=<name>
radosgw-admin zonegroup create --rgw-zonegroup=us --master \
  --default --endpoints=http://localhost:8000
radosgw-admin zone create --rgw-zone=<primary region> --master \
  --rgw-zonegroup=us --endpoints http://localhost:8000
  --access-key=<key> --secret=<secret> --default
radosgw-admin user create --uid=<user> --display-name="User" \
  --access-key=<key> --secret=<secret> --system
radosgw-admin period update --commit
systemctl restart ceph-radosgw@*
```

Secondary:
```
radosgw-admin realm pull --rgw-realm=<name> --url=http://localhost:8000 \
  --access-key=<key> --secret=<secret> --default
radosgw-admin period pull --url=http://localhost:8000 \
  --access-key=<key> --secret=<secret> --default
radosgw-admin zone create --rgw-zone=<secondary region> --rgw-zonegroup=us \
  --endpoints=http://localhost:8001 --access-key=<key> \
  --secret=<secret> --default
radosgw-admin period update --commit
systemctl restart ceph-radosgw@*
```

Failover to secondary:

```
# On Primary..
radosgw-admin zone modify --master --rgw-zone=<secondary region>
radosgw-admin zonegroup modify --rgw-zonegroup=us --endpoints=http://localhost:8001
radosgw-admin period update --commit
```

#### Walkthrough

Primary:
```
# create realm
radosgw-admin realm create --rgw-realm=X --default
# delete existing default zonegroup
radosgw-admin zonegroup delete --rgw-zonegroup=default
# Create the new zonegroup
radosgw-admin zonegroup create --rgw-zonegroup=Y --endpoints=http://$HOST:80 --master --default
# create new zone
radosgw-admin zone create --rgw-zonegroup=Y --rgw-zone=Z --endpoints=http://$HOST:80 \
  --access-key=abc123 --secret=def456
# create replication user
radosgw-admin user create --uid="$USER" --display-name="$USERNAME" --access-key=abc123 \
  --secret=def456 --system
# commit realm structure
radosgw-admin period update --commit
# set the zone in /etc/ceph/ceph.conf
# add rgw_zone = Z
# disable resharding for multisite
# add rgw_dynamic_resharding = false
systemctl restart ceph-radosgw@*
```

Secondary:
```
radosgw-admin realm pull --url=http://$HOST:80 \
  --access-key=abc123 --secret=def456
radosgw-admin period pull --url=http://$HOST:80 \
  --access-key=abc123 --secret=def456
radosgw-admin zone create --rgw-zonegroup=classroom --rgw-zone=Z2 \
  --endpoints=http://$HOST:80 --access-key=abc123 --secret=def456 --default  
radosgw-admin period update --commit --rgw-zone=Z2
# set the zone in /etc/ceph/ceph.conf
# add rgw_zone = Z2
# disable dynamic resharding as per primary
systemctl restart ceph-radosgw@*
```

## CephFS

### Installation

Define hosts in the Ansible inventory for the Metadata Server:

```
[mdss]
mdshost1
...
```

Define `mdss` group_vars:
```
# Distributes the admin key to the MDSs
copy_admin_key: true
```

Define global group_vars if required:

| variable | definition |
| -------- | ---------- |
| fetch_directory | cluster dir |

Launch the play:

```
ansible-playbook site.yml --limit mdss
```

### Managing filesystems

#### Initialisation 

Creating a CephFS filesystem:

```
# the following parameters are defaults created during playbook run
ceph fs new cephfs cephfs_metadata cephfs_data
```

Preparing a host for CephFS filesystem use:

* Install `ceph-common`
* Distribute `ceph.conf` to the host from the cluster.
* Install the appropriate key for FUSE or kernel

#### Mounting (FUSE)

Manually mounting a CephFS filesystem (FUSE):

```
ceph-fuse -m monhost /mnt/fs
fusermount -u /mnt/fs
```

`/etc/fstab` equivalent:

```
none /mnt/fs fuse.ceph _netdev 0 0
```

#### Mounting (kernel)

```
mount -t ceph monhost:/ /mnt/fs
# optionally, mount a subdir
mount -t ceph monhost:/data /mnt/fs
```

`/etc/fstab` equivalent:

```
monhost:/ /mnt/fs ceph name=<cephx user>,secretfile=<secret key path>,_netdev 0 0
```

### Debugging (from a client)

#### Mapping file to object

```
INODE=`stat -c %i /myfile`
INODE_HEX=`printf '%x\n' $INODE`
RADOS_OBJ=`rados -p cephfs_data ls | grep $INODE_HEX`
ceph osd map cephfs_data $RADOS_OBJ
```

Will return format indicating:

```
osdmap <map epoch> pool '<pool>' (<pool id>) object \
  '<RADOS object ID>' -> pg <placement group> -> <status> ([<osds>], <primary osd>)
```

#### Viewing extended ceph fileattributes

```
getfattr -d -m ceph.* <file/dir>
```

File attributes:
* `ceph.file.layout.pool`
* `ceph.file.layout.stripe_unit`
* `ceph.file.layout.stripe_count`
* `ceph.file.layout.object_size`
* `ceph.file.layout.pool_namespace`

Dir layout attributes:
* `ceph.dir.layout.pool`
* `ceph.dir.layout.stripe_unit`
* `ceph.dir.layout.stripe_count`
* `ceph.dir.layout.object_size`
* `ceph.dir.layout.pool_namespace`

Dir attributes:
* `ceph.dir.entries` (descendents)
* `ceph.dir.files` (num files in dir)
* `ceph.dir.rbytes` (total file size in subtree)
* `ceph.dir.rctime` (most recent create time)
* `ceph.dir.rentries` (recursive descendents)
* `ceph.dir.rfiles` (files in subtree)
* `ceph.dir.rsubdirs` (dirs in subtree)
* `ceph.dir.subdirs` (dirs in dir)

Attributes can be changed with `setfattr`.


