## cephfs-csi Setup

List RADOS pools in our ceph cluster
```
rados lspools
rados ls --pool ceph_ssd_replicated
```

### RADOS pool namespaces
By default, objects are created in the default namespace, which is what you get when you do not explicitly state one.
The first command will use the default namespace.

```bash
rados ls --pool ceph_ssd_replicated
rados ls --pool ceph_ssd_replicated --namespace abc
```

All data in Ceph is stored as RADOS objects, this includes the data (and metadata) from our CephFS volumes.

We will use the namespace `nekropolis-ceph-csi-cephfs` for our ceph-csi-cephfs objects. It's not something we need to create beforehand.


## Creating CephFS Volumes (Filesystems)

It is recommended to store the CephFS data and metadata in separate pools each.

```bash
ceph osd pool create cephfs_data 128
ceph osd pool create cephfs_metadata 32
# create a new cephfs filesystem called cephfs
# with the last two arguments being the metadata and data pools backing it
ceph fs new cephfs cephfs_metadata cephfs_data
```

The ceph tool suggested this, it should cause the pool to have its full complement of PGs immediately. Sounded okay for a pool that will store all our data, so let's try it out. See [the docs](https://docs.ceph.com/en/reef/rados/operations/placement-groups/#managing-pools-that-are-flagged-with-bulk)
```
ceph osd pool set cephfs_data bulk true
```

We create a subvolume group in which the subvolumes for our PVs will be created
```
ceph fs subvolumegroup create cephfs k8s
```


### CephFS Subvolumes

Each CephFS volume requires at least one dedicated Metadata Server ([see official docs](https://docs.ceph.com/en/reef/cephfs/add-remove-mds/#deploying-metadata-servers)). This would get out of hand if we created a new CephFS volume for each Persistent Volume in our Kubernetes cluster. Instead the Ceph people came up with subvolumes.

A CephFS subvolume is a subdivision of a volume's filesystem, which can be handed out to clients 

```
ceph fs subvolume create cephfs nekropolis-csi-cephfs --group_name k8s --namespace-isolated
```

Now we can check whether our subvolume was created correctly and retrieve the RADOS namespace used by the subvolume.
```
ceph fs subvolume info cephfs nekropolis-csi-cephfs --group_name k8s
```


### Creating a user with correct permissions for csi-cephfs

https://github.com/ceph/ceph-csi/blob/devel/docs/capabilities.md


**OUR MODIFIED VERSION**
```
USER=nekropolis-csi-cephfs
FS_NAME=cephfs
SUB_VOL=nekropolis-csi-cephfs
NAMESPACE=fsvolumens_nekropolis-csi-cephfs

ceph auth get-or-create client.$USER
ceph auth caps client.$USER mgr "allow rw"
ceph auth caps client.$USER osd "allow rwx pool=${FS_NAME}_metadata namespace=${NAMESPACE}"
ceph auth caps client.$USER osd "allow rw pool=${FS_NAME}_data namespace=${NAMESPACE}"
ceph auth caps client.$USER mds "allow r fsname=${FS_NAME} path=/volumes/nekropolis-csi-cephfs"
ceph auth caps client.$USER mds "allow rws fsname=${FS_NAME} path=/volumes/nekropolis-csi-cephfs/${SUB_VOL}"
ceph auth caps client.$USER mon "allow r"
```

**ORIGINAL DOCS VERSION**
```
USER=-csi-cephfs-nekropolis
FS_NAME=cephfs
SUB_VOL=k8s
ceph auth get-or-create client.$USER \
  mgr "allow rw" \
  osd "allow rwx tag cephfs metadata=$FS_NAME, allow rw tag cephfs data=$FS_NAME" \
  mds "allow r fsname=$FS_NAME path=/volumes, allow rws fsname=$FS_NAME path=/volumes/$SUB_VOL" \
  mon "allow r fsname=$FS_NAME"
```

### Side notes

List RADOS block devices in a pool
```
rbd ls --pool ceph_ssd_replicated
```

```bash
rados ls --pool ceph_ssd_replicated | sort | less
```

```bash
ceph fs volume ls
```

```
ceph fs subvolumegroup ls cephfs
```

```
ceph fs subvolumegroup info cephfs cephfs-csi-nekropolis
{
    "atime": "2025-08-03 19:46:02",
    "bytes_pcent": "undefined",
    "bytes_quota": "infinite",
    "bytes_used": 281,
    "created_at": "2025-08-03 19:46:02",
    "ctime": "2025-08-03 20:14:05",
    "data_pool": "cephfs_data",
    "gid": 0,
    "mode": 16877,
    "mon_addrs": [
        "192.168.9.10:6789",
        "192.168.9.20:6789",
        "192.168.9.11:6789"
    ],
    "mtime": "2025-08-03 20:14:05",
    "uid": 0
}
```

Get info, including namespace on subvolume
```
ceph fs subvolume info cephfs k8s --group_name cephfs-csi-nekropolis
```

Here's the subvolume that was created with --namespace-isolation
```
ceph fs subvolume info cephfs k8s --group_name cephfs-csi-nekropolis
{
    "atime": "2025-08-03 19:46:19",
    "bytes_pcent": "undefined",
    "bytes_quota": "infinite",
    "bytes_used": 0,
    "created_at": "2025-08-03 19:46:19",
    "ctime": "2025-08-03 19:46:19",
    "data_pool": "cephfs_data",
    "features": [
        "snapshot-clone",
        "snapshot-autoprotect",
        "snapshot-retention"
    ],
    "flavor": 2,
    "gid": 0,
    "mode": 16877,
    "mon_addrs": [
        "192.168.9.10:6789",
        "192.168.9.20:6789",
        "192.168.9.11:6789"
    ],
    "mtime": "2025-08-03 19:46:19",
    "path": "/volumes/cephfs-csi-nekropolis/k8s/a88fb64e-dc88-4ad9-a031-5b25e3bde9b6",
    "pool_namespace": "fsvolumens_k8s",
    "state": "complete",
    "type": "subvolume",
    "uid": 0
}
```

And here's the one that was created without. Note the empty `pool_namespace` value.
```
ceph fs subvolume info cephfs k8s-unisolated --group_name cephfs-csi-nekropolis
{
    "atime": "2025-08-03 20:14:05",
    "bytes_pcent": "undefined",
    "bytes_quota": "infinite",
    "bytes_used": 0,
    "created_at": "2025-08-03 20:14:05",
    "ctime": "2025-08-03 20:14:05",
    "data_pool": "cephfs_data",
    "features": [
        "snapshot-clone",
        "snapshot-autoprotect",
        "snapshot-retention"
    ],
    "flavor": 2,
    "gid": 0,
    "mode": 16877,
    "mon_addrs": [
        "192.168.9.10:6789",
        "192.168.9.20:6789",
        "192.168.9.11:6789"
    ],
    "mtime": "2025-08-03 20:14:05",
    "path": "/volumes/cephfs-csi-nekropolis/k8s-unisolated/455ae6ee-e9a1-4e84-b1c6-ba9605df3ba3",
    "pool_namespace": "",
    "state": "complete",
    "type": "subvolume",
    "uid": 0
}
```

## Storage
This repo contains the deployments for various Container Storage Interfaces (CSIs). Their job is to make the various volumes we create as kubernetes resources actually appear in the respective backends.

CSI plugins can either be in-tree (i.e. in the K8s source code) or out-of-tree (i.e. outside of the code and usually deployed using Kubernetes resources).
See [here](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes) for a list of supported in-tree CSI plugins as well as deprecated plugins which will need to be migrated in the future!
This is worth taking a look at in future releases, because we will need to migrate away from deprecated in-tree plugins and deploy the relevant out-of-tree plugin before we roll out the update that completely removes the in-tree version.

There are many components that cooperate, to make each csi-plugin work. Other than the plugin itself, which handles the communication to the storage backend, there are multiple components responsible for the kubernetes side of things. So when we create, resize, create a snapshot of, attach, ..., a volume, they are going to talk to the csi-plugin using a standardised interface, and then the plugin can do its idiosyncratic backend magic, which Kubernetes does not know about. The generic componentes like the attacher, provisioner, and so on, are provided by SIG-Storage. A storage backend maintainer only needs to implement the specific csi-plugin with the correct interface.

## TODO: Legal Stuff
- Openshift Docs are under the apache license
- https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/storage/legal-notice

Here's a figure displaying this. It's from the [Openshift documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/storage/using-container-storage-interface-csi#persistent-storage-csi-architecture_persistent-storage-csi), but it's close enough:

![image](docs/images/csi_architecture.png)


## Components
### OpenStack CSIs
See [here](https://github.com/kubernetes/cloud-provider-openstack) for the repo of these components.
- **cinder-csi**
  - Container Storage Interface for Cinder.
- **manila-csi**
  - Container Storage Interface for Manila.
#### ceph-csi
Container Storage Interface for Ceph.
#### snapshot-controller
Creates snapshots when users create `VolumeSnapshot` objects. The contents of those snapshots live in `VolumeSnapshotContent` objects, like PVs and PVCs.


## CSI Generic Components
Each CSI has these, usually. See [here](https://github.com/kubernetes-csi) for their repos. Here's what each one does:
#### attacher
Responsible for attaching and detaching external volumes to nodes.
#### provisioner
Responsible for creating volumes in the storage backend when users create PVCs.
#### snapshotter
Contains the CSI specific logic for creating snapshots
#### resizer
Resizes PVs when users modify the respective PVC.
#### registrar
This thing registers the CSI driver with kubelet because microservices.
#### livenessprobe
This thing serves the `/healthz` endpoint of the CSI driver. Because microservices, that's why.

## CSI Specific Components
#### plugin
The actual plugin offering the functionality for the respective backend.

