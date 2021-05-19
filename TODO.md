todo.

secure boot: https://github.com/infokiller/win10-vm.git

`GRUB_CMDLINE_LINUX_DEFAULT="quiet threadirqs nohz_full=0-3,6-9 rcu_nocbs=0-3,6-9 isolcpus=0-3,6-9 rc
u_nocb_poll  kvm.nx_huge_pages=force pti=on page_poison=1 mce=0 pcie_aspm=off random.trust_cpu=off e
fi=disable_early_pci_dma slab_nomerge slub_debug=FZP page_alloc.shuffle=1 transparent_hugepage=never
 default_hugepagesz=1G hugepagesz=1G hugepages=14 vsyscall=none i915.enable_fbc=1 vfio-pci.ids=10de:
1b81,10de:10f0 intel_iommu=on iommu=pt rd.driver.pre=vfio-pci cryptdevice=UUID=be1bd61a-675c-42ed-a3
20-01a5267e3b82:luks-be1bd61a-675c-42ed-a320-01a5267e3b82 root=/dev/mapper/luks-be1bd61a-675c-42ed-a
320-01a5267e3b82 splash rd.udev.log_priority=3 vt.global_cursor_default=0 systemd.unified_cgroup_hie
rarchy=1 resume=/dev/mapper/luks-2d62bafe-c7f0-46b8-8d6a-459878d3ca16 loglevel=3"`

```
 ╰─λ cat /etc/modprobe.d/noime.conf
File: /etc/modprobe.d/noime.conf
# Intel VPRO remote access technology driver.
blacklist mei
blacklist mei_me
```

```
 ╰─λ cat /etc/modprobe.d/kvm.conf
File: /etc/modprobe.d/kvm.conf
# These should solve some bugcheck issues along with the patched vBIOS.
options kvm ignore_msrs=1
options kvm report_ignored_msrs=0
options kvm_intel enable_apicv=Y enable_shadow_vmcs=Y nested=Y nested=1
```

```
 ╰─λ lspci -vvn
00:00.0 0600: 8086:3ec2 (rev 07)
	DeviceName: Onboard - Other
	Subsystem: 1462:7b48
	Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
	Status: Cap+ 66MHz- UDF- FastB2B+ ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort+ >SERR- <PERR- INTx-
	Latency: 0
	IOMMU group: 0
	Capabilities: <access denied>
	Kernel driver in use: skl_uncore
	Kernel modules: ie31200_edac

00:01.0 0604: 8086:1901 (rev 07) (prog-if 00 [Normal decode])
	Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0, Cache Line Size: 64 bytes
	Interrupt: pin A routed to IRQ 122
	IOMMU group: 1
	Bus: primary=00, secondary=01, subordinate=01, sec-latency=0
	I/O behind bridge: 0000e000-0000efff [size=4K]
	Memory behind bridge: dd000000-de0fffff [size=17M]
	Prefetchable memory behind bridge: 0000002fd0000000-0000002fe1ffffff [size=288M]
	Secondary status: 66MHz- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort+ <SERR- <PERR-
	BridgeCtl: Parity- SERR+ NoISA- VGA- VGA16+ MAbort- >Reset- FastB2B-
		PriDiscTmr- SecDiscTmr- DiscTmrStat- DiscTmrSERREn-
	Capabilities: <access denied>
	Kernel driver in use: pcieport

00:02.0 0300: 8086:3e92 (prog-if 00 [VGA controller])
	DeviceName: Onboard - Video
	Subsystem: 1462:7b48
	Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0, Cache Line Size: 64 bytes
	Interrupt: pin A routed to IRQ 128
	IOMMU group: 2
	Region 0: Memory at 2ffe000000 (64-bit, non-prefetchable) [size=16M]
	Region 2: Memory at 2fc0000000 (64-bit, prefetchable) [size=256M]
	Region 4: I/O ports at f000 [size=64]
	Expansion ROM at 000c0000 [virtual] [disabled] [size=128K]
	Capabilities: <access denied>
	Kernel driver in use: i915
	Kernel modules: i915

00:08.0 0880: 8086:1911
	DeviceName: Onboard - Other
	Subsystem: 1462:7b48
	Control: I/O- Mem- BusMaster- SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Interrupt: pin A routed to IRQ 255
	IOMMU group: 3
	Region 0: Memory at 2fff027000 (64-bit, non-prefetchable) [disabled] [size=4K]
	Capabilities: <access denied>

00:14.0 0c03: 8086:a2af (prog-if 30 [XHCI])
	DeviceName: Onboard - Other
	Subsystem: 1462:7b48
	Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
	Status: Cap+ 66MHz- UDF- FastB2B+ ParErr- DEVSEL=medium >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0
	Interrupt: pin A routed to IRQ 129
	IOMMU group: 4
	Region 0: Memory at 2fff010000 (64-bit, non-prefetchable) [size=64K]
	Capabilities: <access denied>
	Kernel driver in use: xhci_hcd
	Kernel modules: xhci_pci

00:14.2 1180: 8086:a2b1
	DeviceName: Onboard - Other
	Subsystem: 1462:7b48
	Control: I/O- Mem+ BusMaster- SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Interrupt: pin C routed to IRQ 255
	IOMMU group: 4
	Region 0: Memory at 2fff026000 (64-bit, non-prefetchable) [size=4K]
	Capabilities: <access denied>

00:16.0 0780: 8086:a2ba
	DeviceName: Onboard - Other
	Subsystem: 1462:7b48
	Control: I/O- Mem- BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0
	Interrupt: pin A routed to IRQ 255
	IOMMU group: 5
	Region 0: Memory at 2fff025000 (64-bit, non-prefetchable) [disabled] [size=4K]
	Capabilities: <access denied>
	Kernel modules: mei_me

00:17.0 0106: 8086:a282 (prog-if 01 [AHCI 1.0])
	DeviceName: Onboard - SATA
	Subsystem: 1462:7b48
	Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
	Status: Cap+ 66MHz+ UDF- FastB2B+ ParErr- DEVSEL=medium >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0
	Interrupt: pin A routed to IRQ 127
	IOMMU group: 6
	Region 0: Memory at dc104000 (32-bit, non-prefetchable) [size=8K]
	Region 1: Memory at dc107000 (32-bit, non-prefetchable) [size=256]
	Region 2: I/O ports at f090 [size=8]
	Region 3: I/O ports at f080 [size=4]
	Region 4: I/O ports at f060 [size=32]
	Region 5: Memory at dc106000 (32-bit, non-prefetchable) [size=2K]
	Capabilities: <access denied>
	Kernel driver in use: ahci

00:1b.0 0604: 8086:a2e7 (rev f0) (prog-if 00 [Normal decode])
	Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0, Cache Line Size: 64 bytes
	Interrupt: pin A routed to IRQ 123
	IOMMU group: 7
	Bus: primary=00, secondary=02, subordinate=02, sec-latency=0
	I/O behind bridge: [disabled]
	Memory behind bridge: [disabled]
	Prefetchable memory behind bridge: [disabled]
	Secondary status: 66MHz- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort+ <SERR- <PERR-
	BridgeCtl: Parity- SERR+ NoISA- VGA- VGA16+ MAbort- >Reset- FastB2B-
		PriDiscTmr- SecDiscTmr- DiscTmrStat- DiscTmrSERREn-
	Capabilities: <access denied>
	Kernel driver in use: pcieport

00:1b.4 0604: 8086:a2eb (rev f0) (prog-if 00 [Normal decode])
	Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0, Cache Line Size: 64 bytes
	Interrupt: pin A routed to IRQ 124
	IOMMU group: 8
	Bus: primary=00, secondary=03, subordinate=03, sec-latency=0
	I/O behind bridge: 0000d000-0000dfff [size=4K]
	Memory behind bridge: da000000-dc0fffff [size=33M]
	Prefetchable memory behind bridge: 0000002fe8000000-0000002ff3ffffff [size=192M]
	Secondary status: 66MHz- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort+ <SERR- <PERR-
	BridgeCtl: Parity- SERR+ NoISA- VGA- VGA16+ MAbort- >Reset- FastB2B-
		PriDiscTmr- SecDiscTmr- DiscTmrStat- DiscTmrSERREn-
	Capabilities: <access denied>
	Kernel driver in use: pcieport

00:1c.0 0604: 8086:a290 (rev f0) (prog-if 00 [Normal decode])
	Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0, Cache Line Size: 64 bytes
	Interrupt: pin A routed to IRQ 125
	IOMMU group: 9
	Bus: primary=00, secondary=04, subordinate=04, sec-latency=0
	I/O behind bridge: [disabled]
	Memory behind bridge: [disabled]
	Prefetchable memory behind bridge: [disabled]
	Secondary status: 66MHz- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort+ <SERR- <PERR-
	BridgeCtl: Parity- SERR+ NoISA- VGA- VGA16+ MAbort- >Reset- FastB2B-
		PriDiscTmr- SecDiscTmr- DiscTmrStat- DiscTmrSERREn-
	Capabilities: <access denied>
	Kernel driver in use: pcieport

00:1c.3 0604: 8086:a293 (rev f0) (prog-if 00 [Normal decode])
	Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0, Cache Line Size: 64 bytes
	Interrupt: pin D routed to IRQ 126
	IOMMU group: 10
	Bus: primary=00, secondary=05, subordinate=05, sec-latency=0
	I/O behind bridge: 0000c000-0000cfff [size=4K]
	Memory behind bridge: de200000-de2fffff [size=1M]
	Prefetchable memory behind bridge: [disabled]
	Secondary status: 66MHz- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort+ <SERR- <PERR-
	BridgeCtl: Parity- SERR+ NoISA- VGA- VGA16+ MAbort- >Reset- FastB2B-
		PriDiscTmr- SecDiscTmr- DiscTmrStat- DiscTmrSERREn-
	Capabilities: <access denied>
	Kernel driver in use: pcieport

00:1f.0 0601: 8086:a2c9
	DeviceName: Onboard - Other
	Subsystem: 1462:7b48
	Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
	Status: Cap- 66MHz- UDF- FastB2B- ParErr- DEVSEL=medium >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0
	IOMMU group: 11

00:1f.2 0580: 8086:a2a1
	DeviceName: Onboard - Other
	Subsystem: 1462:7b48
	Control: I/O- Mem- BusMaster- SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
	Status: Cap- 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	IOMMU group: 11
	Region 0: Memory at dc100000 (32-bit, non-prefetchable) [disabled] [size=16K]

00:1f.3 0403: 8086:a2f0
	DeviceName: Onboard - Sound
	Subsystem: 1462:9b48
	Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 32, Cache Line Size: 64 bytes
	Interrupt: pin A routed to IRQ 131
	IOMMU group: 11
	Region 0: Memory at 2fff020000 (64-bit, non-prefetchable) [size=16K]
	Region 4: Memory at 2fff000000 (64-bit, non-prefetchable) [size=64K]
	Capabilities: <access denied>
	Kernel driver in use: snd_hda_intel
	Kernel modules: snd_hda_intel

00:1f.4 0c05: 8086:a2a3
	DeviceName: Onboard - Other
	Subsystem: 1462:7b48
	Control: I/O+ Mem+ BusMaster- SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
	Status: Cap- 66MHz- UDF- FastB2B+ ParErr- DEVSEL=medium >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Interrupt: pin A routed to IRQ 16
	IOMMU group: 11
	Region 0: Memory at 2fff024000 (64-bit, non-prefetchable) [size=256]
	Region 4: I/O ports at f040 [size=32]
	Kernel driver in use: i801_smbus
	Kernel modules: i2c_i801

01:00.0 0300: 10de:1b81 (rev a1) (prog-if 00 [VGA controller])
	Subsystem: 1462:3301
	Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0, Cache Line Size: 64 bytes
	Interrupt: pin A routed to IRQ 132
	IOMMU group: 1
	Region 0: Memory at dd000000 (32-bit, non-prefetchable) [size=16M]
	Region 1: Memory at 2fd0000000 (64-bit, prefetchable) [size=256M]
	Region 3: Memory at 2fe0000000 (64-bit, prefetchable) [size=32M]
	Region 5: I/O ports at e000 [size=128]
	Expansion ROM at de000000 [disabled] [size=512K]
	Capabilities: <access denied>
	Kernel driver in use: vfio-pci
	Kernel modules: nouveau, nvidia_drm, nvidia

01:00.1 0403: 10de:10f0 (rev a1)
	Subsystem: 1462:3301
	Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0, Cache Line Size: 64 bytes
	Interrupt: pin B routed to IRQ 133
	IOMMU group: 1
	Region 0: Memory at de080000 (32-bit, non-prefetchable) [size=16K]
	Capabilities: <access denied>
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel

03:00.0 0300: 10de:1244 (rev a1) (prog-if 00 [VGA controller])
	Subsystem: 3842:1559
	Control: I/O- Mem- BusMaster- SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Interrupt: pin A routed to IRQ 255
	IOMMU group: 12
	Region 0: Memory at da000000 (32-bit, non-prefetchable) [disabled] [size=32M]
	Region 1: Memory at 2fe8000000 (64-bit, prefetchable) [disabled] [size=128M]
	Region 3: Memory at 2ff0000000 (64-bit, prefetchable) [disabled] [size=64M]
	Region 5: I/O ports at d000 [disabled] [size=128]
	Expansion ROM at dc000000 [disabled] [size=512K]
	Capabilities: <access denied>
	Kernel modules: nouveau, nvidia_drm, nvidia

03:00.1 0403: 10de:0bee (rev a1)
	Subsystem: 3842:1559
	Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0, Cache Line Size: 64 bytes
	Interrupt: pin B routed to IRQ 17
	IOMMU group: 12
	Region 0: Memory at dc080000 (32-bit, non-prefetchable) [size=16K]
	Capabilities: <access denied>
	Kernel driver in use: snd_hda_intel
	Kernel modules: snd_hda_intel

05:00.0 0200: 10ec:8168 (rev 15)
	Subsystem: 1462:7b48
	Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0, Cache Line Size: 64 bytes
	Interrupt: pin A routed to IRQ 19
	IOMMU group: 13
	Region 0: I/O ports at c000 [size=256]
	Region 2: Memory at de204000 (64-bit, non-prefetchable) [size=4K]
	Region 4: Memory at de200000 (64-bit, non-prefetchable) [size=16K]
	Capabilities: <access denied>
	Kernel driver in use: r8169
	Kernel modules: r8169
```

iptables in interrupt_perf_script

```
LOCAL CPU MASK 00000fff = 111111111111
BANNED CPUS 0000079e = 11110011110

111111111111
11110011110
00000000000000000000000000000001 = cpu 0
00000000000000000000000000000010 = 1
00000000000000000000000000000011 = 0,1
00000000000000000000000000000100 = 2
00000000000000000000000000000111 = 0,1,2
00000000000000000000000000001000 = 3
00000000000000000000000000010000 = 4
00000000000000000000000000100000 = 5
00000000000000000000000001000000 = 6
00000000000000000000000010000000 = 7
00000000000000000000000100000000 = 8
00000000000000000000001000000000 = 9
00000000000000000000010000000000 = 10
00000000000000000000100000000000 = 11

00000000000000000000001111001111


000003cf
```