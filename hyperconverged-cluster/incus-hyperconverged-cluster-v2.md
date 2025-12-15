> [!CAUTION]
> THIS DOCUMENT IS CURRENTLY INCOMPLETE.
> Not all steps are listed.  Following these steps
> is not sufficient to have a working cluster.
> This document should be updated soon.

This is the second major iteration of my hyperconverged cluster.  The goals of this version include adding automation to build and deployment process as well as adding some monitoring and metrics to the cluster.
The cluster hardware consists of 3 PCs with dual port 10Gb ethernet network adapters. The exact model doesn't matter too much - the PCs in this example are older HP small desktops with 6th Gen Intel Core i3 CPUs and 16GiB of RAM. The important part is that the motherboard and CPU support hardware virtualization and IOMMU. (Actually, IOMMU is not really a strict requirement.) Each PC has a 256GB NVMe and a 500GB SATA SSD. These are not super powerful machines, but they should allow some VMs and containers to run for testing.
The networking is set up in a way that eliminates the need for a 10GbE switch and allows for the use of cheaper DAC cables. Each PC has three network interfaces - the internal gigabit ethernet adapter and the dual port 10Gb ethernet card that appears as two network adapters to the OS.
The gigabit adapters connect to a network switch to allow management of the node as well connect the node to outside world. VLAN traffic isolation can be implemented here, which would require a managed switch. For testing purposes with no traffic isolation an unmanaged switch works fine. For the purposes of this document this is called the "management LAN." The 10Gb ethernet adapters are linked to each other directly with DAC cables. Each node is connected to the other two nodes - forming a triangle arrangement. The interfaces are not automatically configured so it is important that the DAC cables are connected to right ports. PC 1 port 1 goes to PC 2 port 2 - this is link 1. PC 2 port 1 goes to PC 3 port 1 - this is link 2. PC 3 port 2 goes to PC 1 port 2 - this is link 3. For the purposes of this document the combination of these links is called the "cluster LAN."
In addition to the main cluster nodes, a fourth independent node is needed.  This node will be referred to as the “build node” and used to compile all the software packages that are not included in with Slackware.  Therefore, this node should have a powerful CPU as well as lots of RAM and storage.  32GB of RAM and 500GB of storage is recommended to build Ceph.  This node connects to the gigabit ethernet “management LAN.”

# Node Software

## Operating System (Slackware)

The nodes will run Slackware Linux -current.  This is the development branch of Slackware so packages are updated often.  Despite being the development branch, it is nearly as reliable as the release version (currently 15).

## Lower Networking Layer (Linux bridge with STP)

Since the high-speed network consists of three independent links, we need a way to join those together. There are several ways to do this, this may not be the best way. To keep things simple in this iteration of the cluster, the gigabit interfaces on each PC will be bridged and STP enabled.  Then, each node can be connected to the other two, and STP will resolve the loop.

## Hypervisor Layer (Qemu / KVM / LXC)

The standard Slackware kernel provides KVM virtualization support. Qemu will fill out the rest of the device emulation and management to provide a full VM. While not really a hypervisor, LXC is used here because Linux Containers (LXC) will be used to run containers. LXC support is included with Slackware.  These hypervisor components will be controlled by the orchestration layer.

## Software Defined Networking (SDN) Layer (Open vSwitch / OVN)

Virtual machines and containers can be bridged to the management LAN, but for more flexibility, the cluster will provide software-defined networks. This networking layer is provided via OVN that runs on top Open vSwitch. This SDN layer then operates on top of the 'Lower Networking Layer' mentioned earlier.

## Distributed Storage Layer (LINSTOR over LVM)

To allow for live migration of running virtual machines the cluster will need a shared storage layer. For this cluster, we'll implement this using LINSTOR. Each node will devote some of its storage toward this shared storage pool. Each node will have a thinly allocated LVM pool that will be joined with LINSTOR to the distributed storage pool.

## Orchestration Layer (Incus)

Incus will be used to manage all these components. Incus can run VMs using QEMU/KVM and run both LXC and Docker (OCI) containers. Incus allows all cluster nodes to be managed from single interface via the command-line utility or an API.

## Monitoring Layer (Loki, Prometheus, Grafana)

With 3 nodes and multiple VMs it is important to monitor what's going on. On each node, Incus can provide its metrics and logs to Prometheus and Loki respectively. Grafana can then be used to visualize trends.

## “Build Node” Software Installation

We'll install `slackware64-current` on the node.  Since that is the development release, it’s best to download a copy of it using `rsync` to ensure all nodes use a consistent copy for their installation source.  I have documented my standard method for installing Slackware elsewhere, so refer to that for installation details.  I’ll use custom tagfiles to make sure only needed packages are installed.  Since this is the node that will compile software, make sure to use the mini-dev version of my tagfiles.

#### User Account

Once Slackware is installed, login as `root`. Let’s manage the cluster with an account called `clusteradm`.  To keep things consistent on all nodes, create that account with a UID of 1050 and a member of the users group.  Also make the clusteradm account a member of the wheel group.  This will allow sudo access in the future after sudo is configured.

```sh
# Run as root since sudo isn't configured
useradd -d /home/clusteradm -g users -u 1050 -m -s /bin/bash clusteradm 
usermod -a -G wheel clusteradm
passwd clusteradm
# You will be prompted for the password
```

If you want to enable key based SSH authentication, you can make an `.ssh` directory and add your key. 

```sh
mkdir /home/clusteradm/.ssh
echo "<your-public-key>" > /home/clusteradm/.ssh/authorized_keys
chown -R clusteradm:users /home/clusteradm/.ssh
chmod 600 /home/clusteradm/.ssh
chmod 644 /home/clusteradm/.ssh/authorized_keys
```

#### NTP

Edit /etc/ntp.conf and uncomment one of the ntp.org servers or add a custom server.

```
# NTP server (list one or more) to synchronize with:
#server 0.pool.ntp.org iburst
server 1.pool.ntp.org iburst
#server 2.pool.ntp.org iburst
#server 3.pool.ntp.org iburst
```

> [!WARNING]
> Don’t skip this step.  An incorrect or changing system time
> on the build node can cause builds to fail.  In addition,
> the build node’s system clock needs to be synchronized
> with the cluster nodes.  The easiest way to do this is to
> synchronize them all to the same NTP server.

#### SSH

By default, Slackware allows non-root users to log in via SSH with simple password authentication.  If you want to disable that, edit `/etc/sshd.conf`.

```text
# Around line 57 of /etc/sshd.conf
# To disable tunneled clear text passwords, change to "no" here!
PasswordAuthentication no
```

#### Sudo

Finally, we need to stop using the root account as much as possible, so let's make sure sudo is configured.  Using `visudo`, uncommenting the line that allows for the `wheel` group to have sudo access.  In addition, if for some reason you have additional packages installed in /opt that need to run as root, you can add that path here.  Let’s add /opt/java17/bin and /opt/go/bin so they will be available for sudo use later.  This is needed because the Slackbuilds scripts are intended to be run as `root`.

```text
# ... usually around line 51
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/java17/bin:/opt/go/bin"
# ... usually around line 128 - remove # from line
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
```

The rest of this document assumes you are logged in as a regular user (clusteradm) and will run root commands with `sudo`.

### Clone Build Scripts

Most of the software builds will be done using Jenkins.  Before Jenkins can be used, a JDK and Go will need to be installed onto the build system.  This can be done by building those components from my build scripts and installing the resulting packages.

```
git clone https://github.com/scottr131/slackbuilds.git
cd slackbuilds
```

### Package and Install JDK and Go

A JDK is required to build Jenkins.  This also seems like a good time to install Go as many of packages require it.  The packages can be built using the included Makefile and then installed.

```
make temuriun-jdk17
make go
sudo installpkg /tmp/packages/termurin-jdk17*.txz
sudo installpkg /tmp/packages/go*.txz
```

### Add JDK and go to sudo path

Slackbuild scripts are designed to run as root.  In this case, my Makefile calls the scripts via sudo.  Therefore, we need to update the secure sudo path to include the JDK path.

```
visudo
## around line 51, modify this line
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/java17/bin:/opt/go/bin"
```

### Start Jenkins

Jenkins will be used to execute the build pipelines for all the additional software that needs compiled from source. Jenkins will use Slackbuild scripts cloned from my GitHub.

```
export JAVA_HOME=/opt/java17
/opt/java17/bin/java -jar jenkins.war
```

### Start Rundeck

Rundeck will be used to run the Ansible playbooks.  This isn't strictly necessary, as it can be done with just Ansible, but this adds a nice web UI.

```
export RDECK_BASE=$HOME/rundeck;
mkdir -p $RDECK_BASE
cp /opt/rundeck/rundeck-5.17.0-SNAPSHOT.war $RDECK_BASE
cd $RDECK_BASE
# Start Rundeck to initialize the $RDECK_BASE directory
java -Xmx4g -jar rundeck-5.17.0-SNAPSHOT.war
```

When the following message appears, Rundeck has completed the initialization.  Press Ctrl+C to exit Rundeck.

```
Grails application running at http://localhost:4440 in environment: production
```
Now we need to add the Rundeck binaries and man pages to the path.
```
PATH=$PATH:$RDECK_BASE/tools/bin
MANPATH=$MANPATH:$RDECK_BASE/docs/man
```
Then we can restart Rundeck.  Do this in a tmux session so you can disconnect and leave it running.  We'll set up Rundeck properly later.
```
java -Xmx4g -jar rundeck-5.17.0-SNAPSHOT.war
```

## Rundeck 

### Add Inventory
Create a nodes.yaml for Rundeck to use as its inventory.
```
localhost:
  nodename: localhost
  hostname: localhost
  description: Rundeck server node
  tags: ''
tnode3:
  nodename: tnode3
  hostname: tnode3.i131.net
  tags: clusternode
tnode2:
  nodename: tnode2
  hostname: tnode2.i131.net
  tags: clusternode
tnode1:
  nodename: tnode1
  hostname: tnode1.i131.net
  tags: clusternode
```
### Prepare Jobs

### Pull Config Files from GitHub

### Populate Ansible Files
```
# LINSTOR
mkdir -p ~/ansible/linstor/rc.d
mkdir -p ~/ansible/linstor/default
cp ~/linux/slackware/etc/rc.d/rc.linstor-satellite ~/ansible/linstor/rc.d/
cp ~/linux/slackware/etc/rc.d/rc.linstor-controller ~/ansible/linstor/rc.d/
cp ~/linux/slackware/etc/rc.d/rc.zfs ~/ansible/linstor/rc.d/
cp ~/linux/slackware/etc/rc.d/rc.drbd ~/ansible/linstor/rc.d/
cp ~/linux/slackware/etc/default/linstor-satellite ~/ansible/linstor/default/
cp ~/linux/slackware/etc/default/linstor-controller ~/ansible/linstor/default/
cp ~/linux/slackware/etc/default/zfs ~/ansible/linstor/default/


# Incus
mkdir -p ~/ansible/incus/rc.d
mkdir -p ~/ansible/incus/default
cp ~/linux/slackware/etc/rc.d/rc.incusd ~/ansible/incus/rc.d/
cp ~/linux/slackware/etc/default/incus ~/ansible/incus/default/


# Local
mkdir -p ~/ansible/local-config/rc.d
mkdir -p ~/ansible/local-config/default
cp ~/linux/slackware/etc/rc.d/rc.local ~/ansible/local-config/rc.d/
cp ~/linux/slackware/etc/rc.d/rc.local_shutdown ~/ansible/local-config/rc.d/


# OVN/Open vSwitch
mkdir -p ~/ansible/ovn/rc.d
mkdir -p ~/ansible/ovn/default
cp ~/linux/slackware/etc/rc.d/rc.openvswitch ~/ansible/ovn/rc.d/
cp ~/linux/slackware/etc/rc.d/rc.ovn-central ~/ansible/ovn/rc.d/
cp ~/linux/slackware/etc/rc.d/rc.ovn-host ~/ansible/ovn/rc.d/
cp ~/linux/slackware/etc/default/openvswitch ~/ansible/ovn/default/
cp ~/linux/slackware/etc/default/ovn-central ~/ansible/ovn/default/
cp ~/linux/slackware/etc/default/ovn-host ~/ansible/ovn/default/
```



## Node Software Installation

### Operating System

We'll install `slackware64-current` on the node.  Since that is the development release, I'll download a copy of it using `rsync` to make sure I have a consistent copy to install all the nodes from.  I have documented my standard method for installing Slackware elsewhere, so refer to that for installation details.  

#### Partitioning

The OS is installed to the SATA SSD. The SATA SSD is partitioned as such:

```
Device      Start       End   Sectors  Size Type
/dev/sda1    2048    526335    524288  256M EFI System
/dev/sda2  526336 537397247 536870912  256G Linux filesystem
```

The first partition is used for UEFI, the second partition is the Slackware root partition (there is no swap on these systems for now), the third partition will be for local VM storage (mainly the OSD VM), and the fourth partition will be used for later experimentation and testing.   

#### Slackware Software Sets
