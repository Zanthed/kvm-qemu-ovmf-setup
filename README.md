
- [Specs](#specs)
  - [Hardware](#hardware)
  - [Software](#software)
- [Preinstallation\/Setup](#preinstallationsetup)
  - [Guest Audio and keyboard+mouse](#guest-audio-and-keyboardmouse)
  - [Important files modified or added](#important-files-modified-or-added)
  - [Static Hugepages](#static-hugepages)
- [Installation](#installation)
- [Post-installation (and performance tweaks!)](#post-installation-and-performance-tweaks)
  - [Networking](#networking)
  - [Hyper-V enlightenments, APIC, SMM, IOAPIC driver, and vmport](#hyper-v-enlightenments-apic-smm-ioapic-driver-and-vmport)
  - [Passthrough host CPU cache and enable CPU features](#passthrough-host-cpu-cache-and-enable-cpu-features)
  - [virtio-scsi (and Virtio drivers)](#virtio-scsi-and-virtio-drivers)
  - [NVIDIA Drivers](#nvidia-drivers)
  - [System and VM internal clock](#system-and-vm-internal-clock)
  - [Message signal-based interrupts](#message-signal-based-interrupts)
  - [Bitlocker](#bitlocker)
  - [Reducing VM presence](#reducing-vm-presence)
  - [Minor performance tweaks](#minor-performance-tweaks)
- [CPU Pinning, Interrupts, Affinity, Governors, Topology, Isolating, and Priorities](#cpu-pinning-interrupts-affinity-governors-topology-isolating-and-priorities)
  - [Topology](#topology)
  - [Isolating the CPUs](#isolating-the-cpus)
  - [_Interrupts, Governors, and Affinity coming soon. I need sleep._](#interrupts-governors-and-affinity-coming-soon-i-need-sleep)

<hr>

# Specs

## Hardware
- Motherboard: MSI Z370-A PRO
- CPU: Intel i7-8700k OC @ 4.2GHz
- GPU 1: MSI NVIDIA GTX 1070 (Guest GPU)
- GPU 2: EVGA NVIDIA GTX 550 Ti 2048MB (unused)
- Integrated GPU: Intel UHD 630 (Host GPU)
- Memory: 32GB DDR4 2400MHz
- Host SSD: SATA Western Digital Green 250GB
- Guest SSD: SATA Samsung 860 EVO 250GB

## Software
- Motherboard firmware version: [7B48v2D2 (Beta) 2021-04-20](https://download.msi.com/bos_exe/mb/7B48v2D2.zip)
- Linux distribution (Host OS): Arch Linux
- Linux kernel and version: linux-hardened
- QEMU version: 6.0.0
- Guest OS: Microsoft Windows 10 Enterprise 20H2 (19042.985)
- Distro is using Pipewire JACK with Pipewire JACK dropin.
- Guest audio is using Pipewire JACK _and_ NVIDIA HDMI Audio. I alter between both if I need a microphone or not. Usually I just use HDMI audio.
<hr>

# Preinstallation\/Setup
**UPDATE YOUR UEFI/BIOS TO THE LATEST. PLEASE.** Many, many, many, many, many, many, many, many, _many_ IOMMU and Intel VT-x/AMD-v bugs are solved with updating your motherboard firmware. Hackintosh users know how painfully true motherboard firmware updates solve so many issues.

## Guest Audio and keyboard+mouse

We're simply using Pulseaudio\/Pipewire here. We're also using evdev to pass through our keyboard and mouse with LCTRL+RCTRL as switch keys.

QEMU Commandline for it:

```xml
  <qemu:commandline>
    <qemu:arg value="-audiodev"/>
    <qemu:arg value="driver=pa,id=pa1,server=unix:/run/user/1000/pulse/native,out.buffer-length=4000,timer-period=1000"/>
    <qemu:arg value="-object"/>
    <qemu:arg value="input-linux,id=mouse1,evdev=/dev/input/by-id/usb-Razer_Razer_DeathAdder_Essential_White_Edition-event-mouse"/>
    <qemu:arg value="-object"/>
    <qemu:arg value="input-linux,id=kbd1,evdev=/dev/input/by-id/usb-413c_Dell_KB216_Wired_Keyboard-event-kbd,grab_all=on,repeat=on"/>
    <qemu:env name="PIPEWIRE_RUNTIME_DIR" value="/run/user/1000"/>
  </qemu:commandline>
```

## Important files modified or added

`/usr/local/bin/vfio-pci-override.sh`
```bash
#!/bin/sh

# We know what GPU we want to pass through so we'll define it. Use lspci to find the VGA compat. controller and Audio device of your GPU.
DEVS="0000:01:00.0 0000:01:00.1"

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
MODULES=(vfio_pci vfio vfio_iommu_type1 vfio_virqfd i915)
# ...
FILES=(/usr/local/bin/vfio-pci-override.sh)
# ...
# Add modconf if it doesn't exist, usually after autodetect. Order matters. 
HOOKS="base udev autodetect modconf ..."
```

`/etc/modprobe.d/kvm.conf`
```
# Ignoring MSRS solves bugcheck issue with 1809+ guests. Second option prevents clogging up dmesg.
options kvm ignore_msrs=1
options kvm report_ignored_msrs=0

# Usually on by default, but just to be safe since we will use Hyper-V virtualization for security such as Windows Sandbox and Windows Defender Application Guard.
options kvm_intel nested=1
```

`/etc/modprobe.d/vfio.conf`
```
install vfio-pci /usr/local/bin/vfio-pci-override.sh
options vfio-pci ids=10de:1b81,10de:10f0
```

Warning: Do not enable unsafe interrupts. Disabling unsafe interrupts is for CPUs that has poor\/legacy IOMMU and doesn't support interrupt remapping. You might need this if you're using the ACS patch. You're killing IOMMU anyways by using ACS patch, killing all virtualization-based security protections and risk of malicious devices attacking your host easier.
<br><br>Disabling unsafe interrupts and using Message Signaled Interrupts can allow devices to spoof interrupts from other devices, thus VM escaping by getting host DMA write access to the interrupt block from the guest. This is because MSI interrupts are triggered by a DMA write to a special address range. Interrupt remapping in newer IOMMU protects this address range. If your CPU doesn't support interrupt remapping, throw it out and get something from at least 2013.

If you absolutely need it, check `dmesg` and see if it tells you to use it.

`/etc/default/grub` (Kernel commandline)
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet nohz_full=1,2,3,4,7,8,9,10 rcu_nocbs=1,2,3,4,7,8,9,10 isolcpus=1,2,3,4,7,8,9,10 kvm.nx_huge_pages=force pti=on page_poison=1 mce=0 random.trust_cpu=off efi=disable_early_pci_dma slab_nomerge slub_debug=FZP page_alloc.shuffle=1 transparent_hugepage=never default_hugepagesz=1G hugepagesz=1G hugepages=14 vsyscall=none i915.enable_fbc=1 vfio-pci.ids=10de:1b81,10de:10f0 intel_iommu=on iommu=pt rd.driver.pre=vfio-pci  ..."
```

<hr>

## Static Hugepages

Take note of the following above:
- `transparent_hugepage=never`
- `default_hugepagesz=1G`
- `hugepagesz=1G`
- `hugepages=14`

This is allocating huge pages at boot time.
<br>
We are using static huge pages for improved performance. We set the page files to 1GB each and allocate 14 of them. The VM has 12GB of memory allocated. It usually requires some extra pages rather than the exact else it fails to launch.

We're disabling transparent hugepages as it can hinder performance from the kernel dynamically allocating hugepages, increasing CPU usage. Allocating huge pages at boot time reduces memory fragmentation the most.
<br>See https://pingcap.com/blog/why-we-disable-linux-thp-feature-for-databases

While modern CPUs should be able to do 1GB pages, always double check:
<br>
`cat /proc/cpuinfo | grep -i 'pdpe1gb'`
<br>
If there is output, you can safely set page sizes to 1GB. Otherwise, you'll need to use 2MB pages and calculate how many pages you need to allocate.

Set this line in your fstab to be sure hugepages are allocated, and use noatime of course.
```
hugetlbfs       /dev/hugepages  hugetlbfs       noatime,pagesize=1G        0 0
```

Add yourself to the `libvirt` and `kvm` group: `sudo gpasswd -a $USER kvm,libvirt`

Then add the following to your XML file:
```xml
  <memoryBacking>
    <hugepages/>
    <nosharepages/>
    <locked/>
  </memoryBacking>
```
<hr>

_continued..._

`/etc/libvirt/qemu.conf`
```
nographics_allow_host_audio = 1
# ...

# ...
user = "zanthed" # your user
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

Note you will absolutely want to install the virtio-input driver if you take this route instead of a USB passthrough. The default driver causes very, very, very high DPC latency.

<hr>

# Installation
Attach your installation media.
<br>
Emulate a SATA disk instead of SCSI for now to avoid driver issues.

Install as normal, setup, everything. Detach installation media after reboot. Boot normally. Now proceed to setting up drivers and networking.

<hr>

# Post-installation (and performance tweaks!)

## Networking
Create a bridged connection with nm-connection-editor. Because the "Network Interfaces" tab was removed from virt-manager<sup>[source](https://listman.redhat.com/archives/virt-tools-list/2018-October/msg00032.html)</sup> you will need to do it via virsh.

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
<interface type="bridge">
  <mac address="52:54:00:71:69:a7"/>
  <source bridge="br0"/>
  <model type="virtio-net-pci"/>
</interface>

```

`virtio-net-pci` is used over `virtio` for some performance and to solve possible connection issues using some Hyper-V enlightenments that don't occur on `virtio-net-pci`.

If you pass through a physical network card, check if it can do SR-IOV. It's blazing fast. Intel cards can normally do this:
<br>
`lspci -s "Ethernet controller ID here" -vvv | grep -i "Root"`
<br>
If there is output, replace the model type with `sr-iov`.

Model of network card is virtio. Emulating a real card is very CPU intensive and has a lot of overhead compared to virtio which uses paravirtualization. Though, passing through real hardware is always better. I suggest purchasing an Intel card that can do hardware timestamps.
<br>
Windows requires NetKVM driver from virtio drivers to utilize virtio model for paravirtualization.

If you use `virtio` instead of `virtio-net-pci`, set your driver queues for multiqueue properly to allow faster packet processing!
<br>See https://pve.proxmox.com/pve-docs/chapter-qm.html#qm_network_device

## Hyper-V enlightenments, APIC, SMM, IOAPIC driver, and vmport

Special things to speed up VM performance:
```xml
  <features>
    <acpi/>
    <apic>
    <hyperv>
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vpindex state="on"/>
      <runtime state="on"/>
      <synic state="on"/>
      <stimer state="on"/>
      <reset state="on"/>
      <vendor_id state="on" value="eridanampora"/>
      <frequencies state="on"/>
      <reenlightenment state='on'/>
      <tlbflush state='on'/>
    </hyperv>
    <vmport state="off"/>
    <smm state="on"/>
    <ioapic driver="kvm"/>
  </features>
```

AMD users might need to use
`<apic eoi="on"/>

## Passthrough host CPU cache and enable CPU features
```xml
  <cpu mode="host-passthrough" check="full" migratable="on">
    <topology sockets='1' dies='1' cores='4' threads='2'/>
  </cpu>
```

Intel users do not need to set the require policy on the following features:
```xml
    <feature policy="require" name="invtsc"/>
    <feature policy="require" name="vmx"/>
    <feature policy="require" name="topoext"/>
```
This is only necessary if you are on AMD Ryzen.

## virtio-scsi (and Virtio drivers)
Grab the ISO for Fedora's Virtio drivers: https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/
<br>_Note, UEFI Secure Boot guests: As these drivers are rebuilds of Red Hat's virtio drivers due to their licensing, they are not signed the same way that Windows 10 requires now. Windows 10 now requires, as of 1803 to have drivers signed by Microsoft's WHQL, otherwise they will fail to load. If you use Secure Boot on the guest like me you will need to obtain Red Hat's virtio drivers instead of Fedora's rebuilds. If you use Scream audio, the developer does sign their driver to get around Windows 10 forbidding unsigned drivers, but it is impossible to get it WHQL approved and signed due to the way it works._

_Otherwise, continue and ignore._

Mount it, run `virtio-win-guest-tools.exe`. Install all the drivers and shut down.

Create a virtio-scsi controller:
```xml
<controller type="scsi" index="0" model="virtio-scsi">
  <driver queues="5" iothread="1"/>
</controller>
```

**Important!** The Arch Wiki poorly documents that to increase performance, you should use multiqueue by setting `queues` to `8`. While this is true, setting too many queues than your hardware can handle can cause lockups and thrash the host CPU, decreasing performance severely on heavy I/O operations. You should use no more than the amount of vCPUs you have on your guest. If you have 6 vCPUs, including hyperthreading, use 6. I personally drop 1 queue just to be safe.

Create a SCSI raw block disk using your real drive:
```xml
<disk type="block" device="disk">
  <driver name="qemu" type="raw" cache="none" io="native" discard="unmap"/>
  <source dev="/dev/disk/by-id/ata-Samsung_SSD_860_EVO_250GB_S3YHNX1KB15676F"/>
  <target dev="sda" bus="scsi"/>
  <address type="drive" controller="0" bus="0" target="0" unit="0"/>
</disk>
```

**Important!** The AIO (asynchronous I/O) is a major performance factor based on your hardware. The Arch wiki, again, poorly documents and even gets this section completely wrong. 

Arch Wiki! Let me correct your incorrect advice: Native does **not** use a single threaded synchronous model. That's not how proper asynchronous processing works. It also does not "lower" peak performance. Userspace is slower than kernel space. `threads` uses userspace threads. `native` uses kernel space threads. Spawning dozens of threads to do something in userspace is never more efficient just because you have "enough host cores". If you need to spawn dozens of threads, you need to consider using kernel space for better efficiency and avoiding host lockups.

Now, back to the main topic.

QEMU can use two AIO methods: the userspace method by spawning dozens of userspace threads (this is not the same as I/O threads) and the kernel method. Both asynchronous, but are very difference in performance.

Using the userspace method (`io='threads'`) will spawn and scale userspace threads based on the amount of guest I/O operations. This is very compatible and works in all environments, but it is not as efficient as kernel AIO.

Using the kernel AIO (`cache=none` `io=native`) is the fastest. This is the best choice for performance for block devices. RAW and QCOW2 (**with** pre-allocation) also benefit the best from this. Warning though if you do not pre-allocate, you will find using kernel AIO can decrease performance due to writing to not yet allocated sectors and lock up the threads.

If you happen to have more than one VM and all use `native`, you might find yourself failing to start VMs because you reached the max AIO requests. To see the current value, run `# sysctl fs.aio-nr fs.aio-max-nr`.

Increase `fs.aio-max-nr` to a higher number. E.g. `sysctl -w fs.aio-max-nr=4194304`

**TLDR: Use `cache=none io=native`.**

## NVIDIA Drivers 
I'll assume you have already got the process of passing through your GPU done and now you just need drivers.

NVIDIA drivers before v465 will require you to simply spoof the Hyper-V vendor ID to avoid error 43:
```xml
  <features>
    <hyperv>
      <vendor_id state="on" value="eridanampora"/>
    </hyperv>
  </features>
```
  _NVIDIA actually admits that this error 43 issue is a bug and was not intended, but they made fixing this a low priority until 465. In the past, you had to hide the KVM leaf and spoof the Hyper-V vendor ID. However, they did fix in 393 (?) that hiding the KVM leaf was no longer necessary._

## System and VM internal clock
Very important so you don't suffer the pain of awful latency from things like HPET or a clock that fails to catchup properly which would induce DPC latency.

Note: Windows machines use `localtime` unlike Linux machines which use `rtc`! Very important to set clock offset to localtime. This is related to Linux and Windows dual booting showing incorrect times because Windows is trying to use `localtime` where Linux is using `rtc` and are causing conflicts.
```xml
  <clock offset="localtime">
    <timer name="rtc" tickpolicy="catchup" track="guest"/>
    <timer name="pit" tickpolicy="delay"/>
    <timer name="tsc" present="yes" mode="native"/>
    <timer name="hpet" present="no"/>
    <timer name="kvmclock" present="no"/>
    <timer name="hypervclock" present="yes"/>
  </clock>
```

In your BIOS, disable HPET and make sure Linux is using TSC:
<br>
`cat /sys/devices/system/clocksource/clocksource0/current_clocksource`

See Red Hat's guide on optimizing the host TSC for low latency\/real time: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_for_real_time/8/html-single/tuning_guide/index

## Message signal-based interrupts

Line-based interrupts are shit. And a lot more hardware supports MSI than just your GPU. The effects of using MSI compared to Line-based are _tremendous_ with a QEMU KVM.

Use [MSI_util_v3](http://www.mediafire.com/file/ewpy1p0rr132thk/MSI_util_v3.zip/file) ([original thread](https://forums.guru3d.com/threads/windows-line-based-vs-message-signaled-based-interrupts-msi-tool.378044/)) and enable MSI for all devices that can support it and reboot the guest. If you pass through audio via Pulseaudio like me, the latency and quality will be a LOT better. Hardware acceleration will be smoother.

**DO. NOT. ENABLE. MSI. FOR. DEVICES. THAT. DO. NOT. SUPPORT. IT.** 9/10 times your system will be rendered unbootable until you restore the registry hive where the key is located, usually by a System Restore or booting with last known good configuration.

## Bitlocker
Create a vTPM 2.0.

Requires `swtpm`. 
<br>
`sudo pacman -S swtpm`

```xml
<tpm model="tpm-crb">
  <backend type="emulator" version="2.0"/>
  <alias name="tpm0"/>
</tpm>
```

## Reducing VM presence
_(Not meant to be exhaustive. Hypervisor detection is very easy no matter how hard you try, but you can do some things to get around basic detections such as Faceit Anti Cheat, ESEA, and EasyAC. Do not use a VM for secure exam testing or tournament matches. Even if your intentions are not illicit, they are always forbidden if found to be using one after the fact.)_

Pass through as much physical hardware as you can through IOMMU and remove unused hardware like SPICE displays and graphics and serial controllers.

Hide certain timers (hiding hypervclock may impact performance slightly):
<br>
```xml
  <clock offset="localtime">
    <timer name="kvmclock" present="no"/>
    <timer name="hypervclock" present="no"/>
  </clock>
```

Use host's SMBIOS:
```xml
  <os>
    <smbios mode="host"/>
  </os>
```

[Hide KVM leaf and spoof Hyper-V vendor ID](#nvidia-drivers)

```xml
  <kvm>
    <hidden state="on">
  </kvm>
```

Disable `hypervisor` feature and pass through your CPU features and model:
```xml
  <cpu mode="host-passthrough" check="full" migratable="on">
    <feature policy="disable" name="hypervisor"/>
  </cpu>
```

Remove as many SPICE, virtio, QEMU, etc drivers and hardware. Pass through as much physical hardware as you can.

## Minor performance tweaks
Blacklist the iTCO_wdt (kernel watchdog) module:
<br>
`sudo echo "blacklist iTCO_wdt" > /etc/modprobe.d/nowatchdog.conf`
<br>
Then run `sudo mkinitcpio -P`

Prevent ARP Flux from networks in same segment<sup>[1](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html-single/virtualization_tuning_and_optimization_guide/index#sect-Virtualization_Tuning_Optimization_Guide-Networking-General_Tips)</sup>:
<br>
`sudo echo "net.ipv4.conf.all.arp_filter = 1" > /etc/sysctl.d/30-arpflux.conf`

Try using performance and latency-optimized kernels like linux-xanmod (https://xanmod.org/) or linux-zen. I personally use xanmod. Both also include the ACS patches. Experiment with real-time kernels too. Everyone's hardware is different.

Installed `tuned` and `tuned-adm` and set the profile to `virtual-host` to optimize the host for virtualizing guests.

# CPU Pinning, Interrupts, Affinity, Governors, Topology, Isolating, and Priorities
This warrants an entirely separate section because of how much of an impact this can make while also requiring a decent amount of time. And I mean a _major_ impact. Even after doing all those tweaks above, I still had awful DPC latency and stutters. After just a bit of messing with interrupts and pinning, I am almost consistently under 800 µs.

My system only has a single CPU and six cores twelve threads so messing with NUMA nodes is not documented here unfortunately.

## Topology

Find out how many cores and threads you want to give your VM. **Leave at least 2 cores for the host, otherwise both the guest and host performance will suffer. More is not better.** For me, I'm giving it 4 cores 8 threads. Set your CPU topology to that, and of course pass through your CPU features by setting CPU mode to host-passthrough (unless you have special AMD CPUs that require EPYC emulation for old QEMU versions)
```xml
    <topology sockets="1" dies="1" cores="4" threads="2"/>
```
This says we will have one socket, 4 cores, and 8 threads through hyperthreading. Our CPU pinning will match this too. If you only want physical CPUs and no hyperthreading, set core count to the amount of cores you want and threads to 1. e.g.
```xml
    <topology sockets="1" dies="1" cores="8" threads="1"/>
```

## Isolating the CPUs
We need to isolate the pinned CPUs from being used by the host. This is pretty easy through the kernel arguments `nohz_full, rcu_nocbs, isolcpus`. 

Append the following arguments based on your pinned CPUs:
`nohz_full=0-3,6-9 rcu_nocbs=0-3,6-9 isolcpus=0-3,6-9`

Update your bootloader config and reboot.

## _Interrupts, Governors, and Affinity coming soon. I need sleep._
TODO: Document interrupt pinning, CPU governors, systemd tweaks, memballoon disabling.
