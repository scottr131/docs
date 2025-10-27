# Documentation

Various documentation for my different projects

## Hyperconverged Cluster

My process for building a fully hyperconverged (compute, storage, network) cluster on top of several pieces of open source software including Incus, Ceph, FRRouting, and OVN.

[Incus Hyperconverged Cluster](https://github.com/scottr131/docs/blob/main/hyperconverged-cluster/incus-hyperconverged-cluster.md)

## Ceph on Slackware

My experience building, installing, and configuring Ceph on Slackware Linux.

Building Ceph 20.1.0 and its depdendencies:
[Build Ceph on Slackware](https://github.com/scottr131/docs/blob/main/ceph-on-slackware/build-ceph-on-slackware.md)

Configuring Ceph for MON, MGR, OSD, MDS, RGW.
[Ceph on Slackware](https://github.com/scottr131/docs/blob/main/ceph-on-slackware/ceph-on-slackware.md)

## OpenStack on Slackware

A current project of mine, trying to run OpenStack on Slackware.  I have achieved a minimal working setup, so I know it's possible.  I'm documenting the full setup here.

Building OpenStack 2025.1 and its dependencies:
[Building OpenStack 2025.1 Epoxy on Slackware](https://github.com/scottr131/docs/blob/main/openstack-on-slackware/building-openstack-on-slackware.md)

Installation and Configuration:
[Journey to OpenStack on Slackware](https://github.com/scottr131/docs/blob/main/openstack-on-slackware/journey-to-openstack-on-slackware.md)

## Kubernetes on Slackware

Sometimes the workload calls for a different type of cluster.  While I consider this project in-progress, it works.  I documented the process of building the Kubernetes dependencies of components on Slackware, and then using those packages to create a vanilla K8s cluster.

Building Kubernetes (last tested with 1.34.1)
[Building Kubernetes on Slackware](https://github.com/scottr131/docs/blob/main/kubernetes-on-slackware/building-kubernetes-on-slackware.md)

Create Kubernetes Cluster on Slackware
[Kubernetes on Slackware](https://github.com/scottr131/docs/blob/main/kubernetes-on-slackware/kubernetes-on-slackware.md)

## Windows on KVM

This documents my process for creating a Windows PE ISO that has support for virtio devices.  This ISO can be used to install Windows in virtualization environments that use KVM.  I may add other documents in the future related for Windows running under the KVM hypervisor.

[Windows PE on KVM](https://github.com/scottr131/docs/blob/main/windows-on-kvm/winpe-for-kvm.md)

## Incus Cluster on Slackware

The process for a different Incus cluster I built.  This cluster uses Ansible for deployment of the cluster software onto emtpy Slackware nodes.  This cluster also uses a combination of LINSTOR, DRBD, and LVM for its cluster storage instead of Ceph.

[Incus Cluster On Slackware](https://github.com/scottr131/docs/blob/main/incus-on-slackware/incus-cluster-on-slackware.md)

## Incus by Example

Various "chapters" I'm collecting for some kind of Incus reference. 

[Incus by Example](https://github.com/scottr131/docs/tree/main/incus-by-example)
