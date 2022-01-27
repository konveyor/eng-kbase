---
title: "Fixing duplicate UID ranges on OpenShift namespaces after migration"
date: 2022-01-26
draft: false
authors:
- Pranav Gaikwad
tags:
- Migration Toolkit for Containers
- UID Ranges
- Namespaces
- OpenShift
- Security
- Storage
---

When a Namespace is created in OpenShift, it is assigned a unique User Id (UID) range, a Supplemental Group (GID) range, and unique SELinux MCS labels. They are stored in the `metadata.annotations` field of the Namespace. Everytime a new Namespace is created, OpenShift assigns it a new range from its available pool of UIDs and updates the `metadata.annotations` field to reflect the assigned values. However, if the Namespace resource already has those annotations set, OpenShift does not re-assign new values for the Namespace. It instead assumes that the existing values are valid and moves on. 

### MTC and UID Ranges

Migration Toolkit for Containers (MTC) migrates contents of Namespaces (including the Namespace itself) and the PersistentVolume data from the source cluster to a target cluster. MTC does *not* mutate YAML definitions of Namespace resources. Therefore, definitions of migrated Namespaces in the target cluster are same as that of their source counterparts. Similarly, MTC also preserves ownership and permissions of the persistent data.

In the target cluster, OpenShift assigns the same UID ranges to the Namespace as the source since the annotations are preserved. MTC intentionally preserves these annotations. If it were not to preserve them, OpenShift will assign new UID ranges to the restored Namespace. And the migrated data will be inaccessible to the workloads restored in the target cluster since they will start with a different UID than the source. 

It is possible that the ranges of the restored Namespaces collide with existing Namespaces as they were not originally assigned by OpenShift. They were rather a result of pre-existing annotations. In most cases, the colliding ranges do not pose a security threat as there are other ways to configure workloads in OpenShift that provide sufficient isolation. In some cases, the duplicate ranges may be a problem for users due to various reasons. For example, compliance requirements.

### Fixing UID ranges after migration

As stated in the previous section, MTC intentionally preserves the UID ranges. It cannot automatically change the UID ranges of namespaces in the target cluster due to data access issues. In case the colliding ranges become a problem, the users are required to manually update them after migration. In some cases, this also means updating the file ownership of persistent volume data.

Following is a step by step guide on updating the UID ranges of namespaces manually:

#### Step 1: Updating the UID ranges on Namespace resources

The first step is to make OpenShift assign new UID ranges from its pool. This can be done by simply deleting the existing annotations on the Namespaces. Once deleted, OpenShift will automatically re-assign new ranges such that they don't collide with existing ones. Following annotations need to be deleted:

```yaml
  "openshift.io/sa.scc.mcs": "s0:c26,c15",
  "openshift.io/sa.scc.supplemental-groups": "1000680000/10000",
  "openshift.io/sa.scc.uid-range": "1000680000/10000"
```

> It is recommended to keep the original values backed up so that the workloads can be rolled back in case of failures in further steps.

#### Step 2: Quiescing apps

After step 1, the Namespaces will have new non-overlapping UID ranges assigned to them. However, the workloads will still be running with their old UIDs. The new UIDs will take effect only after the workloads are restarted. In this step, the workloads will simply be quiesced down.

Before quiescing down, the current replica values of workload resources need to stored. The value will be needed at the time of un-quiescing the workloads. They can be simply stored as annotation on the workload itself. An example of saving replica value for a Deployment resource:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    previous-replicas: "3"   <--- Saving replica count
  [...]
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  [...]
```

Once the annotations are applied, the replicas of workload resources will be set to 0:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    previous-replicas: "3"
  [...]
spec:
  progressDeadlineSeconds: 600
  replicas: 0   <--- Quiesce down
  [...]
```

#### Step 3: Updating ownership of PersistentVolume data

In this step, the file ownership of the data will be updated manually. Note that this step may or may not be required depending on the underlying storage used by the PVCs.
#### When this step may not be required?

If the storage is a block storage, Kubernetes will automatically `chown` files on the volumes when the workloads are un-quiesced in Step 4. AWS EBS, GCE Persistent Disks are some examples of block storages that Kubernetes automatically updates the ownership of the data based on the UID of running workloads. One way to determine whether or not the automatic `chown` is supported is by looking at the `fsType` attribute of the underlying PersistentVolume object:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pvc-227d5669-01fc-49df-8d34-294da0d9ee60
spec:
  gcePersistentDisk:
    fsType: ext4    <--- fsType field is set
```

If this field is set, Kubernetes will automatically `chown` the volume. 

If the storage is a shared storage, an additional list of Supplemental Groups can be provided to the workloads. The workloads can simply access the files using one of the GIDs provided in the Supplemental Groups section of the Pod's security context: 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restic-zppwk
  namespace: openshift-migration
spec:
  [...]
  securityContext:
    supplementalGroups:
    - 1000
```

In the above example, the Pods will access the files using GID 1000. Note that the files on the shared storage need to be group accessible by GID 1000. It is not advisable to access shared volumes using UIDs as the files can be shared between multiple workloads. The workloads may access them using different UIDs. Therefore, recommended way of configuring access to group storages is through Supplemental Groups. For Supplemental Groups to work, it is necessary that the SCC assigned to the Pod needs to allow these GID values. An additional custom SCC can also be created to allow these ranges. 

#### Manually updating ownership

In all other cases, the file ownership needs to be fixed manually by using an additional Pod. A dummy Pod can be launched in the namespace with all the PVCs attached to it. The Pod can simply run `chown` recursively for all the paths where the PVCs are attached. It needs to run in privileged mode. If the number of PVCs in the Namespace is large, the Pod can attach PVCs in batches. Additionally, multiple pods can run simultaneously in the same namespace to reduce the total time. The correct UID value to use can be read from the annotation of the Namespace.

For instance, consider that we have a Namespace with its assigned UID range as `1000680000/10000`. It has two PVCs `pvc-0` and `pvc-1` in it. The migrated data in the backing PVs is owned by user beloging to a different UID range. Then a dummy Pod can mount these PVCs and run `chown` with correct UID:

```
apiVersion: v1
kind: Pod
metadata:
  name: chown-files
  namespace: <namespace>
spec:
  containers:
  - name: owner-modifier
    image: rhel:latest
    command:
    - /bin/bash
    - -c
    - /usr/bin/chown -R 1000680000 /tmp/
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /tmp/pvc-0/
      name: volume-0
    - mountPath: /tmp/pvc-1/
      name: volume-1
  volumes:
  - name: volume-0
    persistentVolumeClaim:
      claimName: pvc-0
  - name: volume-1
    persistentVolumeClaim:
      claimName: pvc-1
```

The above Pod attaches all the PVs at distinct locations under `/tmp` directory and runs `chown` recursively on all directories under `/tmp` to fix the ownership of data.

Additionally, to update MCS labels, `chcon -R -l <labels>` can also be added to the Pod command. 

#### Step 4: Bringing workloads back online

Upon successful execution of Step 3, the persistent data will be owned by the correct UID belonging to the Namespace range. In this step, the original replica count of the workloads will be restored from the annotation added in Step 2:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    previous-replicas: "3"
  [...]
spec:
  progressDeadlineSeconds: 600
  replicas: 3   <--- Setting back the original value to un-quiesce app
  [...]
```

### Conclusion

MTC deliberately preserves UID ranges of migrated namespaces to avoid file access issues after migration. Most users don't find it to be an issue. In case they do, the UID ranges can be fixed manually. This document describes one of the several ways to manually fix UID ranges after migration. Some of the tasks in the process can also be automated with Ansible to perform updates at scale.