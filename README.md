# Intel Gen 12 vGPU (SR-IOV) on Proxmox
This Guide is to help you to virtualize Intel Gen 12 iGPU as vGPU for multiple VMs with hardware acceleration and video encoding/decoding.
## Introduction
This is not really for gaming since Intel's iGPU is never good in gaming, and when 7 VMs is sharing the iGPU, it is really only about to do video decoding such as Youtube and RDP acceleration without using CPU to process graphic.
After you finish this guide setup, you can also visit my another project [UpinelBetterRDP](https://github.com/Upinel/BetterRDP) to make your RDP session make use of the vGPU.
## Disclaimer
This guide is not official support by Proxmox so use it at your own risk.
## Credits
Strongz DKMS module is the key to make this possible ([https://github.com/strongtz/i915-sriov-dkms](https://github.com/strongtz/i915-sriov-dkms?ref=michaels-tinkerings))

## Enviroment
My Enviroment:
Model: Intel NUC12 Pro Wall Street Canyon (NUC12WSKi5)
CPU: Intel 12th Gen i5 1240P (12 Core 16 Threads)
Ram: Samsung 64GB DDR4
Storage: Samsung 980 Pro 2TB NVME + Samsung 870 Evo 4TB SATA
GPU: Intel Iris Xe Graphic (80EU)

## Prerequisite
Please make sure:-
Turn on VT-d (IOMMU) and SR-IOV in BIOS
Proxmox Virtual Environment 8+ (I am using Proxmox 8.1.4) **installed with GRUB bootloader** (Should be by default)
EFI **Enabled** and Secure Boot **Disabled**
Linux Kernal 6.1 or newer
check with
```bash
uname -r
```
if you see the kernel version is anything larger than 6.1, you are good to go. In theory, if you are using Proxmox 8.1.4 like me, your linux kernal should be 6.5+.  
If your kernal is lower than 6.1, pleasee check with **Appedix 1** for kernal update

## Proxmox Setup
### DKMS Setup
1. Update apt and install pve-headers, execute it one by one
```bash
apt update && apt install pve-headers-$(uname -r)
apt update && apt install git pve-headers mokutil
rm -rf /var/lib/dkms/i915-sriov-dkms*
rm -rf /usr/src/i915-sriov-dkms*
rm -rf ~/i915-sriov-dkms
KERNEL=$(uname -r); KERNEL=${KERNEL%-pve}
```
2. Clone i915-sriov-dkms repo, execute it one by one
```bash
git clone https://github.com/strongtz/i915-sriov-dkms.git
cd ~/i915-sriov-dkms
cp -a ~/i915-sriov-dkms/dkms.conf{,.bak}
sed -i 's/"@_PKGBASE@"/"i915-sriov-dkms"/g' ~/i915-sriov-dkms/dkms.conf
sed -i 's/"@PKGVER@"/"'"$KERNEL"'"/g' ~/i915-sriov-dkms/dkms.conf
sed -i 's/ -j$(nproc)//g' ~/i915-sriov-dkms/dkms.conf
cat ~/i915-sriov-dkms/dkms.conf
```
3. Install DKMS module
```bash
apt install --reinstall dkms -y
dkms add .
cd /usr/src/i915-sriov-dkms-$KERNEL
dkms status
```
4. Build the i915-sriov-dkms
```bash
dkms install -m i915-sriov-dkms -v $KERNEL -k $(uname -r) --force -j 1
dkms status
```
5. (Optional) If you are
```bash
mokutil --import /var/lib/dkms/mok.pub
```
### GRUB Bootloader Setup
```bash
cp -a /etc/default/grub{,.bak}
sudo sed -i '/^GRUB_CMDLINE_LINUX_DEFAULT/c\GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt i915.enable_guc=3 i915.max_vfs=7"' /etc/default/grub
update-grub
update-initramfs -u -k all
apt install sysfsutils -y
```

### Create vGPU
2\. Now we need to find which PCIe bus the VGA card is on. It’s typically **00:02.0**. 

```bash
lspci | grep VGA
```

3\. Run the following command and modify the PCIe bus number if needed. In this case I’m using **00:02.0**. To verify the file was modified, cat the file and ensure it was modified.

```bash
echo "devices/pci0000:00/0000:00:02.0/sriov_numvfs = 7" > /etc/sysfs.conf
```

```bash
cat /etc/sysfs.conf
```

### Finishing vGPU setup
Now reboot your Host and you should able to see the minor **PCIe IDs 1-7** and finally **Enabled 7 VFs**
```bash
lspci | grep VGA
dmesg | grep i915
```
Now that the Proxmox host is ready, we can install and configure Windows 10/11. 

## Windows Installation

## Appedixies
### Appedix 1 - Kernal lower than 6.1
(not yet ready, come back later)

### Appedix 2 - EFI  **Enabled** and Secure Boot **Enabled**
(not yet ready, come back later)
