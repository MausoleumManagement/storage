## Reading Material
- https://docs.ceph.com/en/reef/cephfs/createfs/
- https://docs.ceph.com/en/latest/cephfs/client-auth/
- https://github.com/ceph/ceph-csi/tree/devel/examples/cephfs
- https://github.com/ceph/ceph-csi/blob/devel/docs/capabilities.md


## CephFS setup


First we create a CephFS Volume, and within that a subvolume group in which the subvolumes for our PVs will exist.

```
ceph fs volume create cephfs
ceph fs subvolumegroup create cephfs nekropolis-csi
```

### Creating a user with correct permissions for csi-cephfs

Next on our list is a client user with appropriate permissions, so that the CSI-Plugin an store and modify data in CephFS.
```
ceph auth get-or-create client.nekropolis-cephfs \
  mgr 'allow rw' \
  osd 'allow rw tag cephfs metadata=cephfs, allow rw tag cephfs data=cephfs' \
  mds 'allow r fsname=cephfs path=/volumes, allow rws fsname=cephfs path=/volumes/nekropolis-csi' \
  mon 'allow r fsname=cephfs'
```

Any invocation of the `ceph auth caps ...` command will overwrite any previously set permissions not explicitly defined in the new command, hence the rather long invocation. Ensure you end up with correct permissions, getting them wrong cost me some head scratching.

```bash
ceph auth get client.nekropolis-csi-cephfs

```
