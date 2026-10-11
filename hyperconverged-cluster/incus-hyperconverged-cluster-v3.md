# Incus Hyperconverged Cluster
Version 3.0

> [!CAUTION]
> THIS DOCUMENT IS CURRENTLY INCOMPLETE.
> Not all steps are listed.  Following these steps
> is not sufficient to have a working cluster.
> This document should be updated soon.

This is the third major iteration of my hyperconverged cluster. It includes lessons learned from the first two versions as well from my tri-plane-infrastructure. This version focuses on a small, efficient 3-node cluster running on commodity mini-PCs. The PCs in this example are 7th Gen Intel Core i5 CPUs with 32GiB of RAM. Unlike the 10Gbit node-to-node network used in the previous cluster, this cluster uses bonded gigabit ethernet links in an attempt to keep cost down. The nodes will operate on a private VLAN called the "cluster LAN". The storage network is currently implemented as a separate VLAN, but could be separate physical network for perfomance reasons. The LAN (or customer LAN) is the end-user network. Additional networks may be added.

In addition to removing the need for 10GB networking, these instructions are structured in a way that a separate build node will not be needed. The entire build process can be completed on three hardware nodes.

## Hardware Components
## Hardware
- (3) PCs, 256GB NVMe, 1TB SSD, 32GB RAM
- (6) patch cords
- 24-port managed ethernet switch
- (3) USB 3.0 Ethernet Adapters
- USB 3.0 Flash Drive - 8GB+, if formatted ext4/exFAT/FAT32 does not need reformatted (Slackware Files USB Drive)
- USB Flash Drive - any size, will be erased (Slackware Boot USB Drive)

## Software Components
### Operating System (Slackware)

The nodes will run Slackware Linux -current.  This is the development branch of Slackware so packages are updated often.  Despite being the development branch, it is nearly as reliable as the release version (currently 15). 

### Lower Networking Layer (bonded ethernet links )

Each node has an internal gigabit ethernet adapter and a USB 3.0 attached gigabit ethernet adapter. These links are bonded for redundancy and some performance increase. Multiple VLANs run across the bonded link to separate the cluster LAN, the customer LAN, and the storage LAN.

### Hypervisor Layer (Qemu / KVM / LXC)

The standard Slackware kernel provides KVM virtualization support. Qemu will fill out the rest of the device emulation and management to provide a full VM. While not really a hypervisor, LXC is used here because Linux Containers (LXC) will be used to run containers. LXC support is included with Slackware.  These hypervisor components will be controlled by the orchestration layer.

### Software Defined Networking (SDN) Layer (Open vSwitch / OVN)

Virtual machines and containers can be bridged to the management LAN, but for more flexibility, the cluster will provide software-defined networks. This networking layer is provided via OVN that runs on top Open vSwitch. This SDN layer then operates on top of the 'Lower Networking Layer' mentioned earlier.

### Distributed Storage Layer (LINSTOR over LVM)

To allow for live migration of running virtual machines the cluster will need a shared storage layer. For this cluster, we'll implement this using LINSTOR. Each node will devote some of its storage toward this shared storage pool. Each node will have a thinly allocated LVM pool that will be joined with LINSTOR to the distributed storage pool.

### Orchestration Layer (Incus)

Incus will be used to manage all these components. Incus can run VMs using QEMU/KVM and run both LXC and Docker (OCI) containers. Incus allows all cluster nodes to be managed from single interface via the command-line utility or an API.

### Monitoring Layer (Loki, Prometheus, Grafana)

With 3 nodes and multiple VMs it is important to monitor what's going on. On each node, Incus can provide its metrics and logs to Prometheus and Loki respectively. Grafana can then be used to visualize trends.

## Build and Install

### Bootstrapping
The first step is create install media. This will have to be done from another computer that's already up and running another operating system such a Linux, Windows, or MacOS. Currently, this document shows steps for creating the flash drive on another Linux computer.

#### Create Static Mirror USB

Obtain a USB flash drive with at least 7GiB free space, and formatted with NTFS, FAT32, exFAT, or ext2/3/4. Other filesystems may work but are untested. For these examples, it is assumed the partition of the flash drive is `/dev/sdc1`. Change the path accordingly.

#### Initial Slackware Download

Next, create a static image of the Slackware-current mirror. Since the mirror updates as much as daily, a stable local static copy is needed. Slackware can be downloaded with rsync on another Linux computer.

On Linux:
```bash
sudo mkdir -p /mnt/usb
sudo mount /dev/sdc1 /mnt/usb
sudo rsync -havP --delete --delete-after  --no-o --no-g --safe-links  --timeout=60 --contimeout=30 --exclude "EFI/*" --exclude "pasture/*" --exclude "patches/*"  --exclude "source/*"  --exclude "testing/*" rsync://mirrors.kernel.org/slackware/slackware64-current /mnt/usb
```

#### Slackware Installer USB

Since there is no infrastructure at this point, create a USB flash drive to use to install Slackware onto the build node. This is a different USB flash drive, and must be at least 64MB in size. Write `slackware64-current/usb-and-pxe-installers/usbboot.img` to a USB drive. On a Unix-like OS, this can be done with `dd`. On Windows, this should probably be done with a program like Rufus.

```bash
mount /dev/sdc1 /mnt/usb
dd if=/mnt/usb/slackware64-current/usb-and-pxe-installers/usbboot.img of=/dev/sdd status=progress bs=4096
```

### Build Node

Now, boot the installer USB on the build node. The process for installing Slackware is briefly documented here, but more thorough documentation is available elsewhere. 

The Slackware installer USB boots to GRUB, choose the `nomodeset` option if the first one fails to work. The installer will then prompt to choose if you need a keyboard map. This is probably needed with a non-US keyboard, otherwise hit enter. The Slackware installer will then ask a login. Press enter to log in as root. Make sure the Slackware Mirror USB is instered. For these examples, it is assumed the partition of the flash drive is `/dev/sdc1`. Change the path accordingly.

Create a mountpoint for the Slackware Mirror USB, as `/mnt` is used by the installer. Mount the Slackware Mirror USB. Partition the SSD.
```
mkdir /usb
mount /dev/sdc1 /usb
fdisk /dev/nvme0n1
```

Complete the `fdisk` session in a manner similar to below. Swap is recommeneded for the build system. 
```
root@darkstar:~# fdisk /dev/nvme0n1

Welcome to fdisk (util-linux 2.42.4).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): g
Created a new GPT disklabel (GUID: C4D8436B-6400-4207-920E-954C38277AC5).

Command (m for help): n
Partition number (1-128, default 1):
First sector (2048-500118158, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-500118158, default 500117503): +128M

Created a new partition 1 of type 'Linux filesystem' and of size 128 MiB.

Command (m for help): t
Selected partition 1
Partition type or alias (type L to list all): uefi
Changed type of partition 'Linux filesystem' to 'EFI System'.

Command (m for help): n
Partition number (2-128, default 2):
First sector (264192-500118158, default 264192):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (264192-500118158, default 500117503): +16GiB

Created a new partition 2 of type 'Linux filesystem' and of size 16 GiB.

Command (m for help): t
Partition number (1,2, default 2):
Partition type or alias (type L to list all): swap

Changed type of partition 'Linux filesystem' to 'Linux swap'.

Command (m for help): n
Partition number (3-128, default 3):
First sector (33818624-500118158, default 33818624):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (33818624-500118158, default 500117503):

Created a new partition 3 of type 'Linux filesystem' and of size 222.3 GiB.

Command (m for help): p
Disk /dev/nvme0n1: 238.47 GiB, 256060514304 bytes, 500118192 sectors
Disk model: SAMSUNG MZVLB256HAHQ-000L7
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: C4D8436B-6400-4207-920E-954C38277AC5

Device            Start       End   Sectors   Size Type
/dev/nvme0n1p1     2048    264191    262144   128M EFI System
/dev/nvme0n1p2   264192  33818623  33554432    16G Linux swap
/dev/nvme0n1p3 33818624 500117503 466298880 222.3G Linux filesystem

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```
> [!CAUTION]
> THE PREVIOUS PARTITION LAYOUT IS SUGGESTED
> FOR THE BUILD NODE ONLY! The build node is
> considered disposable. Production nodes will
> use LVM partitions to isolate the operating
> system from user data.

Now, start the setup program with the `setup` command. The installer will present a list of actions. It will proceed through them in order starting with the chosen item. Choose `ADDSWAP` to set up the swap partition. Choose previously created swap partition. Choose "No" when asked if you want to check for bad blocks. The installer will confirm the swapspace has been configured. Next it will prompt for the root partition. Choose the previously created Linux partition. The installer will then prompt to format the root partition, choose `Format`. When prompted for the filesystem, choose `ext4`. The installer will confirm the selection. Next, the installer will confirm it detected the UEFI partition. If the installer asks to add FAT or NTFS partitions to `/etc/fstab`, choose `No`. The installer will prompt for the source media. Choose `7  -  Install from a pre-mounted directory`. When prompted for the source directory, enter `/usb/slackware64-current/slackware64`. The installer will then prompt for package series. For the purposes of the build station, install package series a, ap, d, k, l, n, tcl, x, xap, and xfce. If the optional tagfiles are on the Slackware Mirror flash drive, see [^1], otherwise choose the `terse` install option to install the default package set.

Once the pacakges are installed, the installer will prompt to create a USB boot stick. This is not needed - choose `Skip`. The Slackware installer might ask to install LILO. Skip this as LILO is legacy. The installer then prompts to install ELILO. Skip that as well as GRUB will be installed later. The install may then ask to install gpm - skip it. When prompted, confirm `Yes` to configure the network. Enter a hostname and domain. Choose `No` when prompted for VLAN configuration. VLANs will be configured later. When prompted for the network configuration type, choose `DHCP`, and then `DHCPv4`. The installer will ask for a DHCP hostname - this is not needed, choose `Skip`. The installer will confirm the networking configuration. Next, the installer will prompt for services to run at startup. Leave the defaults and add `rc.ntp` for NTP support, and `rc.rpc` for NFS support. Choose 'No' when prompted for custom screen fonts. Select the appropriate clock configuration. The installer will then prompt for a default `vi` editor. Use `vim` for a modern `vi`, `elvis` for a more traditional `vi`. When prompted for the default window manager, choose `xinitrc.xfce`. Set an appropriately strong `root` password when prompted. The installer will confirm setup is complete and that you may reboot the system. Do not reboot the system. Choose `EXIT` and then `Shell` to exit the `bash` shell without rebooting.

Additional configuration needs done at this point. The Slackware installer leaves the newly installed root partition mounted at `/mnt`. System directories (such as `/proc`, `/dev`, `/sys`) are bind mounted into `/mnt` - so it's a great time to `chroot` into the new system and perform that configuration.

```bash
# chroot into the new install, load the profile
chroot /mnt
source /etc/profile

# install GRUB
grub-install /dev/nvme0n1
grub-mkconfig -o /boot/grub/grub.cfg

# add clusteradm user
useradd -d /home/clusteradm -g users -u 1050 -m -s /bin/bash clusteradm 
```
Set a password on the clusteradm account.

```
passwd clusteradm
```

> [!WARNING]
> THE PREVIOUS STEP SETS A PASSWORD ON THE clusteradm ACCOUNT!
> If a password is not set, that user will not be able to log
> into the system.

In addition, grant `sudo` access to the `clusteradm` account. The path includes `go` - it is not yet installed, but will be soon. Edit `/etc/sudoers.d/clusteradm` as follows (preferably with `visudo /etc/sudoers.d/clusteradm`):
```
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/go/bin"
clusteradm ALL=(ALL:ALL) ALL
```

Now, configure NTP. Edit /etc/ntp.conf and uncomment one of the ntp.org servers or add a custom server.

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





**Exit the `chroot` envronment by with the `exit` command.** Reboot the system with `reboot` and allow it boot the new Slackware installation from the NVMe.

Once restarted, the build system should now be ready for login on the local console and via SSH. Slackware enables SSH passwords by default for non-root accounts.



### build-system

#### Download

We need to install the custom build-system. The build-system is a script that sets up a local Jenkins and Semaphore instance.

```bash
git clone --depth=1 https://github.com/scottr131/build-system.git
```

#### Configure

Now, configure and start the build system. Create a DNS wildcard record that points to your build server. *.<servername> should resolve to the build server.

```bash
cd build-system
# Download binaries
./build-system.sh download all
# Create directories
./build-system.sh setup dirs
# Set up JDKs
./build-system.sh setup jdk
# Setup the auth system and reverse proxy
# You will be prompted for a username and password
./build-system.sh setup rp
./build-system.sh setup jenkins
```

#### Start

Next, start the reverse proxy and Jenkins. The reverse proxy requires `sudo` so that it can bind to port 443.

```bash
./build-system.sh start jenkins
sudo ./build-system.sh start rp # Requires sudo
```

#### Jenkins Configuration

After Jenkins starts it will generate an initial password. Obtain that password for the next step with `cat /home/clusteradm/build-system/jenkins/secrets/initialAdminPassword`

Finally, in a web browser, go to the Jenkins instance at `https://jenkins.<your-server>.<domain>`. The browser will present two certificate warnings because of self signed certificates. One is for the Authelia certficiate and one is for the Jenkins certificate. Bypass these errors and you should get an Authelia login page. Login with the reverse proxy account created earlier. Make sure to check "Remember me" to prevent multiple password prompts.

Jenkins will prompt for the initial passowrd obtained above. Jenkins will then prompt for plugins to install. Choose "Install suggested plugins." Go ahead and create a "First Admin User" account in Jenkins once the plugin install is complete. On the next page, make sure the Jenkins URL is correct and save your configuration. Jenkins is now configured it's (almost) time to start building software.



#### Build prerequisites

Now, create a Jenkins a job to build the "dev-tools". This includes packaging Go and JDK binaries as Slackware packages. These Go and JDK packages are needed to build other software. This is the basic process to create the job. It will only be documented here. Other jobs are similar.

In Jenkins, click "+ New Item". Call the new item "dev-tools" and choose "Pipeline" as the type. The job will created. In the General configuration, under "Pipeline", set "Definition" to "Pipeline script from SCM". Set "SCM" to "Git" and repository URL to "https://github.com/scottr131/slackbuilds.git". Under "Branches to build" set "Branch Specifier" to `*/main`. Set "Script Path" to `dev-tools.Jenkinsfile`. Save the job and build it.

The dev-tools do not install as part of the Jenkinsfile. To build the software needed for the cluster, we'll need to install Go and JDK 21 on the build node to build the rest of the software. 

```bash
# Note that <build number> may change depending on
# how many builds you perform
cd ~build-system/jenkins/jobs/dev-tools/builds/<build number>/archive/

# Note that package versions may change
sudo installpkg go-1.27.0-x86_64-1_SBo.txz temurin-jdk21-21.0.12.1+1-x86_64-1_SBo.txz
```

The yarn build system also needs installed. Yarn can be installed via npm corepack. This is required for the incus-stack to build correctly.

```bash
sudo npm install -g corepack
```

#### Build Software Stacks

Now, go back to Jenkins and build and install additional package "stacks". Next will be the management tools. Click " + New Item". Call the new item "mgmt-tools" and instead of choosing an item type, click "Duplicate and existing item." Type "dev-tools" and click OK. The job will created with the settings from dev-tools. Change "Script Path" to `mgmt-tools.Jenkinsfile`. Save the job and build it. Continue with these additional stacks. This is a recommended build order, but for these packages, the order isn't really important.

- utils-stack (Not required, nice to have)
- mgmt-tools (Ansible, OpenTofu)
- storage-stack (LINSTOR, ZFS, DRBD)
- ovn-stack (Open vSwitch and OVN)
- incus-stack (Incus and support tools)
- qemu-stack (QEMU and support libraries)



### Install Ansible and OpenTofu on Build Node
Incus will be installed on the build node using Ansible, but first Ansible will need to be installed. This is also a good time to deploy OpenTofu.
```bash
cd ~/build-system/jenkins/workspace/mgmt-tools
sudo installpkg ansible*.txz opentofu*.txz
```
#### Clone Deployment Repositories
Clone the tri-plane-infrastructure repository to obtain the various scripts (these instructions do not build the tri-plane-infrastructure, but uses the deployment scripts). Clone the ansible playbooks into the `build-system` directory. Go ahead clone the `linux` repository that contains the configuration files and tagfiles.
```bash
cd ~
git clone https://github.com/scottr131/tri-plane-infrastructure --depth=1
cd ~/build-system
git clone https://github.com/scottr131/ansible.git --depth=1
git clone https://github.com/scottr131/linux.git --depth=1
```

#### Incus on Build Node (with Ansbile)
The next step is create a deployment node that will used to network boot and deploy software to the other nodes. The deployment node will be a virtual machine, so we need to get Incus running on the build node. To do this, we need to install the packages from qemu-stack and incus-stack. Do this using the same Ansible playbooks that will be used for node deployment.



Create a basic Ansible inventory. Edit `~/build-system/ansible/hosts.ini`:
```
[deploynode]
buildhost ansible_user=clusteradm ansible_host=localhost
```

Generate a host key and trust it
```bash
ssh-keygen
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
```

Create an API key in Jenkins. Edit the script to reflect your API key. Collect packages.
```bash
cd ~/tri-plane-infrastructure/scripts
./collect_packages-api.sh incus-stack qemu-stack 
```

Config files are currently stored in a seperate repository. Clone that and copy the config files into the ansible directory. Then run the `copy_configs.sh` script to copy the scripts to appropriate location for Ansible deployment.
```bash
~/tri-plane-infrastructure/scripts/copy_configs.sh
```

Now, install the Qemu and Incus stacks onto the build node by running the Ansible playbooks. LINSTOR and DRBD will be used on production nodes, but are not needed on the build node.

```bash
cd ~/build-system/ansible
ansible-playbook -i hosts.ini -e "target_hosts=buildhost" qemu/qemu-on-slackware.yaml
ansible-playbook -i hosts.ini -e "target_hosts=buildhost" incus/incus-on-slackware.yaml
ansible-playbook -i hosts.ini -e "target_hosts=buildhost" incus/incus-groups.yaml
ansible-playbook -i hosts.ini -e "target_hosts=buildhost" local-config/local-config.yaml
```

Finally, we need to create a local network bridge to allow the VMs to connect to the LAN. Edit `/etc/rc.d/rc.inet1.conf`. In most cases you can just add two lines to create a bridge on eth0.

```bash
# IPv4 config options for eth0:
IPADDRS[0]="10.1.31.18/24"
USE_DHCP[0]=""
# IPv6 config options for eth0:
IP6ADDRS[0]=""
USE_SLAAC[0]=""
USE_DHCP6[0]=""
# Generic options for eth0:
DHCP_HOSTNAME[0]=""

## Add these lines
BRNICS[0]="eth0"
IFNAME[0]="br-lan"
```


Restart the system to allow the CGROUPS version change (from the Incus playbook) and new networking configuration to take effect. The incus daemon should come up automatically after restart. 

### Configure Incus on Build Node

Now, configure Incus on the build node.

```bash
sudo incus admin init
```

Below is a sample run. Settings may need adjusted per the environment.

```text
Would you like to use clustering? (yes/no) [default=no]:
Do you want to configure a new storage pool? (yes/no) [default=yes]:
Name of the new storage pool [default=default]:
Name of the storage backend to use (ceph, lvm, dir, btrfs) [default=btrfs]: dir
Where should this storage pool store its data? [default=/var/lib/incus/storage-pools/default]:
Would you like to create a new local network bridge? (yes/no) [default=yes]: no
Would you like to use an existing bridge or host interface? (yes/no) [default=no]: yes
Name of the existing bridge or host interface: br-lan
Would you like the server to be available over the network? (yes/no) [default=no]: yes
Address to bind to (not including port) [default=all]:
Port to bind to [default=8443]:
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]:
Would you like a YAML "init" preseed to be printed? (yes/no) [default=no]:
```

Verfiy network connectivity by configuring local user client.

```bash
sudo incus config trust add localhost
# incus will return a token
incus remote add localhost
# Confirm fingerprint and paste token
```

Verify server operation.

```bash
incus remote switch localhost
incus list
# This should return an empty list of instances
```

#### Deployment Node (VM)

##### Preparation

Create a `usb` mountpoint and mount the Slackware Mirror USB flash drive.
```bash
sudo mkdir -p /mnt/usb
sudo mount /dev/sdg1 /mnt/usb
```

Import the Slackware installer USB image.

```bash
# Import the slackware installer image (not technically an ISO, but it will import this way)
incus storage volume import default /mnt/usb/slackware64-current/usb-and-pxe-installers/usbboot.img slackware-install --type=iso
```

Create the deployment VM using OpenTofu
```
cd ~/tri-plane-infrastructure/tf/deploy
tofu init
tofu apply
```

You will need to connect a remote Incus client or configure the web interface in order to see the VM console. Those are currently beyond the scope of this document.

##### Install OS

Unmount the Slackware files USB flash drive. Connect the USB flash drive of Slackware files to the VM. Boot the VM and connect to the console. Install Slackware in a manner similar to the build node. This time only install package series a, ap, l, n, tcl, x, xap, and xfce from the `mini-gui` tagfiles (or terse if tagfiles are not available). Configure the VM with a hostname of `deploy` and a static IP address appropriate for your LAN. Just as with the build node, you will need to create the cluster account `clusteradm`.

```bash
# These are examples and will need to be modified.
sudo umount /mnt/usb
incus config device add deploy usb1 usb busnum=2 devnum=2
incus start deploy
```

##### Configure Deployment Node

```
# As root, since there are no other accounts
useradd -d /home/clusteradm -g users -u 1050 -m -s /bin/bash clusteradm
```

In addiition, we need to grant `sudo` access to the `clusteradm` account. The path includes `go` - it is not yet installed, but will be soon. Edit `/etc/sudoers.d/clusteradm` as follows:

```
clusteradm ALL=(ALL:ALL) NOPASSWD: ALL
```

Make sure to set a password on the clusteradm account.

```
passwd clusteradm
```

Also, it's a good idea to configure an NTP server in `/etc/ntp.conf`. Shut down the VM, disconnect the virtual Slackware installer USB, leave your real Slackware files USB connected, and boot up the VM into the newly installed OS.

```
incus stop deploy
incus config device remove deploy installer
incus start deploy
```


##### Clone Deployment Repositories

SSH into the newly started deployment node and clone the needed repositories.

```
cd ~
git clone --depth=1 https://github.com/scottr131/tri-plane-infrastructure.git
git clone --depth=1 https://github.com/scottr131/build-system.git
cd build-system/
git clone --depth=1 https://github.com/scottr131/linux.git
git clone --depth=1 https://github.com/scottr131/ansible.git
```

##### Create Deploy Directory Tree

Restart the `deploy` instance (`incus start deploy`). The USB should be reconnected to the VM. SSH into the VM as clusteradm. Mount the Slackware files USB flash drive to /mnt/hd and prepare the files for network deployment. Network boot files will be stored in `/srv`. Slackware will be stored at `/srv/slackware/slackware64-current`, tagfiles at `/srv/slackware/tagsets`, `/srv/pxe` contains the boot binaries, `/srv/pxe/nodes` contains node-specific configuration files, and `/srv/tftp` contains the boot binaries to be served via TFTP by dnsmasq.

The node-specific config directory is actually `/srv/pxe/nodes`. This document will have a companion repository that contains scripts and config files. The location is yet to be determined. For the moment, assume the Slackware flies USB drive contains a `scripts` directory with deployment related scripts and a `conf` directory that contains various configuration file templates.

As `clusteradm`, create the directory tree and own it. 

```
sudo mkdir -p /srv/pxe/nodes /srv/slackware/tagsets /srv/tftp
sudo chown -R clusteradm /srv/pxe /srv/slackware /srv/tftp
```


##### Copy the tagfiles to the expected location
```
sudo cp -R ~/build-system/linux/slackware/tagfiles/current/* /srv/slackware/tagsets/
```

##### Copy Local Mirror

Now we'll mount the USB and use rsync to copy the Slackware mirror. This same command can be used later to synchronize the Slackware mirror from a USB flash drive.

```
sudo mkdir /mnt/usb              # Or you preferred mount point
sudo mount /dev/sdb1 /mnt/usb    # Modify based on your system
rsync -havP --delete --delete-after  --no-o --no-g --safe-links  --timeout=60  --exclude "EFI/*" --exclude "extra/*" --exclude "pasture/*" --exclude "patches/*"  --exclude "source/*"  --exclude "testing/*" /mnt/usb/slackware64-current /srv/slackware
```




---

[^1] Choose `tagpath` and enter the path to the tagfiles (ex., `/usb/tagfiles/mini-dev-usb`)