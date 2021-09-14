# Azure Files CSI Driver

The Azure Files CSI driver is CSI specification compliant, and used by AKS to manage the lifecycle of Azure file shares attached to pod as `PersistentVolume`.

## Create Static Persistent Volume

In this sample we will statically create `PersistentVolume` with an existing Azure Files share for use by multiple pods in an AKS cluster.

1. Create a Kubernetes Secret with Azure Storage Account Name and Storage Key.

    ```sh
    ```sh
    kubectl create namespace files-csi-test

    $ STORAGE_ACCOUNT_NAME=<Provide Azure Storage Account Name Here>
    $ STORAGE_KEY=<Provide Azure Storage Account Key Here>
    kubectl create secret generic azure-secret -n files-csi-test \
        --from-literal=azurestorageaccountname=$STORAGE_ACCOUNT_NAME \
        --from-literal=azurestorageaccountkey=$STORAGE_KEY
    ```

2. Review the manifest file `manifests/5-azure-files-csi-static-pv.yaml` to ensure PersistentVolume has `csi` section with `driver` as `file.csi.azure.com`.

3. Apply the manifest.

    ```sh
    kubectl apply -f manifests/5-azure-files-csi-static-pv.yaml -n files-csi-test
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
    kubectl get pod,pv,pvc -n files-csi-test -l app.kubernetes.io/name=csi-test
    ```

    ```
    OUTPUT:

    NAME                                       READY   STATUS    RESTARTS   AGE
    pod/1-azure-file-static-65885756b4-6mhnc   1/1     Running   0          3m23s
    pod/2-azure-file-static-65885756b4-zd6hm   1/1     Running   0          3m22s

    NAME                                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS       REASON   AGE
    persistentvolume/azure-file-static   1Gi        RWX            Retain           Bound    csi-test   azure-file-static           3m24s

    NAME                                       STATUS   VOLUME                      CAPACITY   ACCESS MODES  STORAGECLASS           AGE
    persistentvolumeclaim/azure-file-static    Bound    azure-file-static           1Gi        RWX                                  3m23s
    ```

5. Verify that persistent volume type is `CSI` and driver is `file.csi.azure.com`. Other properties should match the manifest file.

    ```sh
    kubectl describe persistentvolume/azure-file-static -n files-csi-test
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
    kubectl exec -it 1-azure-file-static-65885756b4-6mhnc -n files-csi-test -- sh

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
    kubectl exec -it 2-azure-file-static-65885756b4-zd6hm -n files-csi-test -- sh

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

Delete the resources created in `files-csi-test` namespace. 

```sh
kubectl delete namespace files-csi-test
```