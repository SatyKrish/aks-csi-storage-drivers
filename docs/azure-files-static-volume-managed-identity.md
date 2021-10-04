# Azure Files Static Persistent Volume using Managed Identity

The Azure Files CSI driver is CSI specification compliant, and used by AKS to manage the lifecycle of Azure file shares attached to pod as `PersistentVolume`.

In this sample we will statically create `PersistentVolume` with an existing Azure Files share using managed identity. Managed identities can authorize access to file share from AKS cluster using Azure AD credentials. By using managed identities, you can avoid storing Storage Key as a Kubernetes secret in AKS cluster.

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

3. Create a general-purpose storage account and a file share 

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

## Create Static Persistent Volume using Kubelet Identity

1. Get cluster identity used by the AKS cluster agentpools.

    ```sh
    KUBELET_IDENTITY=$(az aks show -g ${RESOURCE_GROUP} -n ${CLUSTER_NAME} --query identityProfile.kubeletidentity.objectId -o tsv)
    ```

2. Get storage account resource id.

    ```sh
    STORAGE_RESOURCE_ID=$(az storage account show -g ${RESOURCE_GROUP} -n ${STORAGE_ACCOUNT} --query id -o tsv)
    ```

3. Assign kubelet identity with `Storage Account Key Operator Service Role` ans `Reader` roles scoped to the storage account..

    ```sh
    az role assignment create --assignee ${KUBELET_IDENTITY} --role 'Storage Account Key Operator Service Role' --scope ${STORAGE_RESOURCE_ID}
    
    az role assignment create --assignee ${KUBELET_IDENTITY} --role 'Reader' --scope ${STORAGE_RESOURCE_ID}
    ```

4. Review the manifest file `manifests/5-azure-files-csi-static-mi.yaml` to ensure PersistentVolume has `csi` section with `driver` as `file.csi.azure.com`.

4. In the manifest replace placeholders `${RESOURCE_GROUP}` and `${FILE_SHARE}` with the values specified above. Apply the manifest.

    ```sh
    kubectl apply -f manifests/5-azure-files-csi-static-mi.yaml -n csi-test
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

    NAME                                         READY   STATUS    RESTARTS   AGE
    pod/1-azure-file-static-mi-f66f9f956-jtpc8   1/1     Running   0          37s
    pod/2-azure-file-static-mi-cfc95bfc5-2jrbr   1/1     Running   0          37s

    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                STORAGECLASS   REASON   AGE
    persistentvolume/pv-azure-file-static-mi   1Gi        RWX            Retain           Bound    csi-test/pvc-azure-file-static-mi                            39s

    NAME                                             STATUS   VOLUME                    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    persistentvolumeclaim/pvc-azure-file-static-mi   Bound    pv-azure-file-static-mi   1Gi        RWX                           38s
    ```

6. Verify that persistent volume type is `CSI` and driver is `file.csi.azure.com`. Other properties should match the manifest file.

    ```sh
    kubectl describe persistentvolume/pv-azure-file-static-mi -n csi-test
    ```

    ```
    OUTPUT:

    Name:            azure-file-static-key
    Labels:          app.kubernetes.io/name=csi-test
    Annotations:     pv.kubernetes.io/bound-by-controller: yes
    Finalizers:      [kubernetes.io/pv-protection external-attacher/file-csi-azure-com]
    StorageClass:
    Status:          Bound
    Claim:           csi-test/azure-file-static-key
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
        VolumeAttributes:      resourceGroup=aks-sandbox-de
                               1shareName=aks-test
                               storageAccount=satyaksstorage

    Events:                <none>
    ```

7. Test the persistent volume for read-write operation on 1st pod. Persistent volume is mounted at `/data` path.

    ```sh
    kubectl exec -it $(kubectl get pod -n csi-test -l app.kubernetes.io/name=csi-test -o jsonpath='{.items[0].metadata.name}') -n csi-test -- sh

    / # ls
    bin        data       dev        etc        home       proc       root       sys        tmp        usr        var
    / # cd data/
    /data # ls
    csi-test1
    csi-test2
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
    csi-test2
    /data # cat csi-test2
    Hello world AKS-CSI from Pod 2 !
    /data # exit
    ```

## Troubleshooting

- For `permission denied` error while mounting Azure Files share, refer to the troubleshooting instructions [here](https://docs.microsoft.com/en-us/azure/storage/files/storage-troubleshoot-linux-file-connection-problems#mount-error13-permission-denied-when-you-mount-an-azure-file-share)

## Clean-up

Uninstall `csi-test` Ltic-pv.yaml -n csi-test
```

Delete the resources created in `files-csi-test` namespace. 

```sh
kubectl delete namespace files-csi-test
```

Delete resource group

```sh
az group delete --name $RESOURCE_GROUP
```