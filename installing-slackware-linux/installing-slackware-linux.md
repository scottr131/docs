This is the basic procedure I use to install Slackware Linux on a system.  This is a network-based installation of Slackware-current (which is the always-updating development release - it’s actually quite stable).
First, I'll download the Slackware-current installer and write it out to a USB drive.  It's `/dev/sdb` in my case.  The list of Slackware mirror is available on the Slackware web site.

```bash
wget https://mirrors.slackware.com/slackware/slackware64-current/usb-and-pxe-installers/usbboot.img
```

wget will download the file.

```text
Resolving ftp.ussg.indiana.edu (ftp.ussg.indiana.edu)... 156.56.247.193
Connecting to ftp.ussg.indiana.edu (ftp.ussg.indiana.edu)|156.56.247.193|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 112755712 (108M) [application/octet-stream]
Saving to: ‘usbboot.img’

usbboot.img                                100%[=======================================================================================>] 107.53M  54.9MB/s    in 2.0s

2025-08-16 22:18:59 (54.9 MB/s) - ‘usbboot.img’ saved [112755712/112755712]
```

I'll write the file to `/dev/sda`

```bash
sudo dd if=usbboot.img of=/dev/sda
```

Now I'll boot the build/deploy node off the USB drive.  I'll make sure to boot the stick in UEFI mode so I can properly install Slackware in UEFI mode.  Also, I have secure boot disabled because I have yet to experiment with that and Slackware.

Once the install USB boots, I am prompted for a keyboard.  You can enter 1 here if you need a different keyboard layout.  I'm using a US keyboard, so I'll hit enter.  At the login prompt, I'll also hit enter.  The Slackware installer login prompt is not a real login prompt.

Slackware can be installed over the Internet from the mirrors.  I will install Slackware from a local NFS server.  Either way, we need to configure the network interface.  In my case, it's a simple network, so I'll just get an address from DHCP and be set.  In Slackware, the first network card is usually called eth0.

```bash
dhcpcd eth0
```

Now, I'll make a directory to mount my NFS share and then mount the NFS share.  If you mount an NFS share at this point in the Slackware installer, you'll probably need to use the nolock option since RPC is not started.  I'll be mounting the share read-only as well since there is no reason to write to it.  Another option is starting the RPC server with `/etc/rc.d/rc.rpc start`.

```bash
mkdir /slackware
mount -t nfs -o ro,nolock nfsserver:/mnt/pool/slackware /slackware
```

Next, I'll need to partition the boot drive.  I'll create a 256MB UEFI partition, 8GB for swap, and leave the rest of the drive as a single ext4 partition.  This gives plenty of space to compile software and stage packages for deployment with Ansible.

```text
root@darkstar:~# fdisk /dev/nvme0n1

Welcome to fdisk (util-linux 2.41.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): g
Created a new GPT disklabel (GUID: DFE425EF-4D27-4DA7-A16A-58DC419AB204).

Command (m for help): n
Partition number (1-128, default 1):
First sector (2048-500118158, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-500118158, default 500117503): +256M

Created a new partition 1 of type 'Linux filesystem' and of size 256 MiB.

Command (m for help): t
Selected partition 1
Partition type or alias (type L to list all): uefi
Changed type of partition 'Linux filesystem' to 'EFI System'.

Command (m for help): n
Partition number (2-128, default 2):
First sector (526336-500118158, default 526336):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (526336-500118158, default 500117503): +8G

Created a new partition 2 of type 'Linux filesystem' and of size 8 GiB.

Command (m for help): t
Partition number (1,2, default 2):
Partition type or alias (type L to list all): swap

Changed type of partition 'Linux filesystem' to 'Linux swap'.

Command (m for help): n
Partition number (3-128, default 3):
First sector (17303552-500118158, default 17303552):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (17303552-500118158, default 500117503):

Created a new partition 3 of type 'Linux filesystem' and of size 230.2 GiB.

Command (m for help): p
Disk /dev/nvme0n1: 238.47 GiB, 256060514304 bytes, 500118192 sectors
Disk model: SAMSUNG MZVLB256HAHQ-000L7
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: DFE425EF-4D27-4DA7-A16A-58DC419AB204

Device            Start       End   Sectors   Size Type
/dev/nvme0n1p1     2048    526335    524288   256M EFI System
/dev/nvme0n1p2   526336  17303551  16777216     8G Linux swap
/dev/nvme0n1p3 17303552 500117503 482813952 230.2G Linux filesystem

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

root@darkstar:~# setup
```

I'll start the installation by choosing "ADDSWAP".  For systems without swap (the nodes), I'll start at "TARGET".  ZRAM will be enabled automatically either way.  The Slackware installer then confirms the location of the swap partition.  I choose "No" when asked if I need to check for bad blocks, this isn't needed.  The Slackware installer then prompts for the root partition and I'll pick the large partition I prepared for it.  After I pick the partition, the Slackware installer asks to format it.  The only reason you wouldn't do this is if there is existing data on the partition.  Since I don't have data, I'll format the partition with no bad block checking.  I'm then prompted for the filesystem.  Standard ext4 will be fine in this case, so I'll choose that.  The Slackware installer will then format the partition.  After that, the installer detected that my UEFI system partition was unformatted, so I let it format it with FAT32 as is standard for the ESP.  The Slackware installer then formats the partition and adds it to /etc/fstab.  Next, the installer prompts for the location of the install media.  I almost always install from the Slackware mirrors over the Internet or from my local NFS server.  In this case, I already mounted my NFS share earlier, so I'll choose 7 - Install from a pre-mounted directory.  The Slackware installer will then prompts for the location of the pre-mounted folder.  It is looking for the `slackware64` folder - the folder with all the Slackware disk sets.  I'll put in `/slackware/slackware64-current/slackware64` to reflect the location on my NFS server. 
If the disk sets are found successfully, the Slackware installer will present a list of sets to choose from.    For the build/deploy node,   I'll choose A, AP, D, K, L, N, TCL, and X,  This won't make a system with a very useable GUI, but should give all the support libraries needed to build the additional software on.  Then, the Slackware installer asks for the prompting mode.  I always choose terse.  Full works well too.  Both those options will install Slackware with the default packages in the tagfile.  Menu, expert, and newbie modes aren't useful anymore.

After the disk sets are installed, the Slackware installer asks to create a USB boot stick.  I say skip, as I just use the Slackware installer boot USB for repairs.  

The Slackware installer then asks to install LILO.  I'll skip this as LILO is legacy.  The installer then prompts to install ELILO, which I also skip as I'll install GRUB later.

I'm then prompted to install gpm, which I'll skip as I don't use it.

I'll say Yes to configure the network.  For the initial configuration, I'll choose a name of `deploy`, a domain of `ic1.local` and DHCP for the IP address.  This is because eth0 of the build/deploy server is connected to my local LAN for Internet access.  It will be connected to the cluster network later.

After the basic network configuration is completed, the installer then prompts for services to start on boot.  I leave the defaults selected, but add NTP as it's important for all the clocks to in sync on a cluster setup.  I'll have to manually configure NTP later, as you can't do it from the installer. Also, don't run NFS or RPC on a cluster node - it seems to cause problems with containers starting, and I have fully diagnosed it.  

I say no the custom screen fonts and set my time zone appropriately.  Next I choose my vi.  vim works fine.  elvis is the classic Slackware vi.  

When prompted, I'll choose Motif as the WM.  The GUI won't really get used except for maybe troubleshooting.   The X disk set was mainly installed for the libraries it includes.

I then set a secure password.  Root logins are only permitted from the console.  

When the Slackware installer completes, it says "you may reboot your system."  This is NOT the case as I did not install ELILO.  I choose OK, then EXIT the installer, and then choose to open a shell.

The easiest way to install GRUB is from the new installation.  At the shell prompt I'll chroot to the new install and run the commands to install and configure GRUB.

```text
root@darkstar:~# chroot /mnt
root@darkstar:/# source /etc/profile
root@darkstar:/# grub-install /dev/nvme0n1
Installing for x86_64-efi platform.
Installation finished. No error reported.
root@darkstar:/# grub-mkconfig -o /boot/grub/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-generic
Found initrd image: /boot/intel-ucode.img /boot/amd-ucode.img /boot/initrd-generic.img
Found linux image: /boot/vmlinuz-6.12.39
Found initrd image: /boot/intel-ucode.img /boot/amd-ucode.img /boot/initrd-6.12.39.img
Warning: os-prober will be executed to detect other bootable partitions.
Its output will be used to detect bootable binaries on them and create new boot entries.
Adding boot menu entry for UEFI Firmware Settings ...
done

root@darkstar:/# exit
exit
root@darkstar:~# reboot
```

### Configure the user

After the system comes back up in Slackware, I'll log in as root and make the final user configurations.  After this, I'll be able to access the build/deploy node via SSH.  I'll use `useradd` to add a user called `myuser` that will have sudo access

```text
root@deploy:~# useradd -d /home/myuser -g users -m -s /bin/bash myuser
root@deploy:~# usermod -a -G wheel myuser
root@deploy:~# passwd myuser
New password:
Retype new password:
passwd: password updated successfully
```

By default in Slackware, the wheel group does not have sudo access.  I can run the `visudo` command to edit `/etc/sudoers`.  Around line 125, there is a template for allowing sudo access to the wheel group.  The user will be required to type their own password to gain sudo access.  I'll uncomment that template to allow this.  Make sure not to add or leave any extra whitespace.

```text
## Uncomment to allow members of group wheel to execute any command
# %wheel ALL=(ALL:ALL) ALL

## Same thing without a password
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
```
