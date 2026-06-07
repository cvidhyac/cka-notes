# Storage Concepts

- Persistent Volumes
- Persistent Volume Claims
- Container Storage Interface (CNI)
- CSI Drivers, popular volume plugins
- Cloud Provider storage options and how they work in k8s (Third party storage solutions)
- Storage Classes

## How storage works in containers?

- Docker organizes the data in multiple directories. Within the `/var/lib/docker`, there are multiple directories.
  Data from images is stored in `/images`, data from storage driver is stored in `/aufs`, and when volumes are attached,
  the data
  is stored in `/volume`.
- It is also important to understand the layered architecture in docker.

- When an image is built, it is built in layers. Docker stores these layers in its cache, and when similar image is
  built, same layer is reused from cache. This helps save time and space.

- Overall, there are two layers to keep in mind - "Image Layer" and "Container Layer".

- `docker build -t quay.io/pyapp1` --> when image is built, the "Image Layer is produced". Image layer is **READ ONLY**

- `docker run pyapp1 --image quay.io/pyapp1` --> When container is run referencing the image, then an additional
  "Container Layer" is created on top of the "Image Layer". The container layer is **READ WRITE**

- By default, source code is packaged within an image. Docker creates a copy of the source code on the container layer.
  This copy can be modified, but the changes are only present on the container and original source code in image
  remains unchanged. This mechanism is called "Copy On Write" mechanism. When container is destroyed, this copy is
  destroyed as well. Image should be rebuilt for these changes to be persistent.

- To be able to move these changes between containers without rebuilding the image, a "Volume" should be added

## Docker Volumes

- Two types of mount: Bind mount, volume mount

### What is volume mount?

Default is volume mount, a volume is created by docker and mounted in `/var/lib/docker/volumes` directory mapped to
the path specified in the container.

Create a volume in docker: `docker volume create my_volume`

Specify volume mount: `docker run mycontainer -v my_volume:/var/lib/mysql mysql` --> indicates volume mapped to
container path

When the volume specified in `docker run` command doesn't exist, docker automatically creates that volume in
`/var/lib/docker/<volume_name>` path.

### What is bind mount?

The directory exists on the host in a specific path, and data from container should be bound to the path that exists on
host.

For example, if the data should be stored in /data/mysql on external volume and not stored in container, it should be
bound.

```shell
docker run mycontainer \
--mount type=bind, source=/data/mysql, target=/var/lib/mysql mysql
```

source = path on the host
target = path on the container

### Storage driver

Docker uses storage drivers to manage the layered architecture. It chooses the storage driver based on OS running
docker.

Popular storage driver:

Ubuntu - `aufs`
Fedora - `Device Mapper`
Others - `overlay`, `overlay2`

### Volume Driver

To persist storage, we need volume. Volumes are not handled by storage driver, they have different drivers called
"Volume drivers" that help persist stored data.

Default volume driver is `local`. Other external volume driver can be Azure file storage, Amazon EBS etc.,

## Static Provisioning and Dynamic Provisioning

- Static provisioning is through `hostPath` typed volumes and `PersistentVolume`
- Dynamic provisioning is through `StorageClasses`.

## What is PersistentVolumeClaim?

- When same volume needs to be shared between multiple pods, we create a `PersistentVolumeClaim` which would then
  claim specific storage requests for 100Mi or 500Mi. When pods are deleted, the request goes into `Released` state.
- Persistent Volume Claim will be in pending state until it is claimed by a pod.
- When a PVC is bound a pod, it cannot be deleted until the pod is deleted.

## PersistentVolume

- Needs to be in the same namespace as the pod.
- It has various settings like `ReclaimPolicy`, `VolumeBindingMode`, `StorageClass` and many others. When PV would wait
  for first consumer or pvc bound to a pod, the state of the PV will be Pending, once it is mounted to a pod via PVC or
  direct volume, the status then changes to `Bound`

## StorageClasses

- Helps in dynamic provisioning
- Very useful when storage is provided by cloud provider like portworx or google cloud storage. Each cloud provider
  has their own specific storage class `Provisioner` that needs to be defined in the yaml file.
- Also define access mode (RWO, RWX etc.,) and how much storage is requested.

