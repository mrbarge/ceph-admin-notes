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

### Setting live configuration

```
ceph daemon <type>.<id> config set <parameter> <value>
```

### Sending command to daemon

```
ceph tell <type>.<id> <command> '<args>'
```

This also works with wildcards, eg `ceph tell osd.* injectargs '--myarg X'`

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

## CRUSH Map

* OSDs are organized into `buckets`.
* `CRUSH rules` define how placement groups are assigned OSDs from buckets.

Display the current CRUSH map hierachy:

```
ceph osd crush tree
```

Viewing CRUSH device classes (eg HDD, SSD):

```ceph osd crush class ls```

View CRUSH rules:

```ceph osd crush rule ls```

View a specific rule:

```ceph osd crush rule dump <Rule>```

Alternatively, retrieving and processing the raw dump:

```
ceph osd getcrushmap -o crushmap.dat
crushtool -d crushmap.dat -o crushmap.txt
```

Interpreting a rule:

* `min_size`: A pool must make at least this many replicas.
* `max_size`: A pool should not make more than this many replicas
* `step take <X>`: Begin iterating at this bucket hierachy.
* `step chooseleaf firstn <X> type <Y>`: For buckets of type Y, select X many buckets (where X is either a number, or up to the pool's replica limit)

### CRUSH Management

Creating a bucket:

```
ceph osd crush add-bucket <name> <type>
[ceph@host ~] ceph osd crush add-bucket USEast datacenter
```

Placing a bucket into the hierachy:

```
ceph osd crush move <name> <type>=<parent>
[ceph@host ~] ceph osd crush move chassis4 rack=R34
```

Creating and using a rule:

```
ceph osd crush rule create-replicated <name> <starting-node> <replication-bucket-type> [<class>]
ceph osd pool create <pool-name> <pgs> <effective-pgs> <CRUSH-rule>
```

### Placement groups

There should be at least 1-2 orders of magnitude more PGs than OSDs.

PG placement can be optimized with `osdmaptool`:

```
ceph osd getmap -o objmap
osdmaptool objmap --test-map-pgs --pool <pool-ID>
osdmaptool objmap --upmap pg-update-commands.txt --pool <pool-ID>
cat pg-update-commands.txt
```

## Cluster map

Viewing the cluster maps:

```
ceph -s -f json-pretty
```

### Mon maps

Used for tracking the ceph mons.
* Has a leader, elected by quorum, which periodically renews the leadership lease.
* If a peon does not response to lease renewal, a new leader quorum is held (initiated by leader)
* If a peon does not get a new lease, a new leader quorum is held (initiated by peon)
* Configurable via `mon_lease_*` in ceph config parameters.

Acting on the monitor map via `monmaptool`:

```
# ceph mon dump                 (mon map to stdout)
# ceph mon getmap -o <out-raw>  (mon map in raw form)
# monmaptool --print <out-raw>  (mon map in viewable form)
# monmaptool <out-raw> --rm <id>  (remove a mon)
# monmaptool <out-raw> --add <id> <addr:port>  (add a mon)
```

### OSD maps

Used for tracking the ceph OSDs.

Acting on the OSD map with `osdmaptool`:

```
# ceph osd dump                 (osd map to stdout)
# ceph osd getmap -o <map-raw>  (osd map in raw form)
# osdmaptool --print <map-raw>  (osd map in viewable form)
# osdmaptool --export-crush <crush-out> <map-raw> (extract crush map)
# osdmaptool --import-crush <crus-in> <map-raw> (add crush map)
```

Acting on the crush map:

```
crushtool -d <crush-out> -o <decompiled-txt>
crushtool -c <decompiled-txt> -o <compiled-bin>
osdmaptool --import-crush <compiled-crush-bin> <map-raw>
osdmaptool --test-map-pgs-dump <map-raw>
```

## Client Troubleshooting

#### RBD Performance

Image creation options:
* Increase the `stripe_count` config.
* Align `stripe_unit` better with application needs.

#### Debugging client

* Set `debug_ms` to 1 in `/etc/ceph/ceph.conf`, or pass as a command-line flag (`--debug-ms=1`)
* Enable an admin socket to retrieve immediate feedback to commands issued (`--admin-daemon /path/to/sock`)  (ie. pass config to running process such as `ceph --admin-daemon /blah set debug_ms 1`)
* Verify the mon configuration in `/etc/ceph/ceph.conf` is correct.
* Ensure `cephx` is wholly enabled or disabled.
* Validate presence and permissions on keyring files.

#### Debugging RADOS Gateway

* Enable `rgw_enable_ops_log` and `rgw_enable_usage_log` config parameters for additional debugging.

## Monitoring / Management

### Cluster flags

Set via `ceph osd set` and `ceph osd unset`:

| flag | desc |
| ---- | ---- |
| noup | don't start OSDs as UP automatically - prevent flapping |
| nodown | mark a stopping OSD as DOWN - prevent flapping |
| noout | don't remove OSDs from CRUSH - use during maintenance to prevent rebalancing |
| noin | mark an OSD as in |
| norecover | prevent recovery ops from running - use during maintenance |
| nobackfill | prevent backfill ops - use during maintenance |
| noscrub | prevent scrubbing ops - use to prevent slow-scrubbing OSDs being marked as down |
| nodeep-scrub | as above but for deep scrubbing |
| norebalance | prevent rebalancing ops - use during maintenance |

### Cluster parameters: OSDs

* osd_heartbeat_interval
* osd_heartbeat_grace
* mon_osd_min_down_reporters (peers that must report OSD as down before mon agrees it is down)
* mon_osd_min_down_reports (times that an OSD must be reported as down before mon agrees it is down)
* osd_mon_report_interval_min
* osd_mon_report_interval_max
* osd_mon_heartbeat_interval
* mon_osd_report_timeout

### Server Troubleshooting

#### HEALTH_WARN/ERR and OSDs are near capacity

Triggered by hitting the `mon_osd_nearfull_ratio` or `mon_osd_full_ratio` parameter.

Add OSDs or delete data.

#### Learning Mon/OSD usage stats

View all options for retrieving usage stats:

```
ceph daemon osd.<id> help
ceph daemon mon.<host> help
```

#### OSD is both down and in

Cluster gives a temporarily down OSD a chance to recover/rejoin to avoid scrubbing/rebalancing.

After a timeout, OSD should switch to `down` and `out`, and PGs are migrated.

#### PGs stuck in states

* Stuck in `inactive` - peering issue
* Stuck in `unclean` - can't recover from failure
* Stuck in `stale` - no OSDs reporting - at least one of the OSDs need to be brought up
* Stuck in `undersized` - not enough OSDs available.

Viewing PG status:

```
ceph pg dump
```

#### Removing failed OSD

* Verify state of OSD as `down` / `out`: ```ceph osd tree | grep <osd>```
* Take the OSD out and stop its service:
```
ceph osd out <osd>
systemctl stop ceph-osd@<id>.service
```
* Remove OSD from CRUSH: ```ceph osd crush remove osd.<id>```
* Remove key rings for OSD: ```ceph auth del osd.<id>```
* Remove the OSD: ```ceph osd rm osd.<id>```

#### Migrating an OSD journal to a new device

```
# prevent rebalancing
ceph osd set noout

# stop service
systemctl stop ceph-osd@X.service

# flush journal
ceph-osd -i X --flush-journal

# remove journal link
rm -f /var/lib/ceph/osd/ceph-X/journal

# soft-link to new device
ln -s /dev/device /var/lib/ceph/osd/ceph-X/journal

# create journal
ceph-osd -i X --mkjournal

# start service
systemctl start ceph-osd@X.service

# disable noout
ceph osd unset noout
```

#### Establishing admin socket to mon/osd

```
ceph daemon <type>.<id> <command>

or 

ceph --admin-daemon <socket> <command>
```

## Performance

### OS Tuning

Important `sysctl` parameters:

* net.ipv4.tcp_{r,w}mem (buffer send/receive sizes: min, default and max)
* net.ipv4.tcp_mem (tcp memory usage: low threshold, pressure start, max pages)
* net.core_{r,w}mem_max (max buffer size for all connections)
* vm.dirty_background_ratio (total dirty memory causing kernel to write out in background)
* vm.dirty_ratio (total dirty memory causing a writing process to block during flush)

Using `tuned` to automatically set up a performance profile:

```
# tuned-adm list
# tuned-adm profile network-throughput
# tuned-adm off
```

Per-device I/O scheduling methods are available at `/sys/block/<dev>/queue/scheduler`.
* noop (fifo)
* deadline (groups into read/write batches, best for large I/O ops)
* cfq (large and small requests at once)

Options for creating file-systems are configurable in `osd_mkfs_options_xfs` in ceph-ansible.

Options for mounting file-systems are configurable in `osd_mount_options_xfs` in ceph-ansible.

### Increasing per-pool PGs

Update the number of PGS:
```
ceph osd pool set rbd pg_num <num>
```

Gradually increase number of current PGs until `pgp_num` == `pg_num` and cluster is rebalanced:
```
ceph osd pool set rbd pgp_num <num>
```

### Accessing metrics

Use `ceph daemon <type>.<id> perf schema` to retrieve all possible metrics available.

```
ceph daemon osd.1 perf dump
```

Then, use `ceph daemon <type>.<id> perf dump` or `ceph osd pool stats <pool>` to retrieve values.

The most recent processed operations can be retrieved with `ceph daemon <type>.<id> dump_historic_ops`. The following config options control this behaviour:
* `osd_op_history_size`: max number of completed options to track
* `osd_op_history_duration`: oldest op to track
These can be changed with commands like the following: `ceph tell <type>.<id> injectargs '--osd_op_history_duration 60'`

`ceph daemonperf <type>.<id>` displays performance stats for a particular daemon.

### Stress-testing

#### Object store

```
rados -p <pool> bench <seconds> write|seq|rand -b <default obj size> -t <concurrent ops>
```

#### Block devices

```
rbd -p <pool> bench <image> --io-type read|write [--io-size, --io-threads, --io-total, --io-pattern]
```
### Filestore (pre-Luminous Cephs)

Files in `/var/lib/ceph/osd` are split and merged with behaviour managed by the following config parameters:

* `filterstore_merge_threshold` (min files per dir)
* `filestore_split_multiple` (when splitting occurs)
* `filestore_split_rand_factor` (controls randomization of splitting)


### Scrubbing, backfill and recovery

Controlled via:

```
ceph osd set noscrub
ceph osd unset noscrub
```

Configurable via:

| parameter | desc |
| --------- | ---- |
| `osd_scrub_(end,begin)_hour` | hour scrubbing begins/ends |
| `osd_scrub_load_threshold` | scrub when load less than threshold> |
| `osd_scrub_min_interval` | scrub no more than X seconds |
| `osd_scrub_max_interval` | don't wait to scrub longer than X |
| `osd_scrub_priority` | priority of scrub ops |
| `osd_deep_scrub_interval` | interval between deep scrubs |
| `osd_Scrub_sleep` | pause between deep scrub ops |

Scrubbing stats are observable via `ceph pg dump`
Scrubbing is intitiated via `ceph pg scrub`
Deep-scrubbing is initiated via `ceph pg deep-scrub`

Concurrent backfill ops are controlled via `osd_max_backfills`
Concurrent recovery ops are controlled via `osd_recovery_max_active`
Recovery priority is controlled via `osd_recoverY_op_priority`

## OpenStack integration

### Glance

#### Ceph user and auth setup 

Create a dedicated ceph storage pool and enable RBD storage:

```
ceph osd pool create <pool> <pg nums>
ceph osd pool application enable <pool> rbd
```

Create a cephx user for glance to use:

```
ceph auth get-or-create <user> mon 'profile rbd' \
  osd 'profile rbd pool=<pool>' \
  -o /etc/ceph/ceph.<user>.keyring
```

Add the user to a user-specific block in `/etc/ceph/ceph.conf`:

```
[<user>]
keyring = /etc/ceph/ceph.<user>.keyring
```

Distribute this config and the keyring to all glance hosts and ensure the glance user will be able to read it:

```
sudo chgrp glance /etc/ceph/ceph.<user>.keyring
sudo chmod 0640 /etc/ceph/ceph.<user>.keyring
```

#### Glance service config

Set the following in `/etc/glance/glance-api.conf`:

```
[glance_store]
default_store = rbd
show_image_direct_url = True
stores = rbd
rbd_store_user = <user>
rbd_store_pool = <pool>
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_Store_chunk_size = <num>
```

Restart the glance service:

```
sudo systemctl restart openstack-glance-api
```

Test an image push and then query it with rbd API:

```
openstack image create --container-format bare --disk-format raw --file test.img "Test"
rbd --id <user> -p <pool> ls
rbd --id <user> info <pool>/<image>
```

### Cinder

#### Ceph user and auth setup

Create a dedicated ceph storage pool:

```
ceph osd pool create <pool> <pg nums>
ceph osd pool application enable <pool> rbd
```

Create a cephx user for cinder to use, it must also have read-only access to the glance pool:

```
ceph auth get-or-create <user> mon 'profile rbd' \
  osd 'profile rbd pool=<cinder-pool>, profile rbd pool=<vm-pool>, profile rbd-read-only pool=<glance-pool>' \
  -o /etc/ceph/ceph.<user>.keyring

```

Add the user to a user-specific block in `/etc/ceph/ceph.conf`:

```
[<user>]
keyring = /etc/ceph/ceph.<user>.keyring
```

Distribute this config and the keyring to all glance hosts and ensure the glance user will be able to read it:

```
sudo chown cinder:cinder /etc/ceph/ceph.<user>.keyring
sudo chmod 0640 /etc/ceph/ceph.<user>.keyring
```

#### Cinder service configuration

Edit `/etc/cinder/cinder.conf` and set:

```
[DEFAULT]
enabled_backends = ceph
# comment out default_volume_type if set

[ceph]
rbd_ceph_conf=/etc/ceph/ceph.conf
rbd_pool=<cinder-pool>
rbd_secret_uuid=<uuid>
rbd_user=<user>
```
(generate the UUID with `uuidgen`)

Hypervisor libvirts must be updated via a file (eg. `/tmp/cephsecret`):
```
<secret ephemeral="no" private="no">
  <uuid>$uuid</uuid>
  <usage type="ceph">
    <name>$user secret</name>
  </usage>
</secret>
```

Then: 

```
virsh secret-define --file /tmp/cephsecret
virsh secret-set-value --secret <uuid> --base64 $(cat <keyring>)
```

Lastly restart the service:

```
systemctl restart openstack-cinder-api
systemctl restart openstack-cinder-volume
systemctl restart openstack-cinder-scheduler
```

#### Nova service config

Edit `/etc/nova/nova.conf` and set:

```
[libvirt]
libvirt_images_rbd_pool = <vm-pool>
libvirt_iamges_rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_secret_uuid = <uuid>
rbd_user = <user>
```

## Swift

### Keystone setup

Define a service and endpoint for the radosgw gateway:

```
openstack service create --name swift object-store
openstack endpoint create --region <region> --publicurl "http://radosgw:8080/swift/v1" \
  --adminurl "http://radosgw:8080/swift/v1" \
  --internalurl "http://radosgw:8080/swift/v1"
  swift
```

Set up the Rados GW to talk to ceph in `/etc/ceph/ceph.conf`:

```
[client.rgw.radosgw]
rgw_keystore_url = http://keystore-host:35357
rgw_keystore_admin_tenant = <keystone admin tenant>
rgw_keystone_admin_user = <keystone admin user>
rgw_keystone_admin_password = <keystone admin pass>
# alternatively..
rgw_keystone_admin_token = <token, obtained from /etc/keystore/keystone.conf>

rgw_keystone_accepted_roles = admin member swiftoperator
rgw_keystone_token_cache_size = 200
rgw_keystone_revocation_interval = 300
nss_db_path = /var/ceph/nss
```

Convert the keystone certs into NSS format:

```
mkdir /var/ceph/nss
openssl x509 -in /etc/keystone/ssl/cert/ca.pem -pubkey | certutil -d /var/ceph/nss -A -n ca -t "TCu,Cu,Tuw"
openssl x509 -in /etc/keystone/ssl/cert/signing_cert.pem -pubkey | certutil -d /var/ceph/nss -A -n signing_cert -t "Tcu,Cu,Tuw"
```

Then restart the Rados GW service: `sudo systemctl restart ceph-radosgw.target`

Test swift:

```
swift -v stat
swift -v list
swift -v post <object>
swift -v upload <object> <file>
```

## Notes

**Installation**

```
# Install ceph-ansible
yum install ceph-ansible -y

cd /usr/share/ceph-ansible

# Not essential...
# Set log_path = /tmp/ansible.log
# Set deprecation_warnings = false
sudo vim ansible.cfg

vim /etc/ansible/hosts
```

Define cluster hosts:
```
[mons]
[mgrs]
[osds]
[clients]
```

Edit deployment play:
```
cp /usr/share/ceph-ansible/site.yml.sample /usr/share/ceph-ansible/site.yml
vim /usr/share/ceph-ansible/site.yml

# Set OSD installation to be serial
# serial: 1
```

Set group vars:
```
sudo cp /usr/share/ceph-ansible/group_vars/all.yml.sample /usr/share/ceph-ansible/group_vars/all.yml
vim /usr/share/ceph-ansible/group_vars/all.yml
```

Modifications:
```
ntp_service_enabled: false

ceph_rhcs_version: "3"
ceph_repository_type: cdn

rbd_cache: "true"
rbd_cache_writethrough_until_flush: "false"
rbd_client_directories: false

monitor_interface: <x>

journal_size: 1024
public_network: <x>
cluster_network: "{{ public_network }}"

ceph_conf_overrides:
  global:
    mon_allow_pool_delete: true
    mon_osd_allow_primary_affinity: 1
    mon_clock_drift_allowed: 0.5
    osd_pool_default_size: 2
    osd_pool_default_min_size: 1
    mon_pg_warn_min_per_osd: 0
  client:
    rbd_default_features: 1
```

Set OSD vars:

```
sudo cp /usr/share/ceph-ansible/group_vars/osds.yml.sample /usr/share/ceph-ansible/group_vars/osds.yml
vim /usr/share/ceph-ansible/group_vars/osds.yml
```

Define:
```
---
devices:
  - /dev/X
  - /dev/Y
  
osd_scenario: "collocated"
```

Set up clients:

```
sudo cp /usr/share/ceph-ansible/group_vars/clients.yml.sample /usr/share/ceph-ansible/group_vars/clients.yml
vim /usr/share/ceph-ansible/group_vars/clients.yml
```

Define:
```
copy_admin_key: true
```
