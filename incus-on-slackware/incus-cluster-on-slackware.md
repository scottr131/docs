# Incus Cluster on Slackware

This is my journey of setting up an Incus cluster on Slackware-current.  Incus will provide a cloud-like service for running containers and virtual machines.  The cluster will have shared storage and virtual networking.  A virtual router will provide connectivity between the cluster, a LAN, and a WAN.  A managed ethernet switch can break out these networks into physical connections.
This is not my first time on this journey.  I have had many stops, starts, and tangents along the way.  I had to apply some working knowledge, learn some new technologies, and even get a refresher on a few skills I hadn't used in a while.  This is a compilation of what I learned, and the process I used to get a working cluster. 
I have used Slackware (among other distros) for a long time.  Since I want to build many of the components from source to make sure I have the latest version, I figured Slackware would make an excellent base since it should let me compile most packages without too much deviation from their default configuration.
This whole journey will be an exercise using many popular open-source tools and popular technologies.  Some things may be overly complex or completely unneeded, but that's fine since the idea is to be able to use several tools in a single cluster.

## The Hardware

I plan to build the cluster from four nodes.  Three nodes will make up the cluster, and one node will be the build/deploy node.  The three cluster nodes should be identical so running virtual machines can be live-migrated from node to node.  My nodes are using 6th generation Intel Core i5 CPUs with 16GB of RAM.  This is probably the very minimum configuration I would recommend.  The build/deploy node can be different.  It can also be a virtual machine.  Since this machine will compile the software, more RAM and CPU cores will be advantageous.
The storage and networking are simple for this proof-of-concept / lab-grade cluster but could be based on much faster hardware.  I will be using the built-in gigabit ethernet as the cluster and storage network.  This network can be strictly internal to the cluster and is used to intra-cluster communication and storage.  For the OVN uplink network, I will be using a USB connected gigabit ethernet adapter on each node.  This network may be slower and less reliable due to the USB connection; I think it will have less impact on the uplink network than the storage network.

## The Software

The software stack will consist of Slackware Linux as the base OS.  Incus will be installed on top of that, along with Qemu for KVM virtual machines.  For the networking layer, Open Virtual Networking (OVN) will be used; OVN, in turn, uses Open vSwitch.  For storage, I will use OpenZFS on top of LINSTOR and DRBD.  These packages will provide a shared networking and storage layer across the cluster.

## The Process

First, I will set up the build/deploy node since that will be used to bootstrap everything else.  Once the build/deploy node is set up, I can download the Slackbuilds scripts I have created previously and use those to compile all the (additional) software needed for the cluster.
I will install many of the packages on the build/deploy host after I build them.  That will leave me with a functional Incus system on the build/deploy host.  I will configure the build/deploy node as a standalone Incus host. 
I will create an OpenWrt VM in Incus on the build/deploy host.  This will serve as the cluster router VM.  All traffic into and out of the cluster will pass through this VM (eventually).  The VM will start its life on the build/deploy host and will be moved to the cluster once it has been created.

## Building the Cluster

First, I’ll start with the build/deploy node.  I’ll install Slackware64-current with software sets A, AP, D, K, L, N, TCL, and X.  This won't make a system with a very useable GUI but should give all the support libraries needed to build the additional software on.  For this system, I’ll include some swap space and make the root partition take the entire drive (except for the UEFI partition of course).  For networking, I’ll connect `eth0` to my LAN and configure the system for DHCP.  I’ll take care of the rest of the network configuration later.  Once Slackware is installed, I’ll login as root for the first time and configure the system.

### Configure the user

Logged in as root, I need to add a user account.  After I have a user account, I'll be able to access the build/deploy node via SSH.  I'll use `useradd` to add a user called `clusteradm` that will be used for Ansible and general cluster administration.  I’ll also add this user to `wheel` group which I later use to grant sudo access. 

```text
root@deploy:~# useradd -d /home/clusteradm -g users -m -s /bin/bash clusteradm
root@deploy:~# usermod -a -G wheel clusteradm
root@deploy:~# passwd clusteradm
New password:
Retype new password:
passwd: password updated successfully
```

I need this user to have sudo access.  I do this by allowing the `wheel` group to have sudo access.  I'll run the `visudo` command to edit `/etc/sudoers`.  Around line 125, there is a template for allowing members of group wheel to execute any command.  I’ll uncomment that line to allow sudo access for the wheel group.

```text
## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL:ALL) ALL
```

I can now log in as the `clusteradm` user for this rest of this document (unless otherwise noted).  I'll use this user for general cluster administration, and I'll also use it later for Ansible.

### Configure Networking

The build/deploy node has two network interfaces.  The internal ethernet interface (eth0) is connected to my local LAN and has DHCP enabled.  The second ethernet interface (eth1) is a USB ethernet adapter and will connect to cluster network.  The cluster nodes will be connected to the cluster network with their eth0 interface.

> [!NOTE]
> The network connections on the build/deploy node
> and the compute nodes are “opposite.”  That is,
> eth1 of the build/deploy node is connected to
> the same switch as eth0 of the cluster nodes.

I'll also want to create a bridge for each interface in case I want to attach VMs to them later.  In Slackware, this is done by editing `/etc/rc.d/rc.inet1.conf`.

```
...
# These options are set for interface [0] and [1].
# All other options are cleared.
IFNAME[0]="lan-br"
BRNICS[0]="eth0"
USE_DHCP[0]="yes"
IFNAME[1]="cluster-br"
BRNICS[1]="eth1"
IPADDRS[1]="172.31.254.10/24"
...
```

> [!WARNING]
> These changes won't take effect until the system
> is restarted.  There are ways around this, but
> I'll take the few seconds to reboot now.

### Build the software

Now, I need to clone my repository of SlackBuilds.  This will give me scripts to build all the software needed for the Incus cluster.

```text
clusteradm@deploy:~$ git clone https://github.com/scottr131/slackbuilds --depth=1
Cloning into 'slackbuilds'...
remote: Enumerating objects: 124, done.
remote: Counting objects: 100% (124/124), done.
remote: Compressing objects: 100% (115/115), done.
remote: Total 124 (delta 36), reused 90 (delta 9), pack-reused 0 (from 0)
Receiving objects: 100% (124/124), 29.35 KiB | 4.89 MiB/s, done.
Resolving deltas: 100% (36/36), done.
```

An individual package can be built by changing into its directory and running the appropriate Slackbuild command.  SlackBuilds are meant to be run as root, hence the use of sudo.

```bash
OUTPUT=~/slackbuilds/packages/ PKGTYPE=txz BLDTHREADS=8 sudo ./usbredir.Slackbuild
wget --directory-prefix=usbredir $DOWNLOAD_x86_64
```

### Prepare Ansible

I’ll need to install Ansible on the build/deploy node. I don’t currently have a Slackbuild script for this, so I’ll install it from `pip`.

```bash
pip install ansible
```

Next I need to grab my Ansible playbooks from GitHub.

```bash
cd ~
git clone https://github.com/scottr131/ansible --depth=1
```

## Deploy Standalone Incus

Now I need to install Incus to the build/deploy node.  I'll do this so that I can have a standalone node I can use to transfer images and instances in to and out of the cluster.  Since I'll use Ansible to deploy the three main nodes, I might as well use it for deployment here too.

I'll grab my Ansible playbooks from GitHub.

```
cd ~
git clone https://github.com/scottr131/ansible --depth=1
```

I'll create a temporary `hosts.ini` with just the build/deploy node in it.  My Ansible playbooks are configured to deploy to a group called `nodes`. 

```ini
[nodes]
deploy.ic1.local
```

Now I can run Ansible playbooks against the build/deploy node.  I'll start by deploying QEMU and Incus in order to be able to run VM instances.

```bash
ansible-playbook -i hosts.ini -b -K qemu/qemu-on-slackware.yaml
ansible-playbook -i hosts.ini -b -K incus/incus-groups.yml
ansible-playbook -i hosts.ini -b -K incus/incus-on-slackware.yaml
```

Before starting Incus for the first time, it's probably a good idea to set up networking on the build/deploy node.  The build/deploy node will be connected to the Internet via my LAN on eth0, and then will have a USB ethernet adapter as eth1 that will be connected to the cluster network.  I'll also want to create a bridge In Slackware, this is done by editing `/etc/rc.d/rc.inet1.conf`.

```
...
# These options are set for interface [0] and [1].
# All other options are cleared.
IFNAME[0]="lan-br"
BRNICS[0]="eth0"
USE_DHCP[0]="yes"
IFNAME[1]="cluster-br"
BRNICS[1]="eth1"
IPADDRS[1]="172.31.254.10/24"
...
```

Once that is complete, I'll need to **reboot the build/deploy node**.  This allows two changes to take place.  My Incus playbook changes the CGroups version, and I just changed the networking configuration.  Both these changes will be effective on reboot.

> [!TIP]
> Check `/etc/udev/rules.d/70-persistent-net.rules` on Slackware
> to see which physical interface is associated with each interface name.
