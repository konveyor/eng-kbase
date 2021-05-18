---
title: "{{ replace .Name "-" " " | }}"
date: {{ .Date }}
draft: true
---

# Cache problems with older restic

Recently the engineering team was engaged in a customer issue where using
restic for PV migration was failing. The issue reported was very close to
[this](https://github.com/restic/restic/issues/2244), the details of which
are captured [here](https://bugzilla.redhat.com/show_bug.cgi?id=1960655).

Prior to restic v0.10.0, restic was using a restore mechanism where it
needed to predict the cache size. The cache size was hard coded into the
codebase around 5 MB [here](https://github.com/restic/restic/blob/ecc2458de8f94a2a0fe8300c74057ab77680d713/internal/restorer/filerestorer.go#L29-L34).
When the backup chunks are greater than 5 MB restic would complain about
`not enough cache capacity`. When `verify` flag for the PV was turned on, it
was found restic restore would skip the 'chunks' for which restic did
not have enough cache size. This gave a lot of verification errors and exposed
the underlying bug.


#### Steps to reproduce the issue

In order to reproduce the issue the pack size in the backup has to be much
larger than 5 MB. For backup, the restic codebase has minimum size of 4 MB
and max size of 16 MB. In order to create a faulty backup, the minimum pack
size was modified to 10 MB and the max to 20 MB. 
It was observed that this change in the backup code produced a backup with
larger packs, that crashed restic 0.9.6. However, running this in v0.12.0
which has the fix for the issue, restic restore work perfectly as expected.

The steps followed to reproduce the issue in MTC 1.4.3 is as follows:

1. Edit the migration-controller spec on the source adding the following:
```
...
spec:
  velero_image_fqin: quay.io/alaypatel07/velero:min-file-size
...
```
This velero image is built from a restic source code that changes the minimum
pack size to 10 MB as explained earlier, the source code is [here](https://github.com/alaypatel07/restic/commit/54cfd3328e1b1714cd54d3107525a7dbbcff2d5d)

2. Wait for the migration-controller status to reflect `phase: Reconciled`, 
that means the restic daemonset is patched with the image in step 1
3. Deploy an application that creates a few gigabytes of random data. 
The PVC use in trial run had 13 gigs of data.
4. Create a migplan for the application and make sure the PVCs have checksum
 `verify: true` flag.
5. The migmigration will complete with warnings.
6. The PodVolumeRestore on the destination will look like this:
```
status:
  completionTimestamp: "2021-05-17T15:20:25Z"
  errors: 126
  phase: Completed
  progress:
    bytesDone: 13078593536
    totalBytes: 13078593536
  resticPod: restic-vc8qx
  startTimestamp: "2021-05-17T15:17:55Z"
  verifyErrors: 1
```
7. The restic pod logs will look like this:
```
time="2021-05-17T15:20:25Z" level=info msg="Ran command=restic restore --repo=s3:https://minio-openshift-operators.apps.jmontleonsrc.migration.redhat.com/alpatel-velero-0/velero/restic/small-
files --password-file=/tmp/velero-restic-credentials-small-files416747925 --insecure-skip-tls-verify --cache-dir=/scratch/.cache/restic e703c4b7 --target=. --delete --verify, stdout=running r
estore with modified pack size\nrestoring <Snapshot e703c4b7 of [/host_pods/4728e98a-4409-4bdc-8836-6353a3932c27/volumes/kubernetes.io~aws-ebs/pvc-e8a200cf-1fa3-423c-b794-fa90c6b30397] at 202
1-05-17 15:08:50.24911997 +0000 UTC by root@velero> to .\nverifying files in .\nfinished verifying 130 files in .\nThere were 126 errors\n, stderr=ignoring error for /file1111: not enough cac
he capacity: requested 12634684, available 600319\nignoring error for /file1009: not enough cache capacity: requested 12108447, available 600319\nignoring error for /file1112: not enough cach
e capacity: requested 8686519, available 600319\nignoring error for /file1113: not enough cache capacity: requested 6850718,
```

Furthermore, once this backup was created, it was verified the newer
version of restic that has the [fix](https://github.com/restic/restic/pull/2195) 
did not fail of the same backup

The steps are as follows:

1. Scale down the migration-operator deployment so that it allows us to muck
with velero image, `oc scale deployment migration-controller --replicas=0`
2. Update the velero image in migration-controller CR on the destination 
cluster to use an image that has restic version v0.12.0 (one with the fix)
3. Run the same migration again by creating another stage. Check the pack 
sizes on minio console, it was greater than 10 MB. The restore succeeded 
without any warnings.

#### Conclusion

This verifies that whenever new restic will land in our product, we will
not see this issue.
