---
title: "Copying Pvc Data Manually"
date: 2021-05-21T13:43:26-04:00
draft: false
author: Pranav Gaikwad
tags:
- crane
- state migration
- persistent volumes
- PVs
- rsync
---

Sometimes there is a need top copy select files from one PVC to another. This
post describes how to use rsync and a bastian host that has access to both
PVCs (same cluster or different) to copy selected files.

#### Set up a temporary Pods

In order to copy data using rsync, both the source and destination PVC needs
to be mounted on a Pod that has `rsync` and `tar` binaries. If you already
have the PVC's mounted with pods that has both the aforementioned binaries,
this step can be skipped.

On source namespace:
1. Login to the source cluster and go to source namespace
   `oc project source-namespace`
2. Get the PVC name using `oc get pvc`
3. Create the pod mounting source PVC
    ``` bash
   $ cat<<EOF | oc create -f -
   apiVersion: v1
   kind: Pod
   metadata:
     name: busybox-sleep
     namespace: source-namespace
   spec:
     containers:
     - name: busybox
       image: registry.redhat.io/rhel7/rhel-tools:latest
       args:
       - sleep
       - "1000000"
       volumeMounts:
       - mountPath: /mnt/pvc-name
         name: pvc-name
     volumes:
     - name: pvc-name
       persistentVolumeClaim:
         claimName: pvc-name
   EOF
    ```
   replace pvc-name with the pvc-name from step 2
4. Confirm the volume mount by rshing into the pod and running df
    ```bash
   $ oc rsh busybox-sleep
   sh-4.2# df -h
   Filesystem                            Size  Used Avail Use% Mounted on
   overlay                               120G   16G  104G  14% /
   tmpfs                                  64M     0   64M   0% /dev
   tmpfs                                 7.9G     0  7.9G   0% /sys/fs/cgroup
   shm                                    64M     0   64M   0% /dev/shm
   tmpfs                                 7.9G  5.1M  7.9G   1% /etc/hostname
   /dev/rbd0                             976M  3.3M  957M   1% /mnt/pvc-0
   /dev/mapper/coreos-luks-root-nocrypt  120G   16G  104G  14% /etc/hosts
   tmpfs                                 7.9G   28K  7.9G   1% /run/secrets/kubernetes.io/serviceaccount
   tmpfs                                 7.9G     0  7.9G   0% /proc/acpi
   tmpfs                                 7.9G     0  7.9G   0% /proc/scsi
   tmpfs                                 7.9G     0  7.9G   0% /sys/firmware
   sh-4.2#
    ```

   In this case the pvc name is pvc-0 and the mount path in the pod is
   `/mnt/pvc-0`

Repeat the above steps for destination PVC as well.

#### Coping the files

Now assume you want to copy the following three files from the source PVC.
```
/mnt/pvc-0/10
/mnt/pvc-0/a/11
/mnt/pvc-0/a/51
```

Use the following steps:
1. On your local machine, create a temperory directory to store these files
   `mkdir ./pvc-files`
2. Use the following loop to copy the files over
   ``` bash
   sudo oc login <server> -u <user> -p <password>
   sudo oc project source-namespace
   for i in /mnt/pvc-0/10 /mnt/pvc-0/a/11 /mnt/pvc-0/a/51; do
       sudo rsync --relative -a --progress --rsh='oc rsh' busybox-sleep:$i ./pvc-files/
   done
   ```
3. Verify the file `ls -lR ./pvc-files` permissions
4. Verify the files checksum
   ```
    for i in /mnt/pvc-0/10 /mnt/pvc-0/a/11 /mnt/pvc-0/a/51; do
           pod_checksum=$(oc exec busybox-sleep -- md5sum "$i" | awk '{printf $1}');
           local_checksum=$(sudo md5sum "/tmp/pvc-files$i" | awk '{printf $1}');
           if [[ $pod_checksum != $local_checksum ]]; then
                  echo "checksum for $i is not equal $pod_checksum $local_checksum"
           fi
    done
   ```
   if the for loop does not print anything all checksums are verified
5. Copy all the files to destination PVC. We will have to change directory
   into the pvc-0 folder to maintain the directory structure.
   ```bash
   sudo oc project destination-namespace
   cd pvc-files/mnt/pvc-0
   for i in 10 a/11 a/51; do
      sudo rsync -a --progress --relative --rsh='oc rsh' $i busybox-sleep:/mnt/pvc-0/  ;    done
   done
   ```

6. Verify the checksum from local to destination
   ```bash
   cd ../..
   oc project destination-namespace
   for i in /mnt/pvc-0/10 /mnt/pvc-0/a/11 /mnt/pvc-0/a/51; do
          pod_checksum=$(oc exec busybox-sleep -- md5sum "$i" | awk '{printf $1}');
          local_checksum=$(sudo md5sum "/tmp/pvc-files$i" | awk '{printf $1}');
          if [[ $pod_checksum != $local_checksum ]]; then
                 echo "checksum for $i is not equal $pod_checksum $local_checksum"
          fi
   done
   ```
   If the above for-loop did not print anything all the files have been safely
   copied over to the destination



