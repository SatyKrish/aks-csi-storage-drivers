# AKS Storage Options Using CSI Storage Drivers

A reference implementation for using CSI Storage Drivers based persistent volumes for `stateful` workloads hosted on Azure Kubernetes Service (AKS).

## Overview

The [Container Storage Interface (CSI)](https://kubernetes-csi.github.io/docs/) is a standard for exposing arbitrary block and file storage systems to containerized workloads on Kubernetes. By adopting and using CSI, AKS can write, deploy, and iterate plug-ins to expose new or improve existing storage systems in Kubernetes without having to touch the core Kubernetes code and wait for its release cycles.

The CSI storage driver support on AKS allows you to natively use:

- **Azure Disks**, which can be used to create a Kubernetes DataDisk resource. Disks can use `Azure Premium Storage`, backed by high-performance SSDs, or `Azure Standard Storage`, backed by regular HDDs or Standard SSDs. For most production and development workloads, use Premium Storage. Azure disks are mounted as `ReadWriteOnce`, so are only available to a single pod. For storage volumes that can be accessed by multiple pods simultaneously, use `Azure Files`.
- **Azure Files**, which can be used to mount an `SMB 3.0` share backed by an `Azure Storage` account to pods. With Azure Files, you can share data across multiple nodes and pods. Azure Files can use `Azure Standard Storage` backed by regular HDDs or `Azure Premium Storage` backed by high-performance SSDs.

    ![AKS Storage Options](/docs/images/aks-storage-options.png)

## Enable CSI Storage Drivers

Starting in Kubernetes version 1.21, AKS will use `CSI drivers` only and by default as storage class. `CSI drivers` are the future of storage extension in Kubernetes. `in-tree` volume plugins are expected to be removed from Kubernetes version 1.23 onwards. After `in-tree` volume plugins are removed, existing volumes using `in-tree` volume plugins will communicate through `CSI drivers` instead.

> `in-tree` volume plugin refers to the current storage drivers that are part of the core Kubernetes code.

Following new storage classes are added to AKS for suporting CSI Storage Drivers.

| **Storage class**         | **Provisioners**          |
|:--------------------------|:--------------------------|
| default                   | disc.csi.azure.com        |
| managed-csi-premium       | disk.csi.azure.com        |
| azure-file-csi            | file.csi.azure.com        |
| azure-file-csi-premium    | file.csi.azure.com        |

When creating new AKS cluster using k8s version 1.21.x or upgrading exiting cluster k8s version from 1.20.x to 1.21.x, the `default` storage class will be the `managed-csi` storage class.

### Storage classes in Kubernetes Version 1.20.x & before

```sh
$ kubectl get storage class
```

```
OUTPUT:

NAME                    PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
azurefile               kubernetes.io/azure-file   Delete          Immediate              true                   15d
azurefile-premium       kubernetes.io/azure-file   Delete          Immediate              true                   15d
default (default)       kubernetes.io/azure-disk   Delete          WaitForFirstConsumer   true                   15d
managed-premium         kubernetes.io/azure-disk   Delete          WaitForFirstConsumer   true                   15d
```

### Storage classes in Kubernetes Version 1.21.x

```sh
$ kubectl get storage class
```

```
OUTPUT:

NAME                    PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
azurefile               kubernetes.io/azure-file   Delete          Immediate              true                   15d
azurefile-csi           file.csi.azure.com         Delete          Immediate              true                   47h
azurefile-csi-premium   file.csi.azure.com         Delete          Immediate              true                   47h
azurefile-premium       kubernetes.io/azure-file   Delete          Immediate              true                   15d
default (default)       disk.csi.azure.com         Delete          WaitForFirstConsumer   true                   47h
managed                 kubernetes.io/azure-disk   Delete          WaitForFirstConsumer   true                   47h
managed-csi-premium     disk.csi.azure.com         Delete          WaitForFirstConsumer   true                   47h
managed-premium         kubernetes.io/azure-disk   Delete          WaitForFirstConsumer   true                   15d
```

## Azure Disk CSI Driver

The Azure disk CSI driver is CSI specification compliant, and used by AKS to manage the lifecycle of Azure disks.

### Create Dynamic Persistent Volume

In this sample we will dynamically create PVs with Azure disks for use by a single pod in an AKS cluster. 

1. Review the manifest file `manifests/1-azure-disk-csi-dynamic-pv.yaml` to ensure PersistentVolumeClaim `storageClassName` is set to `managed-csi-premium`.

2. Apply the manifest.

    ```sh
    $ kubectl create namespace csi-test

    $ kubectl apply -f manifests/1-azure-disk-csi-dynamic-pv.yaml -n csi-test
    ```

    ```
    OUTPUT:

    persistentvolumeclaim/azure-disk-dynamic created
    deployment.apps/azure-disk-dynamic created
    ```

3. Check whether the resources are provisioned correctly and running.

    ```sh
    $ kubectl get pod,pvc -n csi-test -l app.kubernetes.io/name=csi-test
    ```

    ```
    OUTPUT:

    NAME                                      READY   STATUS    RESTARTS   AGE
    pod/azure-disk-dynamic-5bfbcd7b7d-pcfwj   1/1     Running   0          3m7s

    NAME                                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
    persistentvolumeclaim/azure-disk-dynamic   Bound    pvc-8512041f-3839-42d2-a0c9-8b64e2af3a54   1Gi        RWO            managed-csi-premium   3m7s
    ```

4. Verify whether a `PersistentVolume` is created and ensure that the `volumeHandle` is created as a `Disk` in AKS `MC_***` resource group.

    ```sh
    # Persistent Volume or PV is not a namespace bound resource
    $ kubectl get pv
    ```

    ```
    OUTPUT:

    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                             STORAGECLASS          REASON   AGE   
    pvc-8512041f-3839-42d2-a0c9-8b64e2af3a54   1Gi        RWO            Delete           Bound      csi-test/pvc-azure-disk-dynamic   managed-csi-premium            3m
    ```
    
    ```sh
    $ kubectl describe persistentvolume/pvc-8512041f-3839-42d2-a0c9-8b64e2af3a54
    ```

    ```
    OUTPUT:

    Name:              pvc-8512041f-3839-42d2-a0c9-8b64e2af3a54
    Labels:            <none>
    Annotations:       pv.kubernetes.io/provisioned-by: disk.csi.azure.com
    Finalizers:        [kubernetes.io/pv-protection external-attacher/disk-csi-azure-com]
    StorageClass:      managed-csi-premium
    Status:            Bound
    Claim:             csi-test/azure-disk-dynamic
    Reclaim Policy:    Delete
    Access Modes:      RWO
    VolumeMode:        Filesystem
    Capacity:          1Gi
    Node Affinity:
    Required Terms:
        Term 0:        topology.disk.csi.azure.com/zone in []
    Source:
        Type:              CSI (a Container Storage Interface (CSI) volume source)
        Driver:            disk.csi.azure.com
        FSType:
        VolumeHandle:      /subscriptions/xxxxx-xxxx-xxxxxx/resourceGroups/mc_aks-csi-test-rg_akscsitest_eastus2/providers/Microsoft.Compute/disks/pvc-911c6be4-603d-4203-88e9-4b523475c3bb
        ReadOnly:          false
        VolumeAttributes:      skuname=Premium_LRS
                            storage.kubernetes.io/csiProvisionerIdentity=1627418671510-8081-disk.csi.azure.com
    Events:                <none>
    ```

5. Test the persistent volume for read-write operation. In the test pod, persistent volume is mounted at `/data` path.

    ```sh
    $ kubectl exec -it azure-disk-dynamic-5bfbcd7b7d-swqgv -n csi-test -- sh

    / # ls
    bin   data  dev   etc   home  proc  root  sys   tmp   usr   var
    / #
    / # cd data/
    /data # ls
    lost+found
    /data # echo "Hello world AKS-CSI ! - $(date)" > csi-test
    /data # ls
    csi-test    lost+found
    /data # cat csi-test
    Hello world AKS-CSI ! - Sun, Aug  1, 2021 12:30:46 AM
    /data # exit
    ```

### Snapshot Persistent Volume

The Azure disk CSI driver supports creating snapshots of persistent volumes. As part of this capability, the driver can perform either `full` or `incremental` snapshots depending on the value set in the `incremental` parameter (by default, it's true).

For details on all the parameters, see [volume snapshot class parameters](https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/docs/driver-parameters.md#volumesnapshotclass)

1. Review the `VolumeSnapshot` in the manifest - `manifests/2-azure-disk-csi-snapshot.yaml` and ensure `persistentVolumeClaimName` maps to the `PersistenVolumeClaim` created in previous section.

2. Apply the manifest.

    ```sh
    $ kubectl apply -f manifests/2-azure-disk-csi-snapshot.yaml
    ```

    ```
    OUTPUT:

    volumesnapshotclass.snapshot.storage.k8s.io/vsc-azure-disk-dynamic created
    volumesnapshot.snapshot.storage.k8s.io/snapshot-azure-disk-dynamic created
    ```
3. Check whether the `VolumeSnapshot` resources are provisioned correctly.

    ```sh
    $ kubectl get volumesnapshotclass,volumesnapshot,volumesnapshotcontent -n csi-test
    ```

    ```
    OUTPUT:

    NAME                                                                 DRIVER               DELETIONPOLICY   AGE
    volumesnapshotclass.snapshot.storage.k8s.io/vsc-azure-disk-dynamic   disk.csi.azure.com   Delete           34m

    NAME                                                                 READYTOUSE   SOURCEPVC                SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS            SNAPSHOTCONTENT                                   CREATIONTIME    AGE
    volumesnapshot.snapshot.storage.k8s.io/snapshot-azure-disk-dynamic   true         pvc-azure-disk-dynamic                           1Gi           vsc-azure-disk-dynamic   snapcontent-ed014617-5396-47f4-9c87-06eebe16d8bb   34m            34m

    NAME                                                                                             READYTOUSE   RESTORESIZE   DELETIONPOLICY   DRIVER               VOLUMESNAPSHOTCLASS       VOLUMESNAPSHOT                AGE
    volumesnapshotcontent.snapshot.storage.k8s.io/snapcontent-ed014617-5396-47f4-9c87-06eebe16d8bb   true         1073741824    Delete           disk.csi.azure.com   vsc-azure-disk-dynamic        snapshot-azure-disk-dynamic   34m
    ```

4. Verify the `VolumeSnapshotContent` resource to ensure that the `volumeHandle` matches the  Managed Disk created in previous section, `snapshotHandle` is created as a `Snapshot` in AKS `MC_***` resource group and `Status` has `Ready To Use` set to `true`.

    ```sh
    $ kubectl describe VolumeSnapshotContent snapcontent-ed014617-5396-47f4-9c87-06eebe16d8bb -n csi-test
    ```

    ```
    OUTPUT:

    Name:         snapcontent-ed014617-5396-47f4-9c87-06eebe16d8bb
    Namespace:
    Labels:       <none>
    Annotations:  <none>
    API Version:  snapshot.storage.k8s.io/v1beta1
    Kind:         VolumeSnapshotContent
    Metadata:
    Creation Timestamp:  2021-07-31T10:59:05Z
    Finalizers:
        snapshot.storage.kubernetes.io/volumesnapshotcontent-bound-protection
    Generation:  1
    Resource Version:  3743314
    UID:               53d8bd12-f691-4b38-b238-53b3193df30b
    Spec:
    Deletion Policy:  Delete
    Driver:           disk.csi.azure.com
    Source:
        Volume Handle:             /subscriptions/xxxxx-xxxx-xxxxxx/resourceGroups/mc_aks-csi-test-rg_akscsitest_eastus2/providers/Microsoft.Compute/disksb523475c3bb
    Volume Snapshot Class Name:  vsc-azure-disk-dynamic
    Volume Snapshot Ref:
        API Version:       snapshot.storage.k8s.io/v1beta1
        Kind:              VolumeSnapshot
        Name:              snapshot-azure-disk-dynamic
        Namespace:         csi-test
        Resource Version:  3743273
        UID:               ed014617-5396-47f4-9c87-06eebe16d8bb
    Status:
    Creation Time:    1627729149921916700
    Ready To Use:     true
    Restore Size:     1073741824
    Snapshot Handle:  /subscriptions/xxxxx-xxxx-xxxxxx/resourceGroups/mc_aks-csi-test-rg_akscsitest_eastus2/providers/Microsoft.Compute/snapshots/snapsh   CLAIM                             STORAGECLASS          REASON   AGE
    azure-file-static                          1Gi        RWX            Retain           Released   csi-test/azure-file-static                                       2d17h
    pvc-911c6be4-603d-4203-88e9-4b523475c3bb   1Gi        RWO            Delete           Bound      csi-test/pvc-azure-disk-dynamic   managed-csi-premium            28m
    ```

### Restore Volume Snapshot

During a failure event, backed up data can be restored to a Managed Disk from the Snapshot created in the pervious section. This done by creating a `PersistentVolumeClaim` resource based on an existing `VolumeSnapshot`. CSI provisioner will then create a new `PersistentVolume` from the snapshot.

1. Lets simulate failure by deleting the pods and persistent volume created earlier.
    ```sh
    $ kubectl delete -f manifests/1-azure-disk-csi-dynamic-pv.yaml -n csi-test
    ```
    
    ```
    OUTPUT:
    
    persistentvolumeclaim/azure-disk-dynamic deleted
    deployment.apps/azure-disk-dynamic deleted
    ```
2. Ensure the `PersistentVolume` and `Disk` in AKS `MC_***` resource group are deleted.
3. Review the `PersistenVolumeClaim` in the manifest - `manifests/3-azure-disk-csi-restore.yaml` and ensure `dataSource` maps to the `VolumeSnapshot` created in previous section. 

2. Apply the manifest.

    ```sh
    $ kubectl apply -f manifests/3-azure-disk-csi-restore.yaml -n csi-test
    ```

    ```
    OUTPUT:

    persistentvolumeclaim/pvc-azure-disk-dynamic-restored created
    deployment.apps/azure-disk-dynamic created
    ```

3. Check whether the resources are provisioned correctly and running.

    ```sh
    $ kubectl get pod,pvc -n csi-test -l app.kubernetes.io/name=csi-test
    ```

    ```
    OUTPUT:

    NAME                                      READY   STATUS    RESTARTS   AGE
    pod/azure-disk-dynamic-77f885d64b-hp5kf   1/1     Running   0          50m

    NAME                                                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
    persistentvolumeclaim/pvc-azure-disk-dynamic-restored   Bound    pvc-3b5c8f03-6307-4712-8774-573fe4a35e5e   1Gi        RWO            managed-csi-premium   50m
    ```

4. Verify whether a `PersistentVolume` is created and ensure that the `volumeHandle` is created as a `Disk` in AKS `MC_***` resource group.

    ```sh
    # Persistent Volume or PV is not a namespace bound resource
    $ kubectl get pv
    ```

    ```
    OUTPUT:

    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                                          STORAGECLASS          REASON   AGE   
    pvc-3b5c8f03-6307-4712-8774-573fe4a35e5e   1Gi        RWO            Delete           Bound      csi-test/pvc-azure-disk-dynamic-restored   managed-csi-premium            71m
    ```
    
    ```sh
    $ kubectl describe persistentvolume/pvc-3b5c8f03-6307-4712-8774-573fe4a35e5e
    ```

    ```
    OUTPUT:

    Name:              pvc-3b5c8f03-6307-4712-8774-573fe4a35e5e
    Labels:            <none>
    Annotations:       pv.kubernetes.io/provisioned-by: disk.csi.azure.com
    Finalizers:        [kubernetes.io/pv-protection external-attacher/disk-csi-azure-com]
    StorageClass:      managed-csi-premium
    Status:            Bound
    Claim:             csi-test/pvc-azure-disk-dynamic-restored
    Reclaim Policy:    Delete
    Access Modes:      RWO
    VolumeMode:        Filesystem
    Capacity:          1Gi
    Node Affinity:
    Required Terms:
        Term 0:        topology.disk.csi.azure.com/zone in []
    Message:
    Source:
        Type:              CSI (a Container Storage Interface (CSI) volume source)
        Driver:            disk.csi.azure.com
        FSType:
        VolumeHandle:      /subscriptions/xxxxx-xxxx-xxxxxx/resourceGroups/mc_aks-csi-test-rg_akscsitest_eastus2/providers/Microsoft.Compute/disks/pvc-3b5c8f03-6307-4712-8774-573fe4a35e5e
        ReadOnly:          false
        VolumeAttributes:  skuname=Premium_LRS
                           storage.kubernetes.io/csiProvisionerIdentity=1627726829005-8081-disk.csi.azure.com
    Events:                <none>
    ```

5. Check whether the data is restored. In the test pod, persistent volume is mounted at `/data` path.

    ```sh
    $ kubectl exec -it azure-disk-dynamic-77f885d64b-hp5kf -n csi-test -- sh

    / # ls
    bin   data  dev   etc   home  proc  root  sys   tmp   usr   var
    / #
    / # cd data/
    /data # ls
    csi-test    lost+found
    /data # cat csi-test
    Hello world AKS-CSI ! - Sun, Aug  1, 2021 12:30:46 AM
    /data # exit
    ```

## Azure Files CSI Driver

The Azure Files CSI driver is CSI specification compliant, and used by AKS to manage the lifecycle of Azure file shares.

### Create Static Persistent Volume

In this sample we will statically create `PersistentVolume` with an existing Azure Files share for use by multiple pods in an AKS cluster.

1. Create a Kubernetes Secret with Azure Storage Account Name and Storage Key.

    ```sh
    $ STORAGE_ACCOUNT_NAME=<Provide Azure Storage Account Name Here>
    $ STORAGE_KEY=<Provide Azure Storage Account Key Here>
    $ kubectl create secret generic azure-secret -n csi-test \
        --from-literal=azurestorageaccountname=$STORAGE_ACCOUNT_NAME \
        --from-literal=azurestorageaccountkey=$STORAGE_KEY
    ```

2. Review the manifest file `manifests/4-azure-files-csi-static-pv.yaml` to ensure PersistentVolume has `csi` section with `driver` as `file.csi.azure.com`.

3. Apply the manifest.

    ```sh
    $ kubectl apply -f manifests/4-azure-files-csi-static-pv.yaml -n csi-test
    ```

    ```
    OUTPUT:

    persistentvolume/azure-file-static created
    persistentvolumeclaim/azure-file-static created
    deployment.apps/1-azure-file-static created
    deployment.apps/2-azure-file-static created
    ```

4. Check whether the resources are provisioned correctly and running.

    ```sh
    $ kubectl get pod,pv,pvc -n csi-test -l app.kubernetes.io/name=csi-test
    ```

    ```
    OUTPUT:

    NAME                                       READY   STATUS    RESTARTS   AGE
    pod/1-azure-file-static-65885756b4-6mhnc   1/1     Running   0          3m23s
    pod/2-azure-file-static-65885756b4-zd6hm   1/1     Running   0          3m22s

    NAME                                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM      STORAGECLASS       REASON   AGE
    persistentvolume/azure-file-static   1Gi        RWX            Retain           Bound    csi-test   azure-file-static           3m24s

    NAME                                       STATUS   VOLUME                      CAPACITY   ACCESS MODES  STORAGECLASS           AGE
    persistentvolumeclaim/azure-file-static    Bound    azure-file-static           1Gi        RWX                                  3m23s
    ```

5. Verify that persistent volume type is `CSI` and driver is `file.csi.azure.com`. Other properties should match the manifest file.

    ```sh
    $ kubectl describe persistentvolume/azure-file-static -n csi-test
    ```

    ```
    OUTPUT:

    Name:            azure-file-static
    Labels:          app.kubernetes.io/name=csi-test
    Annotations:     pv.kubernetes.io/bound-by-controller: yes
    Finalizers:      [kubernetes.io/pv-protection external-attacher/file-csi-azure-com]
    StorageClass:
    Status:          Bound
    Claim:           csi-test/azure-file-static
    Reclaim Policy:  Retain
    Access Modes:    RWX
    VolumeMode:      Filesystem
    Capacity:        1Gi
    Node Affinity:   <none>
    Message:
    Source:
        Type:              CSI (a Container Storage Interface (CSI) volume source)
        Driver:            file.csi.azure.com
        FSType:
        VolumeHandle:      csi-test-10922
        ReadOnly:          false
        VolumeAttributes:      resourceGroup=aks-sandbox-dev
                            shareName=aks-test
    Events:                <none>
    ```

6. Test the persistent volume for read-write operation on 1st pod. Persistent volume is mounted at `/data` path.

    ```sh
    $ kubectl exec -it 1-azure-file-static-65885756b4-6mhnc -n csi-test -- sh

    / # ls
    bin        csi-test1  data       dev        etc        home       proc       root       sys        tmp        usr        var
    / # cd data/
    /data # echo "Hello world AKS-CSI from Pod 1 !" > csi-test1
    /data # ls
    csi-test1
    /data # cat csi-test1
    Hello world AKS-CSI from Pod 1 !
    /data # exit
    ```

6. Test the persistent volume for read-write operation on 2nd pod. Persistent volume is mounted at `/data` path.

    ```sh
    $ kubectl exec -it 2-azure-file-static-65885756b4-zd6hm -n csi-test -- sh

    / # ls
    bin   data  dev   etc   home  proc  root  sys   tmp   usr   var
    / # cd data/
    /data # ls
    csi-test1
    /data # cat csi-test1
    Hello world AKS-CSI from Pod 1 !
    /data # echo "Hello world AKS-CSI from Pod 2 !" > csi-test2
    /data # ls
    csi-test1  csi-test2
    /data # cat csi-test2
    Hello world AKS-CSI from Pod 2 !
    /data # exit
    ```

## Troubleshooting

- For `permission denied` error while mounting Azure Files share, refer to the troubleshooting instructions [here](https://docs.microsoft.com/en-us/azure/storage/files/storage-troubleshoot-linux-file-connection-problems#mount-error13-permission-denied-when-you-mount-an-azure-file-share)

## Clean-up

Delete the resources created in `csi-test` namespace. 

```sh
$ kubectl delete ns csi-test
```

## Reference

- [Kubernetes Container Storage Interface (CSI) Documentation](https://kubernetes-csi.github.io/docs/)
- [YouTube: Introduction to Using the Container Storage Interface (CSI) Primitives - Michael Mattsson](https://youtu.be/AnfAd6goq-o)
- [GitHub: Azure Disk CSI driver for Kubernetes](https://github.com/kubernetes-sigs/azuredisk-csi-driver)
- [GitHub: Azure File CSI Driver for Kubernetes](https://github.com/kubernetes-sigs/azurefile-csi-driver)
