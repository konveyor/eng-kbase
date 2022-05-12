---
title: "Pre-requisites for preserving owners and groups on migrated data"
date: 2022-05-12
draft: false
authors:
- Pranav Gaikwad
tags:
- Migration Toolkit for Containers
- Persistent Volumes
- Data migration
- File ownership
- rsync
---

Migration Toolkit for Containers (MTC) uses _rsync_ to migrate Persistent Volume data from source cluster to destination cluster. It uses `--group` & `--owner` options in rsync to preserve UID/GID bits on files. The files are owned by same users and groups on the destination cluster as the source cluster. This is to ensure that the workloads run correctly in the destination cluster post migration. The goal of this article is to explain how _rsync_ preserves UID/GIDs in MTC. It describes how some storages can limit _rsync's_ capability to preserve UID/GIDs. Finally, it concludes by listing MTC's expectations from storage layer for its data migration to work effectively. 

### Rsync UID/GID basics

When files are transferred using _rsync_ to a destination, the files are written as the user running the _rsync_ process on the destination side. As a result, UID on the transferred files will be the ID of the user running _rsync_ on destination. The GID will be the primary group of that user.  When `--user` and `--group` options are specified, _rsync_ changes the UID/GID back to whatever they were on the source. To do that, _rsync_ effectively executes `chown`. `chown` can only be run by a user owning the file or a super-user. 

### MTC and Rsync

In MTC, _rsync_ runs as a super-user (i.e. UID 0). The admission of the _rsync_ Pod is handled via a custom SCC named `rsync-anyuid` created in the destination cluster. Being a super-user in the destination side allows _rsync_ to change UID/GIDs of the migrated files to match the source. Most storages do not change the UID/GID on written files themselves. However, there are some exceptions.

#### AWS EFS

In OpenShift, AWS Elastic File System (EFS) is made available by the EFS Operator. It allows provisioning EFS volumes dynamically or statically. When using dynamic provisioning, the EFS CSI driver [enforces user identity](https://docs.aws.amazon.com/efs/latest/ug/efs-access-points.html#enforce-identity-access-points) on the Persistent Volumes by default. For each volume, it creates a new access point and assigns it a unique UID/GID from a pool of pre-configured ID ranges. For all operations performed on such volumes, EFS automatically replaces the client's user and group IDs with the configured IDs. The NFS client's IDs are ignored. As a result, a remote super user is unable to perform `chown` on files stored in the volumes. In case of MTC, this prevents _rsync_ from running `chown` after file transfer and fails the migration (See [upstream issue](https://github.com/kubernetes-sigs/aws-efs-csi-driver/issues/300)). The EFS volumes need to bypass the user enforcement. When using the CSI driver, this is only possible when the PVs are provisioned statically via an access point that does not have user enforcement set.

#### NFS

When an NFS server is configured with `root_squash`, the server does not allow a remote super user (NFS client) to perform operations as a super user on the server. The root users are instead demoted to an anonymous user automatically (often `nfs-nobody`). As a result, any files created by a remote root user will be automatically written as `nfs-nobody` user on the server. The remote user will also be not allowed to run `chown` after writing the file as its not actually operating as root on the server. In case of MTC, this prevents _rsync_ from running `chown` after file transfer and fails the migration as the UID/GIDs cannot be changed to their expected value in the destination Persistent Volumes.

### Conclusion

In order to guarantee that migrated applications run correctly in destination cluster, MTC must preserve file owners and groups. To achieve that, MTC runs _rsync_ with `--owner` and `--group` options that allow _rsync_ to execute `chown` after file transfer. The server _rsync_ Pod in the destination cluster must run as a root user (i.e. UID 0). It is expected that the underlying storage does not change user and group IDs on the files written by _rsync_. Any changes in UID/GID made by the storage will result in a failed migration. Additionally, the admission of the Pod is managed by `rsync-anyuid` SCC that is installed by the Migration Operator. It is necessary that the _rsync_ Pods are admitted via this SCC. A custom less privileged SCC enforced on _rsync_ Pods will prevent the migration from succeeding.

### See also

A more detailed explanation on how MTC handles UIDs assigned to applications can be found in [this article](https://engineering.konveyor.io/posts/manual-uid-updates-post-migration/#mtc-and-uid-ranges).

