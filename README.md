**Intel Gen 12 vGPU (SR-IOV) on Proxmox**

This guide is designed to help you virtualize the 12th-generation Intel integrated GPU (iGPU) and share it as a virtual GPU (vGPU) with hardware acceleration and video encoding/decoding capabilities across multiple VMs.

**Introduction**

Although not suited for gaming due to the limited performance of Intel’s iGPU, especially when shared among multiple VMs, this setup excels at video decoding tasks like streaming YouTube and accelerating RDP sessions without burdening the CPU.

Once you complete this setup, consider enhancing your RDP experience with my project [UpinelBetterRDP](https://github.com/Upinel/BetterRDP), which leverages vGPU capabilities.

**Disclaimer**

This workaround is not officially supported by Proxmox. Use at your own risk.

**Credits**

The DKMS module by Strongz is instrumental in making this possible ([i915-sriov-dkms GitHub repository](https://github.com/strongtz/i915-sriov-dkms?ref=michaels-tinkerings)).  
Additionally, Derek Seaman and Michael's blog post was an inspirational resource ([Derek Seaman’s Proxmox vGPU Guide](https://www.derekseaman.com/2023/11/proxmox-ve-8-1-windows-11-vgpu-vt-d-passthrough-with-intel-alder-lake.html) & [vGPU (SR-IOV) with Intel 12th Gen iGPU](https://www.michaelstinkerings.org/gpu-virtualization-with-intel-12th-gen-igpu-uhd-730/)).  
This guide is tailored for Proxmox 8, aiming to streamline the installation process.

**Environment**

The environment used for this guide:

• Model: Intel NUC12 Pro Wall Street Canyon (NUC12WSKi5)

• CPU: Intel 12th Gen i5 1240P (12 Cores, 16 Threads)

• RAM: 64GB DDR4 by Samsung

• Storage: 2TB Samsung 980 Pro NVMe SSD and 4TB Samsung 870 Evo SATA SSD

• Graphics: Intel Iris Xe Graphics (80 Execution Units)

**Prerequisites**

Before proceeding, ensure the following:

• VT-d (IOMMU) and SR-IOV are enabled in BIOS.

• Proxmox Virtual Environment (VE) version 8.1.4 or newer is **installed with GRUB bootloader**.

• EFI is **enabled**, and Secure Boot is **disabled**. (Please refer to **Appendix 2** during the guide if your Secure Boot is Enabled)

• Linux kernel version 6.1 or newer is present. Validate with 
```bash
uname -r
```

If your kernel is older than 6.1, refer to **Appendix 1** for instructions on updating it.

## Proxmox Setup
### DKMS Setup
1. Update your system and install required packages:
```bash
apt update && apt install pve-headers-$(uname -r)
apt update && apt install git pve-headers mokutil
rm -rf /var/lib/dkms/i915-sriov-dkms*
rm -rf /usr/src/i915-sriov-dkms*
rm -rf ~/i915-sriov-dkms
KERNEL=$(uname -r); KERNEL=${KERNEL%-pve}
```
2. Proceed to clone the DKMS repository and adjust its configuration:
```bash
git clone https://github.com/strongtz/i915-sriov-dkms.git
cd ~/i915-sriov-dkms
cp -a ~/i915-sriov-dkms/dkms.conf{,.bak}
sed -i 's/"@_PKGBASE@"/"i915-sriov-dkms"/g' ~/i915-sriov-dkms/dkms.conf
sed -i 's/"@PKGVER@"/"'"$KERNEL"'"/g' ~/i915-sriov-dkms/dkms.conf
sed -i 's/ -j$(nproc)//g' ~/i915-sriov-dkms/dkms.conf
cat ~/i915-sriov-dkms/dkms.conf
```
3. Install the DKMS module:
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
5. 

### GRUB Bootloader Setup
Backup and update the GRUB configuration:
```bash
cp -a /etc/default/grub{,.bak}
sudo sed -i '/^GRUB_CMDLINE_LINUX_DEFAULT/c\GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt i915.enable_guc=3 i915.max_vfs=7"' /etc/default/grub
update-grub
update-initramfs -u -k all
apt install sysfsutils -y
```

### Create vGPU
Identify the PCIe bus number of the VGA card, usually **00:02.0**: 
```bash
lspci | grep VGA
```

Edit the sysfs configuration to enable the vGPUs. In this case I’m using **00:02.0**. To verify the file was modified, cat the file and ensure it was modified.
```bash
echo "devices/pci0000:00/0000:00:02.0/sriov_numvfs = 7" > /etc/sysfs.conf
cat /etc/sysfs.conf
```

### Secure Boot Configuration (Optional)
If you are using Secure Boot, please follow the **Appedix 2** before next step

### Completing  vGPU setup
Now reboot your Host
```bash
reboot
```
And you should able to see the minor **PCIe IDs 1-7** and finally **Enabled 7 VFs**
```bash
lspci | grep VGA
dmesg | grep i915
```
Now your host is prepared, and you can set up Windows 10/11 VMs with SR-IOV vGPU support.

## Windows Installation
### Download Windows Images and Virtos Driver
1. Download the latest VirtIO Windows driver ISO from [here](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso). 
2. Download the Windows 11 ISO from [here](https://www.microsoft.com/software-download/windows11). Use the **Download Windows 11 Disk Image (ISO) for x64 devices** option.
3. Upload both .iso image to your Proxmox storage, I use local -> ISO Images here
4. Start the VM creation process. On the **General** tab enter the name of your VM. Click **Next**.\
5. On the **OS** tab select the Windows 11 ISO.  Change the Guest OS to **Microsoft Windows, 11/2022**. Tick the box for the VirtIO drivers, then select your Windows VirtIO ISO. Click **Next**. **Note:** The VirtIO drivers option is new to Proxmox 8.1. I added a Proxmox 8.0 step at the end to manually add a new CD drive and mount the VirtIO ISO.
6. On the **System** page modify the settings to match EXACTLY as those shown below. If your local VM storage is named differently (e.g. NOT **local-lvm**, use that instead).
7. On the **Disks** tab, modify the size as needed. I suggest a minimum of 64GB. Modify the **Cache** and **Discard** settings as shown. Only enable **Discard** if using SSD/NVMe storage (not a spinning disk).
8. On the **CPU** tab, change the **Type** to **host**. Allocate however many cores you want. I chose 2.
9.  On the **Memory **tab allocated as much memory as you want. I suggest 8GB or more. \
10.  On the **Network** tab change the model to **VirtIO**.
11.  Review your VM configuration. Click **Finish**. **Note:** If you are on Proxmox 8.0, modify the hardware configuration again and add a CD/DVD drive and select the VirtIO ISO image. Do not start the VM. 

### Windows 11 Installation
1.  In Proxmox click on the Windows 11 VM, then open a console. Start the VM, then press **Enter** to boot from the CD.
2.  Select your language, time, currency, and keyboard. Click **Next**. Click **Install now**.
3.  Click **I don’t have a product key**. 
4.  Select **Windows 11 Pro**. Click **Next**.
5.  Tick the box to accept the license agreement. Click **Next**.
6.  Click on **Custom** install.
7.  Click **Load driver**.
8.  Click **OK**.\
9.  Select the **w11** driver. Click **Next**.
10.  **On Where do you want to install Windows** click **Next**.\
11.  Sit back and wait for Windows 11 to install.

### Windows 11 Initial Configuration

**Note:** I strongly suggest using a Windows local account during setup, and not your Microsoft cloud account. This will make remote desktop setup easier, as you can’t RDP to Windows 11 using your Microsoft cloud account. The procedure below “tricks” Windows into allowing you to create a local account by attempting to use a locked out cloud account. Also, do NOT use the same username for the local account as your Microsoft cloud account. This might cause complications if you later add your Microsoft cloud account.

1.  Once Windows boots you should see a screen confirming your country or region. Make an appropriate selection and click **Yes**.
2.  Confirm the right keyboard layout. Click **Yes**. Add a second keyboard layout if needed. 
3.  Wait for Windows to check for updates. Windows may reboot. 
4.  Enter the name of your PC. Click **Next**. Wait for Windows to reboot.
5.  Click **Set up for personal use**. Click **Next**. Click **Sign in**.
6.  To bypass using your Microsoft cloud account, enter **user @ outlook .com **(no spaces), enter a random password, click **Next** on **Oops, something went wrong**. 
7.  On the** Who’s going to use this device?** screen enter a username. Click **Next**.
8.  Enter a password. Click **Next**.
9.  Select your security questions and enter answers.
10. Select the Privacy settings you desire and click **Accept**.
11. In Windows open the mounted ISO in Explorer. Run **virtio-win-gt-x64** and **virtio-win-guest-tools**. Use all default options.
12. You will probably also want to change the Windows power plan so that the VM doesn’t hibernate (unless you want it to).
13. You may want to disable local account password expiration, as RDP will fail when your password expires with no way to reset. You’d need to re-enable the Proxmox console to reset your password (see later in this post for a how to).
```
wmic UserAccount set PasswordExpires=False
```

## Windows 11 vGPU Configuration
1. Open a Proxmox console to the VM and login to Windows 11. In the search bar type **remote desktop**, then click on remote** desktop settings**.
2. Enable **Remote Desktop**. Click **Confirm**.
3. Open your favorite RDP client and login using the user name and credentials you setup. You should now see your Windows desktop and the Proxmox console window should show the lock screen.\
4. Inside the Windows VM open your favorite browser and download the latest Intel “Recommended” graphics driver from [here](https://www.intel.com/content/www/us/en/download/726609/intel-arc-iris-xe-graphics-whql-windows.html). In my case I’m grabbing **31.0.101.4972**.\
5. Shutdown the Windows VM. \
6. You can now unmount the Windows 11 and VirtIO ISOs.
7. In the Proxmox console click on the Windows 11 VM in the left pane. Then click on **Hardware**. Click on the **Display** item in the right pane. Click **Edit**, then change it to **none**.
**Note:** If in the next couple of steps the 7 GPU VFs aren’t listed, try rebooting your Proxmox host and see if they come back. Then try adding one to your Windows VM again.
8. In the top of the right pane click on **Add**, then select **PCI Device**.\
9. Select **Raw Device**. Then review all of the PCI devices available. Select one of the sub-function (.1, .2, etc..) graphics controllers (i.e. ANY entry except the 00:02.0). Do **NOT** use the root “0” device, for ANYTHING. I chose **02.1**. Click** Add**. **Do NOT** tick the “All Functions” box. Tick the box next to **Primary GPU**. Click **Add**.
10. Start the Windows 11 VM and wait a couple of minutes for it to boot and RDP to become active. Note, the Proxmox Windows console will NOT connect since we removed the virtual VGA device. You will see a **Failed to connect to server** message. You can now ONLY access Windows via RDP. \
11. RDP into the Windows 11 VM. Locate the Intel Graphics driver installer and run it. If all goes well, you will be presented with an **Installation complete!** screen. Reboot. If you run into issues with the Intel installer, skip down to my troubleshooting section below to see if any of those tips help. 

## Windows 11 vGPU Validation
1. RDP into Windows and launch **Device Manager**. \
2. Expand **Display adapters** and verify there’s an Intel adapter in a healthy state (e.g. no error 43).
3. Launch **Intel Arc Control**. Click on the **gear icon**, **System Info**, **Hardware**. Verify it shows **Intel Iris Xe**.
4. Launch **Task Manager**, then watch a YouTube video. Verify the GPU is being used.

## Appedixies
### Appedix 1 - Kernal lower than 6.1
You can update the PVE kernel to 6.2 5.19 using these commands:
1.  Disable enterprise repo.\
```bash
vi /etc/apt/sources.list
```
Then use the following repos from Proxmox:  
(https://pve.proxmox.com/wiki/Package_Repositories)  
My sources.list looks like this:  
```bash
deb http://ftp.uk.debian.org/debian bookworm main contrib
deb http://ftp.uk.debian.org/debian bookworm-updates main contrib
deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
# security updates
deb http://security.debian.org bookworm-security main contrib
```
2. Install 6.1 kernel (or 6.2).\
```bas
apt install pve-kernel-6.1
```
3.  Update apt and install 6.1 headers.\
```bash
apt update && apt install pve-headers-$(uname -r)
```
4. Update initramfs and reboot.\
```bash
update-initramfs -u\
reboot
```
5. Check your kernel
```bash
uname -r
```
   
**Back to the Guide where you left**

### Appedix 2 - EFI  **Enabled** and Secure Boot **Enabled**
You need to install mokutill for Secure Boot
```bash
apt update && apt install mokutil
mokutil --import /var/lib/dkms/mok.pub
```
**Reboot Proxmox Host**, monitor the boot process and wait for the **Perform MOK management** window (screenshot below). If you miss the first reboot you will need to re-run the mokutil command and reboot again. The DKMS module will NOT load until you step through this setup. 

Secure Boot MOK Configuration (Proxmox 8.1+)

Select **Enroll MOK, Continue, Yes, *\<password>*, Reboot. **

**Back to the Guide where you left**

## Conclusion
By completing this guide, you should be able to share your Intel Gen 12 iGPU across multiple VMs for enhanced video processing and virtualized graphical tasks within a Proxmox environment.
