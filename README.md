# xen2osp (via ansible)

This is a simple ansible playbook that will take a running instance on XenServer 6.5 and migrate it over to OpenStacke.

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
- the image is converted in to a bootable cinder volume
- a nova instance is booted with the newly created volume

The playbook is run supplying an extra argument to specifc the VM to export. At the end of a run, the IP of the new instance is displayed. It is invoked like so:

```
# ansible-playbook -i hosts xen2cinder2osp.yaml --extra-vars "vm_name=rhel7xenvm0"

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

TASK [upload volume to cinder] *************************************************
changed: [builder.example.com]

TASK [boot an instance in nova] ************************************************
changed: [builder.example.com]

TASK [debug] *******************************************************************
ok: [builder.example.com] => {
    "os_server.server.accessIPv4": "10.12.133.181"
}

PLAY RECAP *********************************************************************
builder.example.com        : ok=15   changed=7    unreachable=0    failed=0
xenserver.example.com      : ok=6    changed=3    unreachable=0    failed=0

#
```

The VM is now accessible from its new IP:

```
$ ssh root@10.12.133.181
root@10.12.133.181's password:
Last login: Tue Sep 27 17:07:33 2016 from 10.12.33.38
[root@rhel7xenvm0 ~]#
```

If you already have a bunch of VMs exported from Xen and just need to convert and import them to glance, the playbook can be run with a filter against only the builder host. It will skip all the Xen steps, and assume an exported image is already available on the NFS share:

```
# ansible-playbook -i hosts xen2cinder2osp.yaml --extra-vars "vm_name=rhel6srv" -l builder

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

TASK [upload volume to cinder] *************************************************
changed: [builder.example.com]

TASK [boot an instance in nova] ************************************************
changed: [builder.example.com]

TASK [debug] *******************************************************************
ok: [builder.example.com] => {
    "os_server.server.accessIPv4": "10.12.133.182"
}

PLAY RECAP *********************************************************************
builder.example.com        : ok=15   changed=7    unreachable=0    failed=0

#
```

In addition to the main playbook, `xen2osp.yaml` has been created which skips the cinder steps, and boots an ephermeral instance backed by glance.

That's it! If you hit any bugs, let me know.


