# Azure Files Static Persistent Volume using  Storage Key

The Azure Files CSI driver is CSI specification compliant, and used by AKS to manage the lifecycle of Azure file shares attached to pod as `PersistentVolume`.

In this sample we will statically create `PersistentVolume` with an existing Azure Files share using storage key. Kubernetes needs storage key to access the file share, and this cretdential is stored in a Kubernetes Secret. 

## Setup

1. Set environment defaults.

    ```sh
    SUBSCRIPTION_ID=<my-subsscription-id>
    RESOURCE_GROUP=<my-aks-rg>
    LOCATION=eastus2
    CLUSTER_NAME=<my-aks-cluster>
    STORAGE_ACCOUNT=<my-storage-account>
    FILE_SHARE=<my-file-share>
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

4. Before you can use Azure Files as a Kubernetes volume, you must create an Azure Storage account and the file share. The following commands create a storage account and a files share 

    ```sh
    az storage account create \
        --resource-group $RESOURCE_GROUP \
        --name $STORAGE_ACCOUNT \
        --location $LOCATION \
        --encryption-services file

    az storage share update  \
        --name $FILE_SHARE \
        --account-name $STORAGE_ACCOUNT \
        --quota 1Gi
    ```

## Create Static Persistent Volume using Storage Key

1. Retrieve storage key.

    ```sh
    STORAGE_KEY=$(az storage account keys list --resource-group $RESOURCE_GROUP --account-name $STORAGE_ACCOUNT --query [0].value -o tsv)
    ```
    
2. Create a Kubernetes Secret with Azure Storage Account Name and Storage Key.

    ```sh
    kubectl create namespace files-csi-test

    kubectl create secret generic azure-secret -n csi-test \
        --from-literal=azurestorageaccountname=$STORAGE_ACCOUNT \
        --from-literal=azurestorageaccountkey=$STORAGE_KEY
    ```

3. Review the manifest file `manifests/6-azure-files-csi-static-key.yaml` to ensure PersistentVolume has `csi` section with `driver` as `file.csi.azure.com`.

4. In the manifest replace placeholders `${RESOURCE_GROUP}` and `${FILE_SHARE}` with the values specified above. Apply the manifest.

    ```sh
    kubectl apply -f manifests/5-azure-files-csi-static-key.yaml -n csi-test
    ```

    ```
    OUTPUT:

    persistentvolume/azure-file-static-key created
    persistentvolumeclaim/azure-file-static-key created
    deployment.apps/1-azure-file-static-key created
    deployment.apps/2-azure-file-static-key created
    ```

5. Check whether the resources are provisioned correctly and running.

    ```sh
    kubectl get pod,pv,pvc -n csi-test -l app.kubernetes.io/name=csi-test
    ```

    ```
    OUTPUT:

    NAME                                           READY   STATUS    RESTARTS   AGE
    pod/1-azure-file-static-key-65885756b4-6mhnc   1/1     Running   0          3m23s
    pod/2-azure-file-static-key-65885756b4-zd6hm   1/1     Running   0          3m22s

    NAME                                     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS       REASON   AGE
    persistentvolume/azure-file-static-key   1Gi        RWX            Retain           Bound    csi-test   azure-file-static           3m24s

    NAME                                           STATUS   VOLUME                      CAPACITY   ACCESS MODES  STORAGECLASS           AGE
    persistentvolumeclaim/azure-file-static-key    Bound    azure-file-static           1Gi        RWX                                  3m23s
    ```

6. Verify that persistent volume type is `CSI` and driver is `file.csi.azure.com`. Other properties should match the manifest file.

    ```sh
    kubectl describe persistentvolume/pv-azure-file-static-key -n csi-test
    ```

7. Test the persistent volume for read-write operation on 1st pod. Persistent volume is mounted at `/data` path.

    ```sh
    kubectl exec -it $(kubectl get pod -n csi-test -l app.kubernetes.io/name=csi-test -o jsonpath='{.items[0].metadata.name}') -n csi-test -- sh

    / # ls
    bin        data       dev        etc        home       proc       root       sys        tmp        usr        var
    / # cd data/
    /data # echo "Hello world AKS-CSI from Pod 1 !" > csi-test1
    /data # ls
    csi-test1
    /data # cat csi-test1
    Hello world AKS-CSI from Pod 1 !
    /data # exit
    ```

8. Test the persistent volume for read-write operation on 2nd pod. Persistent volume is mounted at `/data` path.

    ```sh
    kubectl exec -it $(kubectl get pod -n csi-test -l app.kubernetes.io/name=csi-test -o jsonpath='{.items[1].metadata.name}') -n csi-test -- sh

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
kubectl delete namespace csi-test
```

Delete resource group

```sh
az group delete --name $RESOURCE_GROUP
```