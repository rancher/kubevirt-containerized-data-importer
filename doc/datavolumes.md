# Data Volumes

## Introduction
Data Volumes(DV) are an abstraction on top of Persistent Volume Claims(PVC) and the Containerized Data Importer(CDI). The DV will monitor and orchestrate the upload/import of the data into the PVC. Once the process is completed, the DV will be in a consistent state that allow consumers to make certain assumptions about the DV in order to progress their own orchestration.

Why is this an improvement over simply looking at the state annotation created and managed by CDI? Data Volumes provide a versioned API that other project like [Kubevirt](https://github.com/kubevirt/kubevirt) can integrate with. This way those project can rely on an API staying the same for a particular version and have guarantees about what that API will look like. Any changes to the API will result in a new version of the API.

### Status phases
The following statuses are possible.
* 'Blank': No status available.
* Pending: The operation is pending, but has not been scheduled yet.
* PVCBound: The PVC associated with the operation has been bound.
* Import/Clone/UploadScheduled: The operation (import/clone/upload) has been scheduled.
* Import/Clone/UploadInProgress: The operation (import/clone/upload) is in progress.
* SnapshotForSmartClone/SmartClonePVCInProgress: The Smart-Cloning operation is in progress.
* Succeeded: The operation has succeeded.
* Failed: The operation has failed.
* Unknown: Unknown status.

## HTTP/S3/Registry source
DataVolumes are an abstraction on top of the annotations one can put on PVCs to trigger CDI. As such DVs have the notion of a 'source' that allows one to specify the source of the data. To import data from an external source, the source has to be either 'http' ,'S3' or 'registry'. If your source requires authentication, you can also pass in a `secretRef` to a Kubernetes [Secret](../manifest/example/endpoint-secret.yaml) containing the authentication information.  TLS certificates for https/registry sources may be specified in a [ConfigMap](../manifests/example/cert-configmap.yaml) and referenced by `certConfigMap`.  `secretRef` and `certConfigMap` must be in the same namespace as the DataVolume.

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: "example-import-dv"
spec:
  source:
      http:
         url: "https://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img" # Or S3
         secretRef: "" # Optional
         certConfigMap: "" # Optional
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: "64Mi"
```
[Get example](../manifests/example/import-kubevirt-datavolume.yaml)
[Get secret example](../manifests/example/endpoint-secret.yaml)
[Get certificate example](../manifests/example/cert-configmap.yaml)

Alternatively, if your certificate is stored in a local file, you can create the `ConfigMap` like this:

```bash
kubectl create configmap import-certs --from-file=ca.pem
```

### Content-type
You can specify the content type of the source image. The following content-type is valid:
* kubevirt (Virtual disk image, the default if missing)
* archive (Tar archive)
If the content type is kubevirt, the source will be treated as a virtual disk, converted to raw, and sized appropriately. If the content type is archive it will be treated as a tar archive and CDI will attempt to extract the contents of that archive into the Data Volume.
An example of an archive from an http source:

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: "example-import-dv"
spec:
  source:
      http:
         url: "http://server/archive.tar"
         secretRef: "" # Optional
   contentType: "archive"
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: "64Mi"
```

## PVC source
You can also use a PVC as an input source for a DV which will cause a clone to happen of the original PVC. You set the 'source' to be PVC, and specify the name and namespace of the PVC you want to have cloned. Be sure to specify the right amount of space to allocate for the new DV or the clone can't complete.

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: "example-clone-dv"
spec:
  source:
      pvc:
        name: source-pvc
        namespace: example-ns
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: "128Mi"
```
[Get example](../manifests/example/clone-datavolume.yaml)

## Upload Data Volumes
You can upload a virtual disk image directly into a data volume as well, just like with PVCs. The steps to follow are identical as [upload for PVC](upload.md) except that the yaml for a Data Volume is slightly different.
```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: example-upload-dv
spec:
  source:
    upload: {}
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
```

## Blank Data Volume
You can create a blank virtual disk image in a Data Volume as well, with the following yaml:
```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: example-blank-dv
spec:
  source:
    blank: {}
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
```

## Image IO Data Volume
Image IO sources are sources from oVirt imageio endpoints. In order to use these endpoints you will need an oVirt installation with imageIO enabled. You will then be able to import disk images from oVirt into KubeVirt. The diskId can be obtained from the oVirt webadmin UI or REST api.
```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: "test-dv"
spec:
  source:
      imageio:
         url: "http://<ovirt engine url>/ovirt-engine/api"
         secretRef: "endpoint-secret"
         certConfigMap: "tls-certs"
         diskId: "1"
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: "500Mi"
```
[Get secret example](../manifests/example/endpoint-secret.yaml)
[Get certificate example](../manifests/example/cert-configmap.yaml)

## Block Volume Mode
You can import, clone and upload a disk image to a raw block persistent volume.
This is done by assigning the value 'Block' to the PVC volumeMode field in the DataVolume yaml.
The following is an exmaple to import disk image to a raw block volume:
```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: "example-import-dv"
spec:
  source:
      http:
         url: "https://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img" # Or S3
         secretRef: "" # Optional
         certConfigMap: "" # Optional
  pvc:
    volumeMode: Block
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: "64Mi"
```

## Conditions
The DataVolume status object has conditions. There are 3 conditions available for DataVolumes
* Ready
* Bound
* Running

The running and ready conditions are mutually exclusive, if running is true, then ready cannot be true and vice versa. Each condition has the following fields:
* Type (Ready/Bound/Running)
* Status (True/False)
* LastTransitionTime The timestamp when the last transition happened.
* LastHeartbeatTime the timestamp the last time anything on the condition was updated.
* Reason The reason the status transitioned to a new value, this is a camel cased single word, similar to an EventReason in events.
* Message A detailed messages expanding on the reason of the transition. For instance if Running went from True to False, the reason will be the container exit reason, and the message will be the container exit message, which explains why the container exitted.


## Kubevirt integration
[Kubevirt](https://github.com/kubevirt/kubevirt) is an extension to Kubernetes that allows one to run Virtual Machines(VM) on the same infra structure as the containers managed by Kubernetes. CDI provides a mechanism to get a disk image into a PVC in order for Kubevirt to consume it. The following steps have to be taken in order for Kubevirt to consume a CDI provided disk image.
1. Create a PVC with an annotation to for instance import from an external URL.
2. An importer pod is start that attempts to get the image from the external source.
3. Create a VM definition that references the PVC we just created.
4. Wait for the importer pod to finish (status can be checked by the status annotation on the PVC).
5. Start the VMs using the imported disk.
There is no mechanism to stop 5 from happening before the import is complete, so once can attempt to start the VM before the disk has been completely imported, with obvious bad results.

Now lets do the same process but using DVs.
1. Create a VM definition that references a DV template, which includes the external URL that contains the disk image.
2. A DV is created from the template that in turn creates an underlying PVC with the correct annotation.
3. The importer pod is created like before.
4. Until the DV status is Success, the virt launcher controller will not schedule the VM to be launched if the user tries to start the VM.
We now have a fully controlled mechanism where we can define a VM using a DV with a disk image from an external source, that cannot be scheduled to run until the import has been completed.

### Example VM using DV
```yaml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  creationTimestamp: null
  labels:
    kubevirt.io/vm: vm-fedora-datavolume
  name: vm-fedora-datavolume
spec:
  dataVolumeTemplates:
  - metadata:
      creationTimestamp: null
      name: fedora-dv
    spec:
      pvc:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 100M
        storageClassName: hdd
      source:
        http:
          url: https://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-datavolume
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: datavolumevolume
        machine:
          type: ""
        resources:
          requests:
            memory: 64M
      terminationGracePeriodSeconds: 0
      volumes:
      - dataVolume:
          name: fedora-dv
        name: datavolumevolume
```
[Get example](../manifests/example/vm-dv.yaml)

This example combines all the different pieces into a single yaml.
* Creation of a VM definition (example-vm)
* Creation of a DV with a source of http which points to an external URL (example-dv)
* Creation of a matching PVC with the same name as the DV, which will contain the result of the import (example-dv).
* Creation of an importer pod that does the actual import work.
