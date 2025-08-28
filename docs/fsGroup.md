# Reading Material

- https://kubernetes.io/blog/2020/12/14/kubernetes-release-1.20-fsgroupchangepolicy-fsgrouppolicy/
- https://kubernetes.io/blog/2022/12/23/kubernetes-12-06-fsgroup-on-mount/
- https://kubernetes-csi.github.io/docs/support-fsgroup.html
- https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/csi-driver-v1/#CSIDriverSpec

The `fsGroup` setting in a `securityContext` will cause kubernetes to attempt to modify permissions on a mounted volume, so that the group of the files contained within the volume matches that of the `fsGroup` value. Apparently, only the topmost directory is checked, and anything else is recursively modified based on the result of that check. This behaviour can be influenced by the `fsGroupChangePolicy` value, but to fully deacivate this functionality, the `fsGroup` value needs to be unset.
