---
title: "Interchangeable Components with Go Interfaces"
date: 2021-07-27T10:38:38-04:00
draft: false
authors:
- Jason Montleon
tags:
- crane
- crane-lib
- golang
- interfaces
---

## Introduction
When MTC was initially released (as CAM) we supported only restic for PV migrations. As we tested ourselves and through the QE process this worked well for us and we felt it filled our needs so we released with just that option.

## The problems
Restic requires first backing up data to object storage like s3. This incurs an additional cost since everything must first be uploaded from the source and then downloaded to the destination. This can have benefits, especially in environments with restricted networking where the clusters cannot communicate with each other. Conversely downtime is often limited and lengthy migrations can be a huge burden. A direct migration with rsync might provide a large performance improvement. 

We also witnessed at least one case where data had become damaged on the backing store and rsync dealt with it more gracefully than restic, which simply froze and stopped transferring data. While direct volume migration was added to MTC it resulted in quite a bit of additional non-reusable code for this use case.

Furthermore, as of right now we also only support transfers wrapped in stunnel and exposed over routes. Using routes elliminates the ability to work with Kubernetes clusters, another capability we would like to add in the future. Mixing and matching transfer protocols, transports, and endpoint types would quickly become a convoluted tangle of code if we didn't do something else.

## The Solution
First, to deal with simplifying controller code we decided to move most of the logic for data migrations into a new library, [crane-lib](https://github.com/konveyor/crane-lib).

Next, when sitting down to think about the problem we broke out the components we are using for direct volume migrations today. Those are rsync, stunnel, and routes. Each of these has served us well, but also potentially has limitations. It would be nice if we could swap each out for another option as neeed. To do this we labeled each of these layers. rsync was is the transfer program. stunnel is the transport wrapper, and created an interface for each of these layers.

Transfer
```
type Transport interface {
	CA() *bytes.Buffer
	Crt() *bytes.Buffer
	Key() *bytes.Buffer
	Port() int32
	ClientContainers() []v1.Container
	ClientVolumes() []v1.Volume
	ServerContainers() []v1.Container
	ServerVolumes() []v1.Volume
	Direct() bool
	CreateServer(client.Client, endpoint.Endpoint) error
	CreateClient(client.Client, endpoint.Endpoint) error
}
```

Transport
```
type Transfer interface {
	Source() *rest.Config
	Destination() *rest.Config
	Endpoint() endpoint.Endpoint
	Transport() transport.Transport
	// TODO: define a more generic type for auth
	// for example some transfers could be authenticated by certificates instead of username/password
	Username() string
	Password() string
	CreateServer(client.Client) error
	CreateClient(client.Client) error
	PVCs() PVCPairList
}
```

Endpoint
```
type Endpoint interface {
	Create(client.Client) error
	Hostname() string
	Port() int32
	NamespacedName() types.NamespacedName
	Labels() map[string]string
	IsHealthy(c client.Client) (bool, error)
}
```

We then set off to create at least two implementations at each layer to prove that they were easily interchangeable and help us refine the interfaces. To do this we needed to ensure that each one implemented the interface.

For transfer rsync and rclone were implemented. For transport we implemented stunnel and null (there is no sense in wrapping applications that provide their own encryption) and for endpoints we added route and load balancer.

Through testing we developed a [Compatibility Matrix](https://github.com/konveyor/crane-lib/blob/main/state_transfer/README.md#compatibility-matrix) that verified our expectations while allowing us to resolve issues and tweak the interfaces.

Now when setting up each layer of our transfer we only need to be concerned with choosing the appropriate option, which is as simple as picking, for example 
`transfer, err := rclone.NewTransfer(s, r, srcCfg, destCfg, pvcList)` or `transfer, err := rsync.NewTransfer(s, r, srcCfg, destCfg, pvcList)`

## Refining the Interfaces
Once we started testing more deeply we realized we did not need and did not want setters for every parameter so we elliminated them so that we had only getters. Most all data is entered at creation and becomes immutable in this way. There is a burden to creating each function defined in the interface, even if they are relatively simple, so ellimating them eases development of future options. Ideally one should be able to copy an interface and adjust configs for the new component they would like to use at the given layer and be ready to go almost within minutes.

## Next Steps
At the moment we are focused on bringing crane-lib up to parity with the current direct volume migration code so we can use it in 1.6.0. In early testing the MTC controller code has become simpler and easy to follow. 

As we look forward to crane 2.0 more endpoint options such as nodeport, and even services, for intercluster transfers, and additional options for Kubernetes to OpenShift transfers seem like good candidates for enabling transfers within in the requirements of whatever network(s) the clusters are running in.

We would also welcome contributions improving or adding options to facilitate transfer within or between clusters.

## Links
[Konveyor Crane](https://www.konveyor.io/crane)  
[crane-lib](https://github.com/konveyor/crane-lib)  
[cran-lib example test](https://github.com/konveyor/crane-lib/blob/main/state_transfer/example_test.go)  
[crane 2.0](https://github.com/konveyor/crane)  
[Go by Example: Interfaces](https://gobyexample.com/interfaces)
