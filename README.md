# KVM QEMU libvirt Windows 10 guest on Garuda Linux (Arch-based) setup and example

⚠️⚠️ Note this is not intended to be a copy and paste or follow as is for your setup, even if you have identical hardware and software. You will run into problems. This is to be used as a baseline for similiar or identical hardware if you are experiencing issues or are looking into the possibility of doing this by looking at examples.

You can find more examples on the Arch Wiki: https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF/Examples

<hr>

## Hardware:
- Motherboard: MSI Z370-A PRO
- CPU: Intel i7-8700k OC @ 4.2GHz
- GPU 1: MSI NVIDIA GTX 1070 (Host GPU)
- GPU 2: EVGA NVIDIA GTX 550 Ti 2048MB (Guest GPU)
- Integrated GPU: Intel UHD 630 (not used)
- Memory: 16GB DDR4 OC @ 2800MHz
- Host SSD: SATA Western Digital Green 250GB
- Guest SSD: SATA Samsung 860 EVO 250GB

## Software:
- Motherboard firmware version: [7B48v2C 2020-06-10](https://download.msi.com/bos_exe/mb/7B48v2C.zip)
- Linux distribution (Host OS): [Garuda Linux GNOME](https://garudalinux.org/)
- Linux kernel and version: linux-zen 5.11.16
- QEMU version: 5.2.0
- Guest OS: Microsoft Windows 10 Enterprise 20H2 (19042.928)

<hr>

## Quirks, Extras, and Notes:

- I did not install Windows. I used my pre-existing Windows installation encrypted with Bitlocker (you must suspend it before doing anything otherwise you'll have a lot of fun entering your recovery key) with Virtio SCSI to maintain compatibility if I need to boot normally in the future\*:
  
```xml
    <disk type="block" device="disk">
      <driver name="qemu" type="raw" cache="writeback" io="threads" discard="unmap"/>
      <source dev="/dev/disk/by-id/ata-Samsung_SSD_860_EVO_250GB_S3YHNX1KB15676F"/>
      <target dev="sda" bus="scsi"/>
      <address type="drive" controller="0" bus="0" target="0" unit="0"/>
    </disk>

    <!-- ... -->

    <controller type="scsi" index="0" model="virtio-scsi">
      <driver queues="8" iothread="1"/>
      <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
    </controller>
```

\*_Note this is not recommended due to Windows not being great at all for "system hotswapping" without using the pre-existing backup software by Windows._
- Distro is using Pipewire with Pulseaudio compatibility.
- The EVGA GTX 550 Ti 2048MB does not have UEFI support. We are required to patch it as shown down below.

### QEMU commandline:
```xml
  <qemu:commandline>
    <qemu:arg value="-audiodev"/>
    <qemu:arg value="pa,id=hda,server=unix:/run/user/1000/pulse/native"/>
    <qemu:arg value="-object"/>
    <qemu:arg value="input-linux,id=mouse1,evdev=/dev/input/by-id/usb-Razer_Razer_DeathAdder_Essential_White_Edition-event-mouse"/>
    <qemu:arg value="-object"/>
    <qemu:arg value="input-linux,id=kbd1,evdev=/dev/input/by-id/usb-Lite-On_Technology_Corp._USB_Multimedia_Keyboard-event-kbd,grab_all=on,repeat=on"/>
  </qemu:commandline>
```

### Bitlocker support:
While you can create a vTPM, hardware TPM is always better for this.

In virt-manager, add a new hardware and be it TPM.
<br>
Configure it as model CRB and path to `/dev/tpm0`.

If model CRB does not work as intended, try TIS. CRB is likely to work though.
```xml
<tpm model="tpm-crb">
  <backend type="passthrough">
    <device path="/dev/tpm0"/>
  </backend>
  <alias name="tpm0"/>
</tpm>
```

Give the guest some RNG too:
```xml
<rng model="virtio">
  <backend model="random">/dev/urandom</backend>
  <alias name="rng0"/>
  <address type="pci" domain="0x0000" bus="0x0c" slot="0x00" function="0x0"/>
</rng>
```

<hr>

## Pre"installation"
- **UPDATE YOUR UEFI/BIOS TO THE LATEST. PLEASE.** Many, many, many, many, many, many, many, many, _many_ IOMMU and Intel VT-x/AMD-v bugs are solved with updating your motherboard firmware. Hackintosh users know how painfully true motherboard firmware updates solve so many issues.

### Important files modified or added:

`/usr/local/bin/vfio-pci-override.sh`
```bash
#!/bin/sh

# We know what GPU we want to pass through so we'll define it. Use lspci to find the VGA compat. controller and Audio device of your GPU.
DEVS="0000:03:00.0 0000:03:00.1"

if [ ! -z "$(ls -A /sys/class/iommu)" ]; then
    for DEV in $DEVS; do
        echo "vfio-pci" > /sys/bus/pci/devices/$DEV/driver_override
    done
fi

modprobe -i vfio-pci
```
Make sure you give it execute permissions or it will not work! `sudo chmod +x vfio-pci-override.sh`

`/etc/mkinitcpio.conf`
```
MODULES=(vfio_pci vfio vfio_iommu_type1 vfio_virqfd nvidia nvidia_modeset nvidia_uvm nvidia_drm)
# ...
FILES=(/usr/local/bin/vfio-pci-override.sh)
# ...
# Add modconf if it doesn't exist, usually after autodetect. Order matters. 
HOOKS="base udev autodetect modconf ..."
```

`/etc/modprobe.d/kvm.conf`
```
# These should solve some bugcheck issues along with the patched vBIOS.
options kvm ignore_msrs=1
options kvm report_ignored_msrs=0
```

`/etc/modprobe.d/vfio.conf`
```
install vfio-pci /usr/local/bin/vfio-pci-override.sh
options vfio-pci ids=10de:1244,10de:0bee

# Unsafe interrupts
options vfio_iommu_type1 allow_unsafe_interrupts=1
```

`/etc/default/grub` (Kernel paremetres)
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash vfio-pci.ids=10de:1244,10de:0bee intel_iommu=on iommu=pt rd.driver.pre=vfio-pci ..."
```

`/etc/libvirt/qemu.conf`
```
# ...
user = "sysop" # your user
group = "kvm"
# ...

cgroup_device_acl = [
     "/dev/input/by-id/usb-Lite-On_Technology_Corp._USB_Multimedia_Keyboard-event-kbd",
     "/dev/input/by-id/usb-Razer_Razer_DeathAdder_Essential_White_Edition-event-mouse",
     "/dev/null", "/dev/full", "/dev/zero",
     "/dev/random", "/dev/urandom",
     "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
     "/dev/rtc","/dev/hpet", "/dev/sev"
]
```
cgroup_device_acl is for allow access to pass through keyboard and mouse data to guest via evdev. See: https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Passing_keyboard/mouse_via_Evdev

- Download the stable ISO of virtio-win: https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md
- Mount it, drag all the files to a USB drive.
- Boot up your Windows installation, plug in the USB, and install all the drivers as necessary.
- Download [Display Driver Uninstaller](https://www.guru3d.com/files-details/display-driver-uninstaller-download.html), boot into safe mode minimal, and remove ALL of the display drivers. Disable Windows Update driver fetching **temporarily**. After correcting Error 43 and bugcheck SYSTEM_SERVICE_EXCEPTION nvlddmkm.sys, we will turn it back on. Error 43 will not be an issue if you are running a modern NVIDIA GPU that has a driver starting v465.

### Networking
- Created a bridged connection with nm-connection-editor. Because the "Network Interfaces" tab was removed from virt-manager<sup>[source](https://listman.redhat.com/archives/virt-tools-list/2018-October/msg00032.html)</sup> you will need to do it via virsh.

First create a temporary file with the bridge defined `/tmp/bridge.xml`
```xml
<network>
  <name>br0</name>
  <forward mode="bridge"/>
  <bridge name="br0"/>
</network>
```
Replace `br0` with your bridge interface if necessary.

Define it: `# virsh net-define /tmp/bridge.xml`
<br>
Start and enable autostarting: `# virsh net-start br0 && virsh net-autostart br0`

Utilize the bridge in the VM:
```xml
    <interface type="network">
      <mac address="52:54:00:71:69:a7"/>
      <source network="br0"/>
      <model type="virtio"/>
      <driver queues="8"/>
      <address type="pci" domain="0x0000" bus="0x02" slot="0x00" function="0x0"/>
    </interface>
```
Model of network card is virtio. Emulating a real card is very CPU intensive and has a lot of overhead compared to virtio. Though, passing through real hardware is always better.
<br>
Windows requires NetKVM driver from virtio drivers to utilize virtio model.

<hr>

## Dumping vBIOS and patching it for EVGA GTX 550 Ti 2048MB
Because the EVGA GTX 550 Ti does not have UEFI support, we will have to patch the vBIOS. You do **NOT** need to flash anything as we can simply set the ROM to it on PCI passthrough.

You'll need a Windows machine to run the software. Could work with WINE but untested and not exactly safe to do.

**The patch is NOT meant to be followed exactly as is. This exact process has only been found to work with the EVGA GTX 550 Ti 2048MB.**

First, dump it to a file:
```
# cd /sys/bus/pci/devices/<GPU_ID_HERE>
# echo 1 > rom
# cat rom > /usr/local/lib/GTX_550Ti.rom
# echo 0 > rom
```
Stick that file onto a FAT32 USB and head to your Windows machine.

Download GOPUpd: https://www.win-raid.com/file.php?url=http%3A%2F%2Ffiles.homepagemodules.de%2Fb602300%2Ff16t892p15730n2_UixHgnbP.rar&r=&content=RE%3A_AMD_and_Nvidia_GOP_update

Extract it. Take your vBIOS you dumped and **drag it onto `GOPupd.bat`**

Let it run. When asked if you want to update GOP to the latest available, hit Y.
<br>
Select `GF10x`.

Once the patching is done, take the new vBIOS appended with `_updGOP.rom`, put it on your USB, and send it back to your host Linux machine. Put the patched vBIOS somewhere like `/usr/local/lib`.
**Make sure the file can actually be read. If it's owned by root, chown it to the user you're launching the guest from e.g. `sysop`.**

In virt-manager, add your GPU PCIe hardware. Head to the XML file and add this line to the `<hostdev>` block:
```xml
      <rom bar="on" file="/usr/local/lib/GTX550Ti_GF10x.bin"/>
```

It should look something like this:
```xml
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x03" slot="0x00" function="0x0"/>
      </source>
      <rom bar="on" file="/usr/local/lib/GTX550Ti_GF10x.bin"/>
      <address type="pci" domain="0x0000" bus="0x0a" slot="0x00" function="0x0"/>
    </hostdev>
```

This should also solve Error 43 and possible bugchecks related to the NVIDIA driver.

<hr>

## Post-"installation"
Line-based interrupts are shit. Plain and simple. And a lot more hardware supports Message-signaled based interrupts than just your GPU. The effects of using MSI compared to Line-based are _tremendous_ with a QEMU KVM.

Use [MSI_util_v3](http://www.mediafire.com/file/ewpy1p0rr132thk/MSI_util_v3.zip/file) ([original thread](https://forums.guru3d.com/threads/windows-line-based-vs-message-signaled-based-interrupts-msi-tool.378044/)) and enable MSI for all devices that can support it and reboot the guest. If you pass through audio via Pulseaudio like me, the latency and quality will be a LOT better. Hardware acceleration will be smoother.

**DO. NOT. ENABLE. MSI. FOR. DEVICES. THAT. DO. NOT. SUPPORT. IT.** 9/10 times your system will be rendered unbootable until you restore the registry hive where the key is located.