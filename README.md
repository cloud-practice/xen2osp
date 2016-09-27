# xen2osp (via ansible)

This is a simple ansible playbook that will take a running instance on XenServer 6.5 and migrate it over to OpenStack's image store, glance.

Most of the heavy lifting is done by [virt-v2v](http://libguestfs.org/virt-v2v.1.html). While it supports exporting from Xen, it doesn't seem to like to talk to XenServer 6.5. The playbooks build [xva-img](https://github.com/eriklax/xva-img) as an intermediate conversion step. There is an [RFE](https://bugzilla.redhat.com/show_bug.cgi?id=1253593) for virt-v2v to get proper support for Citrix Xen, but for now we lean on xva-img.

##Setup

Modify the `./group_vars/all` file to suit your environment:

```
---
nfs_mount: '/xen2osp'
nfs_share: 'nfsserver.example.com:/exports/fileshare/xen'
osp_env:
  OS_USERNAME: 'user0'
  OS_TENANT_NAME: 'tenant0'
  OS_PASSWORD: 'P@ssw0rd'
  OS_AUTH_URL: 'http://openstack.example.com:5000/v2.0/'
  LIBGUESTFS_BACKEND: 'direct'
```

And set the right entries in the hosts file:

```
[builder]
builder.example.com
[xenserver]
xenserver.example.com
```

##Running it

When the playbook is run, a few things will happen:
- both servers will mount a common NFS share
- the instance will be stopped and exported from the xenserver
- the builder will sync and build the xva-img repo (first time only)
- then the builder will run virt-v2v against the raw image and upload to glance

The playbook is run, supplying an extra argument to specifc the VM to export. It is invoked like so:

```
# ansible-playbook -i hosts xen2osp.yaml --extra-vars "vm_name=rhel7xenvm0"

PLAY [all] *********************************************************************

TASK [setup] *******************************************************************
ok: [xenserver.example.com]
ok: [builder.example.com]

TASK [mount nfs share] *********************************************************
ok: [xenserver.example.com]
ok: [builder.example.com]

PLAY [xenserver] ***************************************************************

TASK [setup] *******************************************************************
ok: [xenserver.example.com]

TASK [get vm power state] ******************************************************
changed: [xenserver.example.com]

TASK [debug] *******************************************************************
skipping: [xenserver.example.com]

TASK [stop vm] *****************************************************************
changed: [xenserver.example.com]

TASK [export image] ************************************************************
ok: [xenserver.example.com]

PLAY [builder] *****************************************************************

TASK [setup] *******************************************************************
ok: [builder.example.com]

TASK [install required packages] ***********************************************
ok: [builder.example.com] => (item=[u'virt-v2v', u'cmake', u'gcc-c++', u'python-glanceclient'])

TASK [clone xva-img git repo] **************************************************
ok: [builder.example.com]

TASK [run cmake on project] ****************************************************
ok: [builder.example.com]

TASK [make and install xva-img binary] *****************************************
ok: [builder.example.com]

TASK [create dir for unpacked image] *******************************************
changed: [builder.example.com]

TASK [unpack xva tarball] ******************************************************
changed: [builder.example.com]

TASK [get weird Ref number] ****************************************************
changed: [builder.example.com]

TASK [debug] *******************************************************************
skipping: [builder.example.com]

TASK [convert xva to raw] ******************************************************
changed: [builder.example.com]

TASK [prep image and upload to glance] *****************************************
changed: [builder.example.com]

PLAY RECAP *********************************************************************
builder.example.com        : ok=12   changed=5    unreachable=0    failed=0   
xenserver.example.com      : ok=6    changed=2    unreachable=0    failed=0   

#
```

If you already have a bunch of VMs exported from Xen and just need to convert and import them to glance, the playbook can be run with a filter against only the builder host. It will skip all the Xen steps, and assume an exported image is already available on the NFS share:

```
# ansible-playbook -i hosts xen2osp.yaml --extra-vars "vm_name=rhel6srv" -l builder

PLAY [all] *********************************************************************

TASK [setup] *******************************************************************
ok: [builder.example.com]

TASK [mount nfs share] *********************************************************
ok: [builder.example.com]

PLAY [xenserver] ***************************************************************
skipping: no hosts matched

PLAY [builder] *****************************************************************

TASK [setup] *******************************************************************
ok: [builder.example.com]

TASK [install required packages] ***********************************************
ok: [builder.example.com] => (item=[u'virt-v2v', u'cmake', u'gcc-c++', u'python-glanceclient'])

TASK [clone xva-img git repo] **************************************************
ok: [builder.example.com]

TASK [run cmake on project] ****************************************************
ok: [builder.example.com]

TASK [make and install xva-img binary] *****************************************
ok: [builder.example.com]

TASK [create dir for unpacked image] *******************************************
changed: [builder.example.com]

TASK [unpack xva tarball] ******************************************************
changed: [builder.example.com]

TASK [get weird Ref number] ****************************************************
changed: [builder.example.com]

TASK [debug] *******************************************************************
skipping: [builder.example.com]

TASK [convert xva to raw] ******************************************************
changed: [builder.example.com]

TASK [prep image and upload to glance] *****************************************
changed: [builder.example.com]

PLAY RECAP *********************************************************************
builder.example.com        : ok=12   changed=5    unreachable=0    failed=0   

# 
```

The image is now uploaded in glance, and can be used to boot a new instance:

```
$ glance image-list
+--------------------------------------+------------------------+-------------+------------------+--------------+--------+
| ID                                   | Name                   | Disk Format | Container Format | Size         | Status |
+--------------------------------------+------------------------+-------------+------------------+--------------+--------+
| 5b4414b5-bc54-418d-8845-742843deef85 | rhel6srv               | raw         | bare             | 10737418240  | active |
| 6d6e5048-17b6-49f3-a5a6-fe9e6d6c66ba | rhel7xenvm0            | raw         | bare             | 10737418240  | active |
+--------------------------------------+------------------------+-------------+------------------+--------------+--------+
$ nova boot xen2osp --flavor 2 --image  rhel7xenvm0 --key-name user0
+--------------------------------------+----------------------------------------------------+
| Property                             | Value                                              |
+--------------------------------------+----------------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                             |
| OS-EXT-AZ:availability_zone          |                                                    |
| OS-EXT-SRV-ATTR:host                 | -                                                  |
| OS-EXT-SRV-ATTR:hostname             | xen2osp                                            |
| OS-EXT-SRV-ATTR:hypervisor_hostname  | -                                                  |
| OS-EXT-SRV-ATTR:instance_name        | instance-00000089                                  |
| OS-EXT-SRV-ATTR:kernel_id            |                                                    |
| OS-EXT-SRV-ATTR:launch_index         | 0                                                  |
| OS-EXT-SRV-ATTR:ramdisk_id           |                                                    |
| OS-EXT-SRV-ATTR:reservation_id       | r-k25zzuf3                                         |
| OS-EXT-SRV-ATTR:root_device_name     | -                                                  |
| OS-EXT-SRV-ATTR:user_data            | -                                                  |
| OS-EXT-STS:power_state               | 0                                                  |
| OS-EXT-STS:task_state                | scheduling                                         |
| OS-EXT-STS:vm_state                  | building                                           |
| OS-SRV-USG:launched_at               | -                                                  |
| OS-SRV-USG:terminated_at             | -                                                  |
| accessIPv4                           |                                                    |
| accessIPv6                           |                                                    |
| adminPass                            | 2Yonn25fQShX                                       |
| config_drive                         |                                                    |
| created                              | 2016-09-27T15:46:42Z                               |
| flavor                               | m1.small (2)                                       |
| hostId                               |                                                    |
| id                                   | 92b74cb5-9df2-44ff-a93e-4977596e19f5               |
| image                                | rhel7xenvm0 (6d6e5048-17b6-49f3-a5a6-fe9e6d6c66ba) |
| key_name                             | user0                                              |
| locked                               | False                                              |
| metadata                             | {}                                                 |
| name                                 | xen2osp                                            |
| os-extended-volumes:volumes_attached | []                                                 |
| progress                             | 0                                                  |
| security_groups                      | default                                            |
| status                               | BUILD                                              |
| tenant_id                            | 8e3a8cd43e84481aa35933075b9b42fb                   |
| updated                              | 2016-09-27T15:46:42Z                               |
| user_id                              | 128c53f930e14d86acf5bdc717cc7b4f                   |
+--------------------------------------+----------------------------------------------------+
$
```

That's it! If you hit any bugs, let me know.


