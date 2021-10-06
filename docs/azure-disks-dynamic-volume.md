# Azure Disks Dynamic Persistent Volume

The Azure disk CSI driver is CSI specification compliant, and used by AKS to manage the lifecycle of Azure disks attached to the pod as `PersistentVolume`.

In this sample we will dynamically create `PersistentVolume` with Azure disks for use by a single pod in AKS cluster. Also demonstrate backup and restore persistent volume data using CSI driver. 

## Setup

1. Set environment defaults.

    ```sh
    SUBSCRIPTION_ID=<my-subsscription-id>
    RESOURCE_GROUP=<my-aks-rg>
    LOCATION=eastus2
    CLUSTER_NAME=<my-aks-cluster>
    ```

2. Create an AKS cluster with kubernetes version 1.21, CNI network plugin and managed identity enabled. 

    ```sh
    az group create \
        --name $RESOURCE_GROUP \
        --location $LOCATION
    
    az aks create \
        --resource-group $RESOURCE_GROUP \
        --name $CLUSTER_NAME \
        --enable-managed-identity \
        --network-plugin azure \
        --kubernetes-version 1.21.2
    ```

3. Generate kubeconfig file for connecting to AKS cluster.

    ```sh
    az aks get-credentials \
        --resource-group $RESOURCE_GROUP \
        --name $CLUSTER_NAME \
        --admin
    ```

## Create Dynamic Persistent Volume

1. Review the manifest file `manifests/1-azure-disk-csi-dynamic.yaml` to ensure PersistentVolumeClaim `storageClassName` is set to `managed-csi-premium`.

2. Apply the manifest.

    ```sh
    kubectl create namespace disks-csi-test

    kubectl apply -f manifests/1-azure-disk-csi-dynamic.yaml -n csi-test
    ```

    ```
    OUTPUT:

    persistentvolumeclaim/azure-disk-dynamic created
    deployment.apps/azure-disk-dynamic created
    ```

3. Check whether the resources are provisioned correctly and running.

    ```sh
    kubectl get pod,pvc -n csi-test -l app.kubernetes.io/name=csi-test
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
    kubectl get pv
    ```

    ```
    OUTPUT:

    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                             STORAGECLASS          REASON   AGE   
    pvc-8512041f-3839-42d2-a0c9-8b64e2af3a54   1Gi        RWO            Delete           Bound      csi-test/pvc-azure-disk-dynamic   managed-csi-premium            3m
    ```
    
    ```sh
    kubectl describe persistentvolume/pvc-8512041f-3839-42d2-a0c9-8b64e2af3a54
    ```

5. Test the persistent volume for read-write operation. In the test pod, persistent volume is mounted at `/data` path.

    ```sh
    kubectl exec -it azure-disk-dynamic-5bfbcd7b7d-swqgv -n csi-test -- sh
    ```

    ```sh
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

## Snapshot Persistent Volume

The Azure disk CSI driver supports creating snapshots of persistent volumes. As part of this capability, the driver can perform either `full` or `incremental` snapshots depending on the value set in the `incremental` parameter (by default, it's true).

For details on all the parameters, see [volume snapshot class parameters](https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/docs/driver-parameters.md#volumesnapshotclass)

1. Review the `VolumeSnapshot` in the manifest - `manifests/2-azure-disk-csi-snapshot.yaml` and ensure `persistentVolumeClaimName` maps to the `PersistenVolumeClaim` created in previous section.

2. Apply the manifest.

    ```sh
    kubectl apply -f manifests/2-azure-disk-csi-snapshot.yaml -n csi-test
    ```

    ```
    OUTPUT:

    volumesnapshotclass.snapshot.storage.k8s.io/vsc-azure-disk-dynamic created
    volumesnapshot.snapshot.storage.k8s.io/snapshot-azure-disk-dynamic created
    ```
3. Check whether the `VolumeSnapshot` resources are provisioned correctly.

    ```sh
    kubectl get volumesnapshotclass,volumesnapshot,volumesnapshotcontent -n csi-test
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
    kubectl describe VolumeSnapshotContent snapcontent-ed014617-5396-47f4-9c87-06eebe16d8bb -n csi-test
    ```

## Restore Volume Snapshot

During DR, backed up data can be restored to a Managed Disk from the Snapshot created in the pervious section. This done by creating a `PersistentVolumeClaim` resource based on an existing `VolumeSnapshot`. CSI provisioner will then create a new `PersistentVolume` from the snapshot.

1. Lets simulate failure by deleting the pods and persistent volume created earlier.
    ```sh
    kubectl delete -f manifests/1-azure-disk-csi-dynamic.yaml -n csi-test
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
    kubectl apply -f manifests/3-azure-disk-csi-restore.yaml -n csi-test
    ```

    ```
    OUTPUT:

    persistentvolumeclaim/pvc-azure-disk-dynamic-restored created
    deployment.apps/azure-disk-dynamic created
    ```

3. Check whether the resources are provisioned correctly and running.

    ```sh
    kubectl get pod,pvc -n csi-test -l app.kubernetes.io/name=csi-test
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
    kubectl get pv
    ```

    ```
    OUTPUT:

    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                                          STORAGECLASS          REASON   AGE   
    pvc-3b5c8f03-6307-4712-8774-573fe4a35e5e   1Gi        RWO            Delete           Bound      csi-test/pvc-azure-disk-dynamic-restored   managed-csi-premium            71m
    ```
    
    ```sh
    kubectl describe persistentvolume/pvc-3b5c8f03-6307-4712-8774-573fe4a35e5e
    ```

5. Check whether the data is restored. In the test pod, persistent volume is mounted at `/data` path.

    ```sh
    kubectl exec -it azure-disk-dynamic-77f885d64b-hp5kf -n csi-test -- sh

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

## Migrate `in-tree` volume to `CSI` volume

Azure Disk CSI migration is turned on for 1.21+ clusters. After `in-tree` volume plugins are removed, existing volumes using `in-tree` volume plugins will communicate through `CSI drivers` instead.

In this sample we will use `in-tree` volume storage class to dynamically create `PersistentVolume` with Azure disks for use by a single pod in AKS cluster. 

1. Review the manifest file `manifests/4-azure-disk-csi-migrate.yaml` to ensure PersistentVolumeClaim `storageClassName` is set to `managed-premium`.

2. Apply the manifest.

    ```sh
    kubectl create namespace csi-test

    kubectl apply -f manifests/4-azure-disk-csi-migrate.yaml -n csi-test
    ```

    ```
    OUTPUT:

    persistentvolumeclaim/pvc-azure-intree-disk-dynamic created
    deployment.apps/azure-intree-disk-dynamic create
    ```

3. Check whether the resources are provisioned correctly and running.

    ```sh
    kubectl get pod,pvc -n csi-test -l app.kubernetes.io/name=csi-test
    ```

    ```
    OUTPUT:

    NAME                                             READY   STATUS    RESTARTS   AGE
    pod/azure-intree-disk-dynamic-5bfbcd7b7d-pcfwj   1/1     Running   0          3m7s

    NAME                                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
    persistentvolumeclaim/azure-intree-disk-dynamic   Bound    pvc-8512041f-3839-42d2-a0c9-8b64e2af3a54   1Gi        RWO            managed-csi-premium   3m7s
    ```

4. Verify whether a `PersistentVolume` is created and ensure that the `volumeHandle` is created as a `Disk` in AKS `MC_***` resource group.

    ```sh
    # Persistent Volume or PV is not a namespace bound resource
    kubectl get pv
    ```

    ```
    OUTPUT:

    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                                    STORAGECLASS          REASON   AGE   
    pvc-8512041f-3839-42d2-a0c9-8b64e2af3a54   1Gi        RWO            Delete           Bound      csi-test/pvc-azure-intree-disk-dynamic   managed-csi-premium            3m
    ```

4. Test the persistent volume for read-write operation. In the test pod, persistent volume is mounted at `/data` path.

    ```sh
    kubectl exec -it azure-intree-disk-dynamic-5bfbcd7b7d-swqgv -n csi-test -- sh
    ```

    ```sh
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

## Clean-up

Delete the resources created in `csi-test` namespace. 

```sh
kubectl delete namespace csi-test
```

Delete resource group

```sh
az group delete --name $RESOURCE_GROUP
```