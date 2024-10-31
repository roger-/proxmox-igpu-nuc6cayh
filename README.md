# Introduction

This is a guide for configuring Proxmox 8.2 iGPU passthrough on an Intel NUC6CAYH (9th generation [Apollo Lake J3455](https://www.intel.com/content/www/us/en/products/sku/95594/intel-celeron-processor-j3455-2m-cache-up-to-2-30-ghz/specifications.html)). My goal is to host headless Linux VMs with hardware accelerated video encoding in GVT-d mode (full device passthrough), which may not match other usecases (e.g. running Windows). 

Unfortunately there are many permutations of different options that can be applied, and getting things working isn't straightforward (nor do I fully understand everything). Things are also not completely stable, so proceed at your own risk.

# Step 0: BIOS Configuration and Downgrade kernel

First make sure UEFI and all virtualization optionas are enabled in your BIOS (see `Advanced > Security > Security Features`). You should enable UEFI before installing Proxmox, or things may break.

If you're using Proxmox 8.2 then you should downgrade your kernel from 6.8.x due to issues with Intel passthrough support (see [here](https://www.reddit.com/r/Proxmox/comments/1cg2yzl/question_about_downgrading_to_65133pvesigned/), [here](https://www.thomas-krenn.com/de/wiki/Known_Issues_Proxmox_VE_8.2) and [here](https://forum.proxmox.com/threads/proxmox-8-2-kernel-6-8-breaks-igpu-passthrough-for-uhd630.146256/)). I encountered an issue where Proxmox wouldn't boot and I'd have to disable various kernel options at bootup. With an older kernel there were no issues at all.

Run the following on your Proxmox host:

```bash
apt install proxmox-kernel-6.5.13-5-pve-signed
apt install proxmox-headers-6.5.13-5-pve
proxmox-boot-tool kernel pin 6.5.13-5-pve
reboot
```

# Step 1: Enabling PCI passthrough on the host

On your Proxmox host edit the file `/etc/default/grub` to include:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet iommu=pt intel_iommu=on i915.enable_guc=2 video=efifb:off video=vesa:off"
```

Then run
```bash
update-grub
reboot
```

I'm not sure if GuC is necessary here but it prevents some VM warnings later, see [here](https://wiki.archlinux.org/title/Intel_graphics) for more info.

# Step 2: Extracting VGA BIOS

Some guides (e.g. [here](https://wiki.eofnet.lt/wiki/Frigate#Proxmox_.2B_HassOS_.28Home_Assistant.29_.2B_Frigate_.28Intel_NUC6CAYH.29)) don't require this, but I couldn't get things working without it. A good overview (in Chinese) of the process is [here](https://www.bilibili.com/read/cv3038211/).

For convenience I've uploaded the fixed ROM to [this repo](https://github.com/roger-/proxmox-igpu-nuc6cayh/blob/main/intel_hd500.rom), so feel free to use and skip the steps below.

## 2.1: Getting ROM
To extract the VGA ROM, you need to temporarily disable EUFI boot in your NUC BIOS and boot the physical machine in a Linux live distribution. Then get to a console and run the following commands:

```bash
cd /sys/bus/pci/devices/0000:00:02.0/
echo 1 > rom
cat rom > /tmp/intel_hd500.rom
echo 0 > rom
```

Then transfer the file `intel_hd500.rom` someplace safe (e.g. using `scp` or a USB flash drive), revert your BIOS settings and reboot back into Proxmox.

## 2.2: Fixing ROM
Next you need to compile and run `rom-fixer` (you can do this in a Proxmox VM):

```bash
git clone https://github.com/awilliam/rom-parser
cd rom-parser
make
```

Then you need to modify the ROM file as follows:

```bash
./rom-fixer intel_hd500.rom
```

And answer the prompts with the following:

```
Modify vendor ID 8086? (y/n): n
Modify device ID 0406? ( y/n): y
New device ID: 5a85
Overwrite device ID with 5a85? (y/n): y
Last image ROM checksum is invalid, fix? (y/n): y
```

## 2.3: Add ROM to Proxmox

Now copy `intel_hd500.rom` back to your Proxmox host and save it in `/usr/share/kvm`

# Step 3: Update host modules

## 3.1: Configure vfio

Add the following to `/etc/modules` to enable the vfio module:

```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

Then add this to `/etc/modprobe.d/vfio.conf`

```
options vfio-pci ids=8086:5a85,8086:5a98
```

Note that if you're using a different CPU these IDs may be different (check out `lspci`).

## 3.2: Blacklist modules

Now add this to `/etc/modprobe.d/blacklist.conf`:

```
blacklist snd_hda_intel
blacklist snd_hda_codec_hdmi
blacklist snd_soc_skl
blacklist snd_soc_avs
blacklist i915
```

Now run:

```bash
update-initramfs -u -k all
reboot
```

# Step 4: Verify configuration

After rebooting, run `dmesg | grep iomm` on your Proxmox host and verify you see something like this:

```
[    0.441307] iommu: Default domain type: Passthrough (set via kernel command line)
[    0.532935] pci 0000:00:02.0: Adding to iommu group 0
[    0.533081] pci 0000:00:00.0: Adding to iommu group 1
[    0.533108] pci 0000:00:0e.0: Adding to iommu group 2
[    0.533142] pci 0000:00:0f.0: Adding to iommu group 3
```

Next run `lspci -v` and you should see something like this:

```
00:02.0 VGA compatible controller: Intel Corporation HD Graphics 500 (rev 0b) (prog-if 00 [VGA controller])
        DeviceName:  CPU
        Subsystem: Intel Corporation HD Graphics 500
        Flags: bus master, fast devsel, latency 0, IRQ 131, IOMMU group 0
        Memory at 90000000 (64-bit, non-prefetchable) [size=16M]
        Memory at 80000000 (64-bit, prefetchable) [size=256M]
        I/O ports at f000 [size=64]
        Expansion ROM at 000c0000 [virtual] [disabled] [size=128K]
        Capabilities: [40] Vendor Specific Information: Len=0c <?>
        Capabilities: [70] Express Root Complex Integrated Endpoint, MSI 00
        Capabilities: [ac] MSI: Enable+ Count=1/1 Maskable- 64bit-
        Capabilities: [d0] Power Management version 2
        Capabilities: [100] Process Address Space ID (PASID)
        Capabilities: [200] Address Translation Service (ATS)
        Capabilities: [300] Page Request Interface (PRI)
        Kernel driver in use: vfio-pci
        Kernel modules: i915
```

The `Kernel driver in use: vfio-pci` part is important.

# Step 5: Create VM

You should be all set to create a VM as usual, but use SeaBIOS instead of EUFI (latter may cause issues). After setting it up, power it off and modify the VM settings as follows.

Determine your VM ID (something like 101) and edit the file `/etc/pve/qemu-server/<VM-ID>.conf` to add/modify the following line:

```
args: -device vfio-pci,host=00:02.0,addr=0x02,x-igd-gms=4,romfile=intel_hd500.rom,x-vga=on
vga: none
```

You can now boot up your VM, but note the KVM console won't work, so use SSH/VNC or enable serial access. 

# Step 6: Guest configuration

I noticed a few (possible benign) warnings from `dmesg`, and configuring GuC seemed to help. So inside the VM, create `/etc/modprobe.d/i915.conf` with:

```
options i915 enable_guc=2
```

Then run:

```bash
update-initramfs -u -k all
reboot
```

After rebooting, run the following in the VM and verify the output matches:

```
$ ls /dev/dri
by-path  card0  renderD128
```

Also run this and verify there are no errors:

```
dmesg | grep i915
```

# ffmpeg setup

On Alpine you may need to install the packages `linux-firmware-i915` and `intel-media-driver`. I'm not sure if QSV works however.

On Debian you need the Intel media drivers. First you need to modify `/etc/apt/sources.list` and make sure each line has `non-free-firmware non-free`. Then run:

```
sudo apt update
sudo apt install intel-media-va-driver-non-free
sudo usermod -aG video $USER
```

Alternatively, if you have Docker installed, you can also test things by running:

```bash
docker run --rm -it --device=/dev/dri -v $(pwd):/workdir akashisn/ffmpeg:7.0.2 -y -loglevel verbose -init_hw_device qsv:hw -hwaccel qsv -hwaccel_output_format qsv -extra_hw_frames 32 -i video.mp4 -c:v h264_qsv -f mp4 input_video.mp4;
```

# References

* https://mae.octatile.com/index.php/intel-j4105-igdgpu-passthrough-on-proxmox/
* https://wiki.archlinux.org/title/Intel_GVT-g
* https://github.com/AkashiSN/ffmpeg-docker
* https://wiki.eofnet.lt/wiki/Frigate#Ubuntu_22.04.4_LTS_.2B_Frigate_Docker_.28ThinkCentre_M920q.29
* https://pve.proxmox.com/wiki/PCI(e)_Passthrough#_mediated_devices_vgpu_gvt_g
* https://3os.org/infrastructure/proxmox/gpu-passthrough/igpu-passthrough-to-vm/#proxmox-configuration-for-igpu-full-passthrough
* https://www.derekseaman.com/2024/07/proxmox-ve-8-2-windows-11-vgpu-vt-d-passthrough-with-intel-alder-lake.html
