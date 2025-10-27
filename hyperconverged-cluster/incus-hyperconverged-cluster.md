# Incus Cluster with Point-to-Point Links

## Hyperconverged

The cluster consists of 3 PCs with dual-port 10Gb ethernet network adapters.  The exact model doesn't matter too much - the PCs in this example are older HP small desktops with 6th Gen Intel Core i3 CPUs and 16GiB of RAM.  The important part is that the motherboard and CPU support hardware virtualization and IOMMU.  (Actually, IOMMU is not really a strict requirement.)  Each PC has a 256GB NVMe and a 500GB SATA SSD.  These are not super powerful machines but they should allow some VMs and containers to run for testing.

The networking is set up in a way that elimiantes the need for a 10GbE switch and allows for the use of cheaper DAC cables.  Each PC has three network interfaces - the internal gigabit ethernet adapter and the dual port 10Gb ethernet card that appears as two network adapters to the OS.  
The gigabit adapters connect to a network switch to allow management of the node as well connect the node to outside world.  VLAN traffic isolation can be implemented here, which would require a managed swtich.  For testing purposes with no traffic isolation an unmanaged switch works fine.  For the purposes of this document this is called the "management LAN."
The 10Gb ethernet adapters are linked to each other directly with DAC cables.  Each node is connected to the other two nodes - forming a triangle arrangement.  The interfaces are not automatically configured so it is important the DAC cables are connected to right ports.  PC 1 port 1 goes to PC 2 port 2 - this is link 1.  PC 2 port 1 goes to PC 3 port 1 - this is link 2.  PC 3 port 2 goes to PC 1 port 2 - this is link 3.  For the purposes of this document the combination of these links is called the "cluster LAN."

## Node Software

### Operating System (Slackware)

The nodes will run Slackware Linux.  I chose this because I'm comfortable with the OS but its lack of prebuilt packages and dependency management means I get to learn a little bit more about the software components than I normally would.  In this document I will only cover configuration of the software.  I have other documents on GitHub that cover compiling and packaging the software components for Slackware.

### Lower Routing Layer (FRRouting)

Since the high speed network consists of three independent links, we need a way to join those together.  There are several ways to do this, this may not be the best way.  This cluster will use FRRouting with IP forwarding on the nodes to give each node a stable IP address with a redundant path in case of a link failure.

### Hypervisor Layer (Qemu / KVM / LXC)

Standard Linux KVM will provide the hypervisor with Qemu filling out the rest of the VM emulation.  The standard Slackware kernel provides KVM support.  Qemu will fill out the rest of the device emulation and management to provide a full VM.  While not really a hypervisor, LXC is used here because Linux Containers (LXC) will be used to run containers.  LXC support is included with Slackware.

### Software Defined Networking (SDN) Layer (Open vSwitch / OVN)

Virtual machines and containers can be bridged to the management LAN, but for more flexibility, the cluster will provide software defined networks.  This networking layer is provided via OVN that runs on top Open vSwitch.  This SDN layer then operates on top of the 'Lower Routing Layer' mentioned earlier.

### Distributed Storage Layer (Ceph)

To allow for live migration of running virtual machines the cluster will need a shared storage layer.  For this cluster, we'll implement this using Ceph.  Each node will devote some of its compute and storage for this shared storage.  Ceph provides block, file, and object storage to the cluster nodes and workloads.  Each cluster node will be a Ceph monitor and manager.  Each cluster node will also run a VM connected to the nvme device to provide a Ceph OSD.  There will be 3 hardware nodes and 3 VMs in the Ceph cluster.

### Orchestration Layer (Incus)

Incus will be used to manage all these components.  Incus can run VMs using QEMU/KVM and run both LXC and Docker (OCI) containers.  Incus allows all cluster nodes to be managed from single interface via the command-line utility or an API.

### Monitoring Layer (Zabbix)

With 3 nodes and multiple VMs it is important to monitor what's going on.  The Zabbix server will run on a virtual machine and mointor an agent that runs on each cluster member (VM or hardware).

## Node Software Installation

### Operating System

We'll install `slackware64-current` on the node.  Since that is the development release, I'll download a copy of it using `rsync` to make sure I have a consistent copy to install all the nodes from.  I have documented my standard method for installing Slackware elsewhere, so refer to that for installation details.  

#### Partitioning

The OS is installed to the SATA SSD so that the entire nvme SSD can be passed through to a VM for the Ceph OSD.  The SATA SSD is partitioned as such:

```
Device         Start       End   Sectors  Size Type
/dev/sda1       2048    264191    262144  128M EFI System
/dev/sda2     264192  67373055  67108864   32G Linux filesystem
/dev/sda3   67373056 134481919  67108864   32G Linux filesystem
/dev/sda4  134481920 973342719 838860800  400G Linux filesystem
```

The first partition is used for UEFI, the second partition is the Slackware root partition (there is no swap on these systems for now), the third partition will be for local VM storage (mainly the OSD VM), and the fourth partition will be used for later experimentation and testing.   

#### Slackware Software Sets

I’ll only install software sets A, AP, L, N, TCL, and X. I’m not sure I’ll need X, but a minimal GUI on these systems may be useful later.  Since we're not installing the D software set, there are a couple of things we need to do.
Without the D software set, the system doesn't have Python or Perl.  Some of Slackware scripts (especially related to certificate management) assume Perl is available, so the easiest approach is to install it.  In addition, some of the software components are in Python - so we need to make sure Python is available.  Let's go ahead and manually install these pacakges from the D set.

```sh
installpkg perl-5.42.0-x86_64-1.txz
installpkg python3-3.12.11-x86_64-1.txz \
  python-pip-25.2-x86_64-1.txz \
  python-setuptools-80.9.0-x86_64-1.txz
```

Also - if Perl was not present during the Slackware install process (i.e. you didn't pick the D software set), then the CA certificates are not properly installed.  Now that Perl is available, let's fix that.

```sh
update-ca-certificates --fresh
```

Finally, a counterpoint.  If the D software set was installed, then there may be software packages installed that can cause problems - GCC Go and GCC Rust.

```sh
# Only needed if all of D was installed, won't hurt if it wasn't
removepkg gcc-go-15.2.0-x86_64-1.txz
removepkg gcc-rust-15.2.0-x86_64-1.txz
```

#### Initial Network Configuration

For now, it should be sufficient to assign an IP address to the gigabit ethernet interface or configure it for DHCP.  Static IP addresses will be configured to the 10 gigabit interfaces later.

#### NTPd

Time synchronization is important for a cluster and seems to be something I always forget to configure.  Edit `/etc/ntpd.conf` and add a time server now.  Make sure NTPd is enabled, as it's not by default.

```sh
cmhod +x /etc/rc.d/rc.ntpd
```

#### User Account

To keep things consistent, let's manage the cluster with an account called `clusteradm`.  To keep things consistent on all nodes, let's create that account will have UID 1050 and be a member of the users group.  Also make the clusteradm account a member of the wheel group.  This will allow sudo access in the future after sudo is configured.

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

#### SSH

By default, Slackware allows non-root users to log in via SSH with simple password authentication.  If you want to disable that, edit `/etc/sshd.conf`.

```text
# Around line 57 of /etc/sshd.conf
# To disable tunneled clear text passwords, change to "no" here!
PasswordAuthentication no
```

#### Sudo

Finally we need to stop using the root account as much as possible, so let's make sure sudo is configured.  Using `visudo`, uncommenting the line that allows for the `wheel` group to have sudo access.  In addition, if for some reason you have additional packages installed in /opt that need to run as root, you can add that path here.

```text
# Around line 51
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
# Around line 128 - remove # from below
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
```


### Startup Scripts
I have created a set of startup scripts and configuration files for the various pieces of software used here.  There aren't currently included with my SlackBuilds packages, so install them from GitHub now.  I have more scripts on GitHub than we need here, so I'll only copy the ones needed.  Make sure the scripts aren't set executable because most of the services are not configured yet.

```sh
git clone https://github.com/scottr131/linux --depth=1
chmod -x linux/slackware/etc/rc.d/*

# FRRouting
cp linux/slackware/etc/default/frr /etc/default/
cp linux/slackware/etc/rc.d/rc.frr /etc/rc.d/

# OVN / OVS 
cp linux/slackware/etc/default /etc/default/
cp linux/slackware/etc/rc.d/rc.openvswitch \
	linux/slackware/etc/rc.d/rc.ovn-central \
    linux/slackware/etc/rc.d/rc.ovn-host \
    /etc/rc.d/

# Ceph
cp linux/slackware/etc/default/ceph \
	linux/slackware/etc/default/ceph-mgr \
    linux/slackware/etc/default/ceph-mon \
	linux/slackware/etc/default/ceph-osd \
    /etc/rc.d/
# Need MDS RGW
cp linux/slackware/etc/rc.d/rc.ceph-mgr \
	linux/slackware/etc/rc.d/rc.ceph-mon \
    linux/slackware/etc/rc.d/rc.ceph-osd \
    /etc/rc.d/
# Need MDS RGW

# Incus
cp linux/slackware/etc/default/incus /etc/default/
cp linux/slackware/etc/rc.d/rc.incusd /etc/rc.d/

# Zabbix
cp linux/slackware/etc/default/zabbix-server \
	linux/slackware/etc/default/zabbix-agent \
    /etc/default/
cp linux/slackware/etc/rc.d/rc.zabbix-server \
	linux/slackware/etc/rc.d/rc.zabbix-agent \
    /etc/rc.d/
```

### FRRouting

FRRouting only has a couple dependencies - protobuf-c and libyang.  I have SlackBuilds packages for these that can be installed.

```sh
installpkg protobuf-c-1.5.2-x86_64-1_SBo.txz \
	libyang-3.13.5-x86_64-1_SBo.txz \
    frr-10.4.1-x86_64-1_SBo.txz
```

### Open vSwitch / OVN
OVN and Open vSwitch work together so let's install them together.  If FRR is installed, the system should already had protobuf-c and libyang, but I have included those here as well as they are needed by OVN.

```sh
installpkg openvswitch-3.6.0-x86_64-1_SBo.txz \
	ovn-25.03.1-x86_64-1_SBo \
	libyang-3.13.5-x86_64-1_SBo.txz \
    protobuf-c-1.5.2-x86_64-1_SBo.txz
```

### Ceph
Ceph is a complex piece of software, so it has a lot of dependencies that need to be installed.  First, some additional packages from the D software set are required.

```sh
# GNU Fortran library, Required by SciPy
installpkg gcc-gfortran-15.2.0-x86_64-1.txz
# LUA library, Required by RADOSGW
installpkg lua-5.4.8-x86_64-1.txz
```

