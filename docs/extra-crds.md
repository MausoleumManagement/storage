## External Snapshotter

- https://github.com/kubernetes-csi/external-snapshotter

The CSI snapshot controller [needs additional CRDs](https://kubernetes-csi.github.io/docs/snapshot-controller.html#deployment) which are not bundled with the Helm Chart. So we have a little script in this repo that fetches the necessary CRDs. This should be kept in mind during upgrades in the future.

