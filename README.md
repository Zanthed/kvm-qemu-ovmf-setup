# KVM Qemu libvirt Windows 10 guest on Garuda Linux (Arch-based) setup and example

⚠️⚠️ Note this is not intended to be a copy and paste or follow as is for your setup, even if you have identical hardware and software. You will run into problems. This is to be used as a baseline for similiar or identical hardware if you are experiencing issues or are looking into the possibility of doing this by looking at examples.

You can find more examples on the Arch Wiki: https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF/Examples

<hr>

## Hardware:
- Motherboard: MSI Z370-A PRO
- CPU: Intel i7-8700k OC @ 4.2GHz
- GPU 1: MSI NVIDIA GTX 1070 (Host GPU)
- GPU 2: EVGA NVIDIA GTX 550 Ti (Guest GPU)
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

## Extra notes:
- I did not install Windows. I used my pre-existing Windows installation encrypted with Bitlocker (you must suspend it before doing anything otherwise you'll have a lot of fun entering your recovery key) with Virtio SCSI to maintain compatibility if I need to boot normally in the future\*:
```
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
