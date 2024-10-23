# Introduction

This is a guide for configuring Proxmox 8.2 iGPU passthrough on an Intel NUC6CAYH (9th generation [Apollo Lake J3455](https://www.intel.com/content/www/us/en/products/sku/95594/intel-celeron-processor-j3455-2m-cache-up-to-2-30-ghz/specifications.html)). My goal is to host headless Linux VMs with hardware accelerated video encoding, which may not match other usecases (e.g. running Windows). 

Unfortunately there are many permutations of different options that can be applied, and getting things working isn't straightforward (nor do I fully understand everything). Things are also not completely stable, so proceed at your own risk.

# Step 0: Downgrade kernel

If you're using Proxmox 8.2 then you may need to downgrade your kernel from 6.8.x due to issues with Intel passthrough support (see [here](https://www.reddit.com/r/Proxmox/comments/1cg2yzl/question_about_downgrading_to_65133pvesigned/), [here](https://www.thomas-krenn.com/de/wiki/Known_Issues_Proxmox_VE_8.2) and [here](https://forum.proxmox.com/threads/proxmox-8-2-kernel-6-8-breaks-igpu-passthrough-for-uhd630.146256/)). I encountered an issue where Proxmox wouldn't boot and I'd have to disable various kernel options at bootup. With an older kernel there were no issues at all.

Run the following on your Proxmox host:

```
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
```
update-grub
reboot
```

See [here](https://wiki.archlinux.org/title/Intel_graphics) for more info about GuC.

# Step 2: Extracting VGA BIOS

Some guides (e.g. [here](https://wiki.eofnet.lt/wiki/Frigate#Proxmox_.2B_HassOS_.28Home_Assistant.29_.2B_Frigate_.28Intel_NUC6CAYH.29)) don't require this, but I couldn't get things working without it. A good overview (in Chinese) of the process is [here](https://www.bilibili.com/read/cv3038211/).

For convenience I've uploaded the original and fixed ROMs to this repo, so feel free to use and skip the steps below.

## 2.1: Getting ROM
To extract the VGA ROM, you need to disable EUFI boot in your bios and boot your Proxmox host in a Linux live distribution to run the following commands:

```
cd /sys/bus/pci/devices/0000:00:02.0/
echo 1 > rom
cat rom > /tmp/vga.rom
echo 0 > rom
```

## 2.2: Fixing ROM
Then transfer the file `vga.rom` someplace safe and reboot back into Proxmox. Next you need to compile and run `rom-fixer` (you can do this in a VM):

```
git clone https://github.com/awilliam/rom-parser
cd rom-parser
make
```

Then you need to modify the ROM file as follows:

```
./rom-fixer vga.rom
```

And answer the prompts with the following:

```
Modify vendor ID 8086? (y/n): n Modify
device ID 0406? ( y/n): y
New device ID: 5a85
Overwrite device ID with 5a85? (y/n): y
    Last image
ROM checksum is invalid, fix? (y/n): y
```

## 2.3: Add ROM to Proxmox

Now copy `vga.rom` back to your Proxmox host and save it in `/usr/share/kvm`

# Step 3: 

# References

* https://mae.octatile.com/index.php/intel-j4105-igdgpu-passthrough-on-proxmox/
* https://wiki.archlinux.org/title/Intel_GVT-g
* https://github.com/AkashiSN/ffmpeg-docker
* https://wiki.eofnet.lt/wiki/Frigate#Ubuntu_22.04.4_LTS_.2B_Frigate_Docker_.28ThinkCentre_M920q.29
* https://pve.proxmox.com/wiki/PCI(e)_Passthrough#_mediated_devices_vgpu_gvt_g
* https://3os.org/infrastructure/proxmox/gpu-passthrough/igpu-passthrough-to-vm/#proxmox-configuration-for-igpu-full-passthrough
* https://www.derekseaman.com/2024/07/proxmox-ve-8-2-windows-11-vgpu-vt-d-passthrough-with-intel-alder-lake.html
