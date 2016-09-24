# Installing a guest VM with PCI Passthrough
To start this article, I'll express why I would go through this rather complicated procedure for installing, configuring and running a VM with direct access to the PCI bus. I **hate** windows. Back in XP days which I actually remember fairly fondly, I would say Microsoft was at the top of their game, and had a brief bounce back during with 7. But now with Windows 10 being an unreliable bastard that gives me more headaches than even some of the more difficult Linux distributions, I really have to ask myself, why do always need to keep at least one Windows machine around? One word, games. Coming home from work, sometimes I just want to zone out and play something, and as many great strides Linux has made in terms of courting more game studios, including Valve's massive contributions to Linux games, there's still just too many that are Windows exclusives that I need to keep a windows gaming system around. So for a long time I've been dual booting Arch Linux with Windows, and although it's not a bad solution, you just can't beat the convenience of running a VM with near-native performance. This also gives me the infrastructure to run other OS's to experiment with things, and having a *always* on OS to run background tasks like my own cloud server, Plex server, etc. So if I get the urge to play a game, or need proprietary content-creation software like Ableton, Creative-Suite, orEngineering software like Fusion 360 or FPGA software, all I need to do now is run my script to launch the Windows VM, flick the switch on my DIY hardware KVM (*build-log coming soon*) and presto, with my fastest SSD storing the VM, I'm looking at the Windows login screen in under 20 seconds. And because every file is hosted on the OS hosting the VM, all my files are synced and available through the virtual network port.

### The Software Stack 

- KVM
	- A [hypervisor][3] which provides the abstraction layer between your physical hardware and the virtualization layer, making it seem for the guest OS that it is in fact running on its own hardware, when in fact it's running on a provisioning of the actual hardware on top of the hypervisor. Instead of providing firmware support for the specific hardware, the hypervisor must support the hardware it is installed on, and provides virtual hardware for the Virtual Machine. So in short, if the hypervisor supports the hardware, and if the hypervisor has virtual drivers and firmware for the guest OS, everything should work.
- QEMU
	- If the hypervisor provides the abstraction layer between hardware and virtual machine, QEMU is there to provision hardware resources between the host OS and the guest. It will also emulate the hardware components so it looks like normal hardware to the guest OS
- VirtIO
	- A virtualization standard for computer compenents connected to the IO bus (think network interfaces, storage controllers, etc.). This also recompiles the guest OS so that it works as cleanly in a Virtual Machine host as possible
- VirtFS
	- The file system used for creating virtual disk files, that emulate the existense of an individual disk, when in fact it is a file inside a physical disk. It's specifically made to handle the complications of having a host OS using a physical disk, while also allowing full emulation of the functions a stand alone disk would have while only being a virtual filesystem. If you're familiar with Docker volumes, VirtFS is equivalent to them
- VFIO
	- This is where the magic happens, we've all likely used a virtual machine before and accepted the fact that graphics performance would be decidedly weak. But with the advent of CPU's and motherboards that support passing the PCI bus directly to the virtualization layer, it is now possible to connect to whatever device is on the PCI bus (including graphics cards) and use them at near-native speeds, assuming the PCI bus isn't being saturated by other virtualized guest OS's or the host OS. VFIO allows the host OS to provision what virtual machines get access to what hardware on the bus, and performas all the memory maps necessary to make the guest OS able to use these devices.
- Libvirt
	- A collection of software that provides a convenient way to manage virtual machines and virtualization functionality like storage, network, audio mixing, etc.

### Install & Setup

Let's leave any more editorial comments or conclusions after actually setting up the system and testing it. Most credit in terms of researching how to do this goes to [1][hkemler] on the *pcmasterrace* subreddit for posting a terrific guide, and as always the loaded, yet not always straight-forward [2][Arch Wiki]. First, on the host OS, download or acquire an `.iso` file for the Windows install. If you go to their site there should be a download and install tool that will allow you to either re-install Windows on a current windows system, format it onto a UBS drive, or simply download the iso. This is my preffered method to do it, and when I first did, it was on a bootcamp partition on my macbook. Whatever way you acquire it, just get the iso onto the host OS somehow.

Next, the system needs to have [5][IOMMU] enabled. IOMMU, as the link there shows, is the physical circuitry within compatible motherboards that physically maps memory (usually using DMA) to IO devices such as graphics cards such that they can be accessed directly by the [3][hypervisor]. Your motherboard BIOS will need to be modified to enable the IOMMU feature, and depending on the motherboard and CPU you have, these settings could be called different things, such as VT-d (Intel), or AMD-Vi (AMD), or the more general Virtual. Just about any setting in BIOS that has the word virtual in it conceivably be enabled. If you don't know how to do it for your own motherboard, just google it. If your motherboard and CPU supports it, there will more than likely be something out there on the web describing how to activate it.

With the hardware enabling [5][IOMMU] it's time to start configuring low level software to use it. The bootloader that loads your host OS on booting, needs to have kernel options set that loads the OS with this feature enabled. If you check the [2][Arch Wiki] you'll see a link to how to set kernel options for your specific device and bootloader. I'll be showing how to do this with a GRUB bootloader installed. To do this, *for intel virtualization*, add `intel_iommu=on` to your grub defaults file located in `/etc/defaults/grub` like this:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on"
```
 Any kernal options you want are seperated by spaces within the quotation marks of the `GRUB_CMDLINE_LINUX_DEFAULT` line like above. Now with new default options, the bootloader needs to be updated by using the `grub-mkconfig` program. If you have another type of bootloader, again, find out how to properly add the IOMMU option to it during bootup and how to update it. Afterwards, you will need to reboot for the changes to take effect. For me, the command line below is what I needed to use:
```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```
Ok, hopefully now the host OS is configured to make use of IOMMU, but let's be sure by entering this line into the terminal:
```
lspci -nnk
```
Which should return several items, but most importantly some lines about the graphics card that will be passed through like the line below. It will be different for you based on the particular graphcis card you have installed. I have an NVIDIA GTX 1070 installed, so I pipe a `grep` statement into the line above for `NVIDIA` and below I see that it is listed when polling the PCI bus for devices with an output like this:
```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP104 [GeForce GTX 1070] [10de:1b81] (rev a1)
01:00.1 Audio device [0403]: NVIDIA Corporation GP104 High Definition Audio Controller [10de:10f0] (rev a1)
```
Now that we know the hardware is detectable, recall the PCI grouping number in the first part of the output above. `01:00.0` is the enumerated reference for the OS to access the graphics, *known as VGA controller*, card. The one below it is likely the enumeration of its internal sound card that passes audio through the card's HDMI port. With that number in mind, and it is quite possible that yours will be different, run this command below, and check the output for something similar to see if IOMMU works and is able to memory map the graphics card by seeing if a reference to the PCI number comes up. Remember to use the correct PCI grouping number in the terminal, for me this comes up in the `\:01` part of the command line entry.
```
ls -lha /sys/bus/pci/devices/0000\:01\:00.0/iommu_group/devices
total 0
drwxr-xr-x 2 root root 0 Sep 24 10:52 .
drwxr-xr-x 3 root root 0 Sep 24 10:52 ..
lrwxrwxrwx 1 root root 0 Sep 24 10:52 0000:00:01.0 -> ../../../../devices/pci0000:00/0000:00:01.0
lrwxrwxrwx 1 root root 0 Sep 24 10:52 0000:01:00.0 -> ../../../../devices/pci0000:00/0000:00:01.0/0000:01:00.0
lrwxrwxrwx 1 root root 0 Sep 24 10:52 0000:01:00.1 -> ../../../../devices/pci0000:00/0000:00:01.0/0000:01:00.1
```
Great! Now it's been verified that the host OS is correctly mapping the PCI slot with the graphics card to memory using the IOMMU functionality just enabled. It is very important to get this step right, because without IOMMU configured to work properly this whole guide is kind of pointless because it won't be possible to pass the graphics card through to the virtual machine. If for any reason this was setup properly, one of three things has happened, either the bootloader for the host OS doesn't have kernel options enabled during the bootloading process, in which case check this guide again to configure GRUB properly to do so or whatever guide you've found to enable kernel options for your particular bootloader. Two, it is possible that IOMMU wasn't enabled on the hardware level because the BIOS options that enable this hardware wasn't activated. Three, it is possilbe that the hardware of the host OS, namely the motherboard or CPU, doesn't support IOMMU in which case I'm afraid you're SOL, you'll need to acquire both a CPU and motherboard that supports it.

### Tweaking IOMMU Groups

With IOMMU enabled and properly mapping PCI devices, it's time to install the `linux-vfio` package, which is the name for the Arch AUR package, if you're on another distribution of linux install it with their associated package manager. The vfio program will handle how PCI devices get provisioned to guest OS installed as virtual machines.


### References
[1](https://www.reddit.com/r/pcmasterrace/comments/3lno0t/gpu_passthrough_revisited_an_updated_guide_on_how/)r/pcmasterrace post on installing Windows 10 with QEMU/KVM on Arch
[2](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF)Arch Wiki: PCI Passthrough via OVMF
[3](https://en.wikipedia.org/wiki/Hypervisor) Wikipedia article on Hypervisors
[4](http://lime-technology.com/wiki/index.php/UnRAID_6/VM_Management)Lime Technology Docs: VM_Management 
[5](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit)Wikipedia Article on IOMMU
[6](https://wiki.archlinux.org/index.php/Kernel_parameters#GRUB)Arch Wiki: Kernel Parameters
