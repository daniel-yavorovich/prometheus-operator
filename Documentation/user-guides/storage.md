# Storage

To keep data cross deployments and version upgrades the data must be persisted to some volume other than `emptyDir`, to be able to reuse it by `Pod`s after an upgrade.

There are various kinds of volumes supported by Kubernetes. The Prometheus Operator works with `PersistentVolumeClaim`s. `PersistentVolumeClaims` are especially useful, because they support the underlying `PersistentVolume` to be provisioned when requested.

This document assumes you have a basic understanding of `PersisentVolume`s, `PersisentVolumeClaim`s, and their [provisioning](https://kubernetes.io/docs/user-guide/persistent-volumes/#provisioning).

# Storage Provisioning on AWS

For automatic provisioning of storage a `StorageClass` is required.

```yaml
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
```

> Make sure that AWS as a cloud provider is properly configured with your cluster, as otherwise storage provisioning will not work.

It is recommended to use volumes that have high I/O throughput therefore we're using SSD EBS volumes here. Make sure to read the [documentation](https://kubernetes.io/docs/user-guide/persistent-volumes/#aws) to adapt this `StorageClass` to your needs.

The `StorageClass` that was created can be specified in the `storage` section in the `Prometheus` resource.

```yaml
apiVersion: "monitoring.coreos.com/v1alpha1"
kind: "Prometheus"
metadata:
  name: "persisted"
spec:
  storage:
    class: ssd
    resources:
      requests:
        storage: 40Gi
```

> The full documentation of the `storage` field can be found in the [spec documentation](../api.md#storagespec).

When now creating the `Prometheus` object a `PersistentVolumeClaim` is used for each `Pod` in the `StatefulSet` and the storage should automatically be provisioned, mounted and used.

