On a Windows 11 PC, download and install the Windows ADK and WinPE add-on.  Also download the Virtio ISO.

Start "Deployment and Imaging Tools Environment" as administrator.  Create a set of WinPE files.

```cmd
copype amd64 C:\winpe-kvm
```

Mount the WinPE WIM file so it can be customized.

```cmd
dism /mount-image /imagefile:"C:\winpe-kvm\media\sources\boot.wim" /index:1 /MountDir:"C:\winpe-kvm\mount"
```

Mount (or extract if desired) the virtio ISO

```PowerShell
Mount-DiskImage virtio-win-0.1.271.iso
```

Assume the image was mounted to H:.  Now, we need to load the virtio storage and network drivers into WinPE so that it can see the virtio storage and network devices on boot.

```cmd
# Incorporate virtio SCSI driver
dism /add-driver /image:"C:\winpe-kvm\mount" /driver:"H:\vioscsi\w11\amd64\vioscsi.inf"
# Incorporate virtio network driver
dism /add-driver /image:"C:\winpe-kvm\mount" /driver:"H:\NetKVM\w11\amd64\netkvm.inf"
```

Now, it may be helpful to install a few optional components like PowerShell, WSH, HTA support,

```cmd
# WMI can be used to obtain info about the virtual hardware
dism /image:"C:\winpe-kvm\mount" /add-package /packagepath:"C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\WinPE_OCs\WinPE-WMI.cab"
# NetFX subset of .NET Framework 4.5
dism /image:"C:\winpe-kvm\mount" /add-package /packagepath:"C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\WinPE_OCs\WinPE-NetFx.cab"
# WSH and other scripting support for automation
dism /image:"C:\winpe-kvm\mount" /add-package /packagepath:"C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\WinPE_OCs\WinPE-Scripting.cab"
# HTA if we want a GUI to scripts
dism /image:"C:\winpe-kvm\mount" /add-package /packagepath:"C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\WinPE_OCs\WinPE-HTA.cab"
# Add Bitlocker and TPM support
dism /image:"C:\winpe-kvm\mount" /add-package /packagepath:"C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\WinPE_OCs\WinPE-SecureStartup.cab"
# PowerShell
dism /image:"C:\winpe-kvm\mount" /add-package /packagepath:"C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\WinPE_OCs\WinPE-PowerShell.cab"
# DISM PowerShell cmdlets
dism /image:"C:\winpe-kvm\mount" /add-package /packagepath:"C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\WinPE_OCs\WinPE-DismCmdlets.cab"
```

Now, I'll copy a few drivers to the WinPE root.  This is just kind of a safety measure in case they are needed to load on demand or to install into another OS later.  Please note, these end up the WinPE RAM disk, so the fewer, the better.

```cmd
md C:\winpe-kvm\mount\Drivers\vioscsi
md C:\winpe-kvm\mount\Drivers\viostor
md C:\winpe-kvm\mount\Drivers\netkvm
md C:\winpe-kvm\mount\Drivers\qxldod
copy H:\vioscsi\w11\amd64\* C:\winpe-kvm\mount\Drivers\vioscsi\
copy H:\viostor\w11\amd64\* C:\winpe-kvm\mount\Drivers\viostor\
copy H:\NetKVM\w11\amd64\netkvm.inf C:\winpe-kvm\mount\Drivers\netkvm\
copy H:\qxldod\w10\amd64\qxldod.inf C:\winpe-kvm\mount\Drivers\qxldod\
```

Add any required updates.  This will be based on the version of the ADK you are using.  For the latest WinPE, you need the latest cumulative update for the OS.  In this case, Windows 11 24H2 August 2025 update.

```cmd
dism /add-package /image:"C:\winpe-kvm\mount" /packagepath:"C:\Users\insane131\Downloads\windows11.0-kb5063875-x64_9940f1ee9f1d127965dba841c8652267268fd73d.msu"
```

Cleanup the image to lock in the udpates

```cmd
md C:\temp
dism /cleanup-image /image:"C:\winpe-kvm\mount" /startComponentCleanup /resetBase /scratchDir:C:\temp
```

Unmount the image while commiting the changes to the WIM file.

```cmd
dism /unmount-image /mountDir:"C:\winpe-kvm\mount" /commit
```

(You may get an error that a file is in use, then try the commit again.  If you then get an error that the earlier commit probably succeeded, then it usually did.  Change /commit to /discard and hope the chages stay.)

At this point, you can add files to `C:\winpe-kvm\media`.  Those files will be included in the ISO, but won't be loaded into the RAM drive.  I like to just add the entire virtio ISO to a folder.

```cmd
mkdir C:\winpe-kvm\media\virtio
robocopy /e /np /mt H:\ C:\winpe-kvm\media\virtio\ *.*
```

Finally, create an ISO file from the `C:\winpe-kvm\media` folder contents.

```cmd
makewinpemedia /iso /f C:\winpe-kvm C:\winpe-kvm\winpe-kvm.iso
```

This ISO should now be bootable and have networking and storage support in (most?) KVM hypervisors.  This should include bare QEMU, libvirt, Incus, Nutanix AHV, OpenStack, Proxmox VE, oVirt, and probably more.

1 - https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-mount-and-customize?view=windows-11
2 - https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/?C=M;O=D
