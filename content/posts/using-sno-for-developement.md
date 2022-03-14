---
title: "Using Single Node OpenShift for Development"
date: 2021-11-03T10:38:38-04:00
draft: false
authors:
- Jason Montleon
tags:
- openshift
- sno
- single-node
---

## Introduction
Single Node OpenShift is a new offering that also works great for development, although it does require some planning and preparation before using it for the first time. You will need to provide 8 vCPU, 32GB of RAM, and 120GB of storage at a minimum.  

- Decide where you want to run SNO after reviewing the minimum requiremnents
- Configuring a static lease is helpful but not required
- Configuring DNS entries is preferable, but again optional

Minimum Requirements:
- vCPU: 8
- RAM: 32GB
- Storage: 120GB

In the example below I used a KVM virtual machine with a static lease and DNS entries created in dnsmasq.

## Installation

- Visit [https://console.redhat.com/openshift/assisted-installer/clusters/](https://console.redhat.com/openshift/assisted-installer/clusters/) and login
- Click `Create Cluster`

### Cluster Details
On this page we will fill in some basic information 

- Give the cluster a name
- Choose the base domain
- Check `Install single node OpenShift (SNO)`
- Choose version 4.9.0
- Click Next

### Host Discovery
On this page we will set up a discovery image for our cluster to boot the VM off of.

- Click `Generate Discovery Image`
- Enter an ssh key. 
- Click `Generate Discovery ISO`
- Use one of the provided options to download the ISO
  
In the next section we will be configuring a VM and booting it, but do not close this page. We will be returning to it.

### Virtual Machine Creation
We must now configure a VM to boot off of the ISO we downloaded. In the following steps I will use Virtual Machine Manager to create a KVM VM, but other virtualization options are likely also acceptable.

- Start Virtual Machine Manager 
- Click File>New Virtual Machine
- Leave the default `Local install media (ISO Image or CDROM)
- Click Forward
- Browse to the ISO you downloaded
- Uncheck Automatically detect from installation media / source
- Search for and select `Red Hat Enterprise Linux 9.0` and select it
- Click Forward
- Set the Memory to 32GiB or greater
- Set teh CPUs to 8 or greater
- Click Forward
- Set Storage to 120 GiB or greater
- Click Forward
- Update the VM name if desired
- Check `Customize configuration before install`
- Choose the networking option you want to use
- Click Finish
  
Since we checked `Customize configuration before install` a new window will pop up.  
  
If you would like to set a static lease review the MAC address assigned to the NIC now and do so. The means of doing this can vary drastically by environment, so I will not attempt to cover the process. Note that you should not name the host the same as your cluster. For example, if you configured your cluster to be `openshift.example.com` the node should not be configured as `openshift.example.com`, but `openshift-node.example.com` would be acceptable.

When done click `Begin Installation` and the VM should boot from the ISO.

### Finishing Discovery
Return to the assisted installer discovery page.

After a moment the VM you booted from the ISO should be discovered. If you set a static lease with a DNS name for the system it should appear with that name, otherwise you will likely see `localhost`. If you see `localhost` change the name to something else.
  
If you hit the drop down you may see `NTP Status` not yet ready. I find that the process is smoother if I wait until this is marked Ready, which may take a moment or two, though no action is required.

When done, click Next.

### Networking
- Choose your subnet in the `Available Subnets` drop down
- Click Next

### Review and create
- If everything looks acceptable click `Install cluster`

As the installation proceeds you will be provided with a kubeconfig for the new cluster, and once it completes you will be provided with a password for the kubeadmin user.

### Configure DNS
You may either configure DNS with two entries, or edit /etc/hosts on your system to access the new cluster. It is important that you do not create a wildcard address for the cluster domain. Instead create an entry for api.cluster-domain.base-domain, and a wildcard for apps.cluster-domain.base-domain.

In dnsmasq an example of this would look like:
```
host-record=api.openshift.example.com,192.168.1.202
address=/.apps.openshift.example.com/192.168.1.202
```

For bind:
```
api.openshift.example.com      A	192.168.1.202
*.apps.openshift.example.com   A	192.168.1.202
```

Alternatively, modify /etc/hosts:
```
192.168.1.202	api.openshift.example.com
192.168.1.202	oauth-openshift.apps.openshift.example.com
192.168.1.202	console-openshift-console.apps.openshift.example.com
192.168.1.202	grafana-openshift-monitoring.apps.openshift.example.com
192.168.1.202	thanos-querier-openshift-monitoring.apps.openshift.example.com
192.168.1.202	prometheus-k8s-openshift-monitoring.apps.openshift.example.com
192.168.1.202	alertmanager-main-openshift-monitoring.apps.openshift.example.com
192.168.1.202	example-node.example.com
```

### Optional SSH config
From time to time I have reprovisioned my cluster. Since this changes the SSH host keys for the cluster I have opted to not store them.

In my .ssh/config I have an entry for the node:
```
Host openshift-node.example.com
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
  User core
```

### PVs and Storageclass
For testing stateful migrations with MTC it is helpful to have a storageclass and associated PVs available. As SNO does not configure any by default you will need to do so yourself. With the ssh configuration above, and my ssh key unlocked with my ssh agent, I run the following script:

```
#!/bin/bash

export KUBECONFIG=~/Documents/sno-kubeconfig

ssh -i ~/.ssh/id_rsa_ocp core@openshift-node.example.com "sudo /bin/bash -c 'mkdir -p /srv/openshift/pv-{0..99} ; chmod -R 777 /srv/openshift ; chcon -R -t svirt_sandbox_file_t /srv/openshift'"

for i in {0..99}; do
  oc create -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-$i
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  hostPath:
    path: "/srv/openshift/pv-$i"
EOF
done

oc create -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: manual
  annotations:
    storageclass.kubernetes.io/is-default-class: 'true'
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF
```

### LoadBalancer
You may wish to work on services that require the creation of Load Balancers. One option is to install MetalLB from Operator Hub.  
  
To configure MetalLB you need to provide a range of addresses and create the controller, after which it will handle any LB requests.  
  
In this example I installed it in the metallb namespace. If you installed it in a different namespace adjust the resources below accordingly.

### Provide MetalLB a range of addresses
Adjust the IP address CIDR or Range and create the config configmap. For more information see the [documentation](https://metallb.universe.tf/configuration/#layer-2-configuration).

```
cat << EOF | oc create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: config
  namespace: metallb
data:
  config: |
    address-pools:
    - protocol: layer2
      name: default
      addresses:
      - 192.168.1.176/28
EOF
```

### Create the MetalLB controller
```
cat << EOF | oc create -f -
apiVersion: metallb.io/v1beta1
kind: MetalLB
metadata:
  name: metallb
  namespace: metallb
EOF
```

### Shutting down, Rebooting, and Upgrades
Shutting down and rebooting your cluster should not present a problem.

Upgrades are supported as of SNO version 4.9.0. At the time of this blog the fast channel has an updated available up to version 4.9.5.

If my cluster was off for long periods of time the console would not come back up with version 4.9.0. So far I have not experienced this with 4.9.5, so I'd recommend upgrading after the initial install is complete.
