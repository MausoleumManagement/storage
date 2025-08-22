# Storage

## Reading Material
- https://docs.ceph.com/en/reef/cephfs/createfs/
- https://docs.ceph.com/en/latest/cephfs/client-auth/
- https://github.com/ceph/ceph-csi/tree/devel/examples/cephfs
- https://github.com/ceph/ceph-csi/blob/devel/docs/capabilities.md

This repo contains the deployments for various Container Storage Interfaces (CSIs). Their job is to make the various volumes we create as kubernetes resources actually appear in the respective backends.

CSI plugins can either be in-tree (i.e. in the K8s source code) or out-of-tree (i.e. outside of the code and usually deployed using Kubernetes resources).
See [here](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes) for a list of supported in-tree CSI plugins as well as deprecated plugins which will need to be migrated in the future!
This is worth taking a look at in future releases, because we will need to migrate away from deprecated in-tree plugins and deploy the relevant out-of-tree plugin before we roll out the update that completely removes the in-tree version.

There are many components that cooperate, to make each csi-plugin work. Other than the plugin itself, which handles the communication to the storage backend, there are multiple components responsible for the kubernetes side of things. So when we create, resize, create a snapshot of, attach, ..., a volume, they are going to talk to the csi-plugin using a standardised interface, and then the plugin can do its idiosyncratic backend magic, which Kubernetes does not know about. The generic componentes like the attacher, provisioner, and so on, are provided by SIG-Storage. A storage backend maintainer only needs to implement the specific csi-plugin with the correct interface.

Here's a figure displaying this. It's from the [Openshift documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/storage/using-container-storage-interface-csi#persistent-storage-csi-architecture_persistent-storage-csi), but it's close enough:

![image](docs/images/csi_architecture.png)

The Openshift documentation (owned by Red Hat) is licensed under the [Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0).

## CSI-Components
### ceph-csi
Container Storage Interface for Ceph.
### snapshot-controller
Creates snapshots when users create `VolumeSnapshot` objects. The contents of those snapshots live in `VolumeSnapshotContent` objects, like PVs and PVCs.


## CSI Generic Components
Each CSI has these, usually. See [here](https://github.com/kubernetes-csi) for their repos. Here's what each one does:
### attacher
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

