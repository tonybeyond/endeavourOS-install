# My EndeavourOS Installation Guide

After numerous attempts at installing and re-installing various Linux distributions, even reverting to Windows on my x86 computer, I've finally settled on an ideal setup—at least for now.

## My Rig Configuration

- **Motherboard:** Gigabyte B550 AORUS ELITE AX V2
- **Processor:** AMD Ryzen 9 5950X
- **RAM:** 64GB DDR4
- **Storage:** 2x NVMe (system & data)
- **Primary GPU:** AMD ATI Radeon Pro WX 5100 (8GB VRAM)
- **Passed-through GPU:** NVIDIA GeForce RTX 3060 (12GB VRAM)

## Installation Process

I started by installing EndeavourOS with its shipped i3 window manager (WM) configuration, which provided a solid base. In the past, I've experimented with QTILE, AWESOME, and others, including Gnome and Pop_Shell. However, EndeavourOS's i3 WM configuration has been the most satisfying, offering no screen tearing and other benefits common to tiling window managers.

You can find EndeavourOS [here](https://endeavouros.com/). It's an Arch-based Linux distro without the setup hassle. The default configurations are a good starting point, requiring minimal tinkering to achieve a functional and aesthetically pleasing environment. Additionally, it's easy on the fingers.

You can select your preferred window manager during installation using Calamares.

# Post-Installation Setup

Once the OS installation is complete, add the following packages:

```bash
yay -S neovim ttf-meslo-nerd stow fzf fd zsh virt-manager virt-viewer qemu vde2 iptables-nft nftables dnsmasq bridge-utils edk2-ovmf swtpm mkinitcpio freerdp btop parsec-bin obsidian pcloud-drive looking-glass nextcloud-client hyfetch
```

make neovim a bit better using nvim-lua kickstart
```bash
git clone https://github.com/nvim-lua/kickstart.nvim.git "${XDG_CONFIG_HOME:-$HOME/.config}"/nvim
```

I'm using [stow](https://www.gnu.org/software/stow/manual/stow.html) to create symlinks to the right places of the dotfiles. You don't have to but it's just easier to keep things in the right folder to upload to git. You can see this [Typecraft video](https://www.youtube.com/watch?v=wXZgUudR41I&t=834s) to get a first glimpse of what it can do.

# GPU Passthru walkthrough
## check if IOMMU groups are working 
```bash
sudo dmesg | grep -i -e DMAR -e IOMMU
```

should spit out this kind of stuff
```bash
[    0.641728] pci 0000:00:00.2: AMD-Vi: IOMMU performance counters supported
[    0.641762] pci 0000:00:01.0: Adding to iommu group 0
[    0.641774] pci 0000:00:01.1: Adding to iommu group 1
[    0.641787] pci 0000:00:01.2: Adding to iommu group 2
[    0.641805] pci 0000:00:02.0: Adding to iommu group 3
[    0.641822] pci 0000:00:03.0: Adding to iommu group 4
[    0.641834] pci 0000:00:03.1: Adding to iommu group 5
```
You can also check the number of CPU threads you have available to check that SVM (amd) or VMX (intel) are properly enabled in the bios
```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```
For my AMD CPU, it shows `32` which is due to the amount of physical cores `16` for the 5950X amd the fact hyperthreading is enabled which makes it a theoretical `2 x 16` threads.


## edit grub
The line should look like this (replace with Interl for intel system instead of amd)
```bash
GRUB_CMDLINE_LINUX_DEFAULT='rd.driver.pre=vfio-pci nowatchdog nvme_load=YES loglevel=3 amd_iommu=on iommu=pt video=efifb:off'
```

run this command to update grub
```bash
sudo grub-mkconfig
```

## edit mkinitcpio.conf
```bash
sudo nvim /etc/mkinitcpio.conf
```

The file should look like this (initial file comments removed here for concision)

```bash
MODULES=(vfio_pci vfio vfio_iommu_type1)

BINARIES=(btrfs setfont)

FILES=()

HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block filesystems fsck)
```
Then run to apply the changes
```bash
sudo mkinitcpio -p linux
```

## Bind vfio-pci drivers to the proper Nvidia card
To find the Nvidia PCI IDs of the card run
```bash
lspci -nn | grep -i "NVIDIA"
```
it will give this output. This first ID is the GPU itself and the second the Audio circuit of the GPU (both need to be passed to the VM later)
```bash
08:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA106 [GeForce RTX 3060 Lite Hash Rate] [10de:2504] (rev a1)
08:00.1 Audio device [0403]: NVIDIA Corporation GA106 High Definition Audio Controller [10de:228e] (rev a1)
```

create a vfio file like this
```bash
sudo nvim /etc/modprobe.d/vfio.conf
```
it should contain the PCI IDs found before
```bash
options vfio-pci ids=10de:2504,10de:228e
```
## enable libvirtd service
```bash
systemctl enable libvirtd.service
```

## edit libvirtd config files
You may want to modify a bit how KVM / QEMU runs in terms of service writes

Remove '#' at the following lines:
```bash
unix_sock_group = "libvirt"
unix_sock_rw_perms = "0770"
```
of the following file:
- **`/etc/libvirt/libvirtd.conf`**

Uncomment and add your user name to user and group to this file.
- **`/etc/libvirt/qemu.conf`**
```bash
user = "your username"
group = "your username"
```
 

_reboot the system_

Check that vfio drivers are loaded for your Nvidia card.
```bash
lspci -knn | grep -i "NVIDIA" -A 3
```
It should show in **kernel driver in use** `vfio-pci` for both PCI IDs of the card
```bash
08:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA106 [GeForce RTX 3060 Lite Hash Rate] [10de:2504] (rev a1)
	Subsystem: eVga.com. Corp. Device [3842:3656]
	Kernel driver in use: vfio-pci
	Kernel modules: nouveau
08:00.1 Audio device [0403]: NVIDIA Corporation GA106 High Definition Audio Controller [10de:228e] (rev a1)
	Subsystem: eVga.com. Corp. Device [3842:3656]
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel
```

_Now you should be good to go_

# Change default libvirtd VM image folder
By default VMs images (their actual disk space) are stored here:
`/var/lib/libvirt/images/`
I change usually that to another drive where I store my user data folders and VMs. It can be changed via the virt-manager gui by first removing the `default` folder and recreating it in your desired destination folder by naming it again default. The CLI version is down below.

**Attention**
the actual VM config files (XML) are still in `/etc/libvirt/qemu/`

# Create a Windows VM
After [downloading](https://www.microsoft.com/software-download/windows11) a Windows 11 ISO file. Create a standard Windows VM via virt-manager

For better performances, use VIRT-IO option for the main VM drive and its Network card.
You need to load the drivers to be able to install the system at startup by adding a second CD-ROM which points to the virtio-win ISO file you can get from [here](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso).

Update Windows and enable remote desktop access.

## RDP acess via terminal
Once Windows VM up and running with RDP enabled, we should be able to access it via this command (remmina is an option as well):
```bash
xfreerdp3 -grab-keyboard /v:VM_IP_ADDRESS /u:WINDOWS_USER /p:WINDOWS_USER_PASSWORD /size:100% /d: /dynamic-resolution /cert:ignore`
```

A simple bash script can make this more easy for future use
```bash
#!/bin/bash
xfreerdp3 -grab-keyboard /v:VM_IP_ADDRESS /u:WINDOWS_USER /p:WINDOWS_USER_PASSWORD /size:100% /d: /dynamic-resolution /cert:ignore &
```

## Edit Windows VM XML
First we need to edit a bit the XML file so it disguises to Windows that it's running on a hypervisor (sneaky us)
In the HyperV section add the following (VM turned off)

```xml
    <hyperv mode="custom">
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vendor_id state="on" value="123456789123"/>   <--- this element
    </hyperv>
    <kvm>                                            <--- this section
      <hidden state="on"/>
    </kvm>
```
Start again the VM and check it can still be accessed with the RDP client.


## Passing the NVIDIA card to the VM
with the VM stopped, via the VM Virt-Manager window, go to the Video hardware device and change the model to `None`
Add two more "fake" hardware devices
- virtio keyboard
- virtio tablet

Add the GPU PCI devices (Add Hardware -> PCI Host Device)
- Nvidia GPU PCI (check the names and IOMMU IDs)
- Nvidia Audio PCI

restart the VM and now via the RDP session you should see the Nvidia card in the device list and you can install Nvidia Drivers as a "normal" PC.

## Better display performances (optional)
At this stage, RDP should be more than enough for standard use (web, office, photo editing, LLMs etc.)
If you need/want better remote display performances, you can use looking-glass which offers at better response time and frame-rate compared to RDP.

In the Windows VM, we need to install the looking-glass "host" service.
So still from the Windows VM, go to this site https://looking-glass.io/downloads and download the Windows [host file](https://looking-glass.io/artifact/B6/host). The B6 is the current stable, you can look to use more recent ones if needed from the release candidates.

Once this program installed in Windows restart the VM and launch it via the RDP session to the VM so it will startup each time the VM boots.

Stop the VM. As we need to add some settings in the VM config file (XML) inside the `devices` group
```xml
    <shmem name="looking-glass">
      <model type="ivshmem-plain"/>
      <size unit="M">64</size>
      <address type="pci" domain="0x0000" bus="0x10" slot="0x01" function="0x0"/>
    </shmem>
```
The value of `unit="M"` parameter depends on the desired resolution. Details are [here](https://looking-glass.io/docs/B6/install/).

**Start your VM**

If looking-glass host service is running in the Windows VM (should display an icon in the system tray) from the linux host, you can launch
```bash
looking-glass-client
```
And voilà, you have access to the VM via looking-glass

I've the full XML config file in the repo for reference.

# my main inspiratons and sources for this installation
Typecraft
- [YouTube Channel](https://www.youtube.com/@typecraft_dev)
- [GitHub Repository](https://github.com/typecraft-dev/)

My Linux For Work
- [YouTube Channel](https://www.youtube.com/@mylinuxforwork)
- [GitHub Repository](https://gitlab.com/stephan-raabe)

Thanks to them for their valuable resources and inspiration.













## virsh command lines to change storage pools
### Adding storage pools
As root user
``` bash
virsh pool-list --all
 Name      State    Autostart
-------------------------------
 default   active   yes
```

Define the directory for VM Disks
``` bash
virsh pool-define-as --name "Disk Images" --type dir --target ~/VMs/VMmainDisk/
```

Define the directory for Installation Media
``` bash
virsh pool-define-as --name "Installation Media" --type dir --target ~/VMs/ISOs/
```

Start the pools
``` bash
virsh pool-start --build "Disk Images"
virsh pool-start --build "Installation Media"
```

Check their status
``` bash
virsh pool-list --all
 Name                 State      Autostart
--------------------------------------------
 default              inactive   no
 Disk Images          active     no
 Installation Media   active     no
```

Enable auto-start
``` bash
virsh pool-autostart "Disk Images"
virsh pool-autostart "Installation Media"
```

Check their status again
``` bash
virsh pool-list --all
 Name                 State      Autostart
--------------------------------------------
 default              inactive   no
 Disk Images          active     yes
 Installation Media   active     yes
```

We can more info if needed with below commands
``` bash
virsh pool-info "Disk Images"
Name:           Disk Images
UUID:           1ed11501-52a3-4640-9229-498538d31c92
State:          running
Persistent:     yes
Autostart:      yes
Capacity:       1.83 TiB
Allocation:     32.00 KiB
Available:      1.83 TiB

virsh pool-info "Installation Media"
Name:           Installation Media
UUID:           e35f5d1d-bf78-4b39-8e6b-954a6bf443a8
State:          running
Persistent:     yes
Autostart:      yes
Capacity:       3.58 TiB
Allocation:     32.00 KiB
Available:      3.58 TiB
```

