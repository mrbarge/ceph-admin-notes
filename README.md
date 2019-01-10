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
service | ports
--- | --- 
mon | 6789/tcp
mgr | 7000/tcp, 8003/tcp, 9283/tcp
osd | 6800-7300/tcp
radosgw | 7480/tcp

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

Create an inventory:

```
[mons]
...
[mgrs]
...
[osds]
...
```

Run the playbook:

```
ansible-playbook -i /path/to/inventory site.yml
```

