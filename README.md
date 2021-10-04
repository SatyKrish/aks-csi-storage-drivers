# AKS Storage Options Using CSI Storage Drivers

A reference implementation for using CSI Storage Drivers based persistent volume for `stateful` workloads hosted on Azure Kubernetes Service (AKS).

## Overview

The [Container Storage Interface (CSI)](https://kubernetes-csi.github.io/docs/) is a standard for exposing arbitrary block and file storage systems to containerized workloads on Kubernetes. By adopting and using CSI, AKS can write, deploy, and iterate plug-ins to expose new or improve existing storage systems in Kubernetes without having to touch the core Kubernetes code and wait for its release cycles.

A `persistent volume` (PV) is a storage resource created and managed by the Kubernetes API that can exist beyond the lifetime of an individual pod. The CSI storage drivers on AKS allows use of `Azure Disks` or `Azure Files` to provide the PersistentVolume.

- **Azure Disks**, which can be used to create a Kubernetes DataDisk resource. Disks can use `Azure Premium Storage`, backed by high-performance SSDs, or `Azure Standard Storage`, backed by regular HDDs or Standard SSDs. For most production and development workloads, use Premium Storage. Azure disks are mounted as `ReadWriteOnce`, so are only available to a single pod. For storage volumes that can be accessed by multiple pods simultaneously, use `Azure Files`.
- **Azure Files**, which can be used to mount an `SMB 3.0` share backed by an `Azure Storage` account to pods. With Azure Files, you can share data across multiple nodes and pods. Azure Files can use `Azure Standard Storage` backed by regular HDDs or `Azure Premium Storage` backed by high-performance SSDs.

    ![AKS Storage Options](/docs/images/aks-storage-options.png)


## Enable CSI Storage Drivers

Container Storage Interface (CSI) drivers for Azure disks and Azure files on `AKS` is now Generally Available (GA) in Kubernetes version 1.21+. Starting in Kubernetes version 1.21, `AKS` will use `CSI drivers` only and by default as storage class. `CSI drivers` are the future of storage extension in Kubernetes. `in-tree` volume plugins are expected to be removed from Kubernetes version 1.23 onwards. 

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
kubectl get storage class
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
kubectl get storage class
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

## Scenarios

- [Azure Disks Dynamic Persistent Volume](docs/azure-disks-dynamic-volume.md)

- [Azure Files Static Persistent Volume using Storage Key](docs/azure-files-static-volume-storage-key.md)

- [Azure Files Static Persistent Volume using Managed Identity](docs/azure-files-static-volume-managed-identity.md)


## References

- [Kubernetes Container Storage Interface (CSI) Documentation](https://kubernetes-csi.github.io/docs/)

- [YouTube: Introduction to Using the Container Storage Interface (CSI) Primitives - Michael Mattsson](https://youtu.be/AnfAd6goq-o)

- [GitHub: Azure Disk CSI driver for Kubernetes](https://github.com/kubernetes-sigs/azuredisk-csi-driver)

- [GitHub: Azure File CSI Driver for Kubernetes](https://github.com/kubernetes-sigs/azurefile-csi-driver)
