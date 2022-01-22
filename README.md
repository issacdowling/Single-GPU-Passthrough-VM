# 1gpupassvm

## Prerequesites
**SOME LATER SCRIPTS REFER TO SDDM BEING YOUR DISPLAY MANAGER, THESE ARE IN THE "HOOKS" SECTION. IF YOU DON'T USE SDDM, REMEMBER TO CHANGE IT! SAME GOES FOR NANO**

## Update things
```
sudo pacman -Syu
```
or
```
sudo apt update && sudo apt upgrade
```

## Download what you'll need later
[VirtIO Drivers](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso)

[Windows ISO](https://www.microsoft.com/en-gb/software-download/windows11/)
## Change bootloader options
### If using grub:
```
sudo nano /etc/default/grub
```
Then, on the line **GRUB_CMDLINE_LINUX_DEFAULT=**, *inside* the quotation marks, add these if you've got an AMD cpu:
```
iommu=pt amd_iommu=on video=efifb:off
```
Of course, on intel it's:
```
iommu=pt intel_iommu=on video=efifb:off
```

CTRL+X, Y, ENTER, to save and exit.

Now, run
```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```
### If not using grub:
Look up changing kernel parameters for your bootloader. It'll be vaguely similar

#### Now reboot

## Checking IOMMU Groups
Run
```
shopt -s nullglob
for g in `find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V`; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```
### If your VGA and AUDIO deivces are in the same IOMMU group, with nothing else in it, good.
### If your VGA and AUDIO devices aren't in the same IOMMU group, but they're each in their own group, with nothing else in either, good.
### If your VGA and AUDIO devices are in the same group as other stuff, not good... but I can help! *Head to the end of this guide, it's for you.*

Note the characters (e.g. 2b:00.0) preceeding your "VGA compatible controller" (gpu), and its corresponding audio device.

## Installing stuff
Run this. It downloads necessary packages, and enables services. It will likely ask you for your password multiple times, however this is why it's important that you can see exactly the commands we're running.
```
sudo pacman -S qemu libvirt edk2-ovmf virt-manager ebtables dnsmasq swtpm
systemctl enable libvirtd.service
systemctl start libvirtd
systemctl enable virtlogd.socket
systemctl start virtlogd.socket
sudo virsh net-autostart default
sudo virsh net-start default
```

IF DOING THIS ON DEBIAN, https://salsa.debian.org/nchevsky/libtpms/-/releases/debian%252F0.8.4-1.0, and https://salsa.debian.org/nchevsky/swtpm/-/releases/debian%252F0.6.0-1.0 is where you can find swtpm. At time of writing, it wasn't in normal repos.

## Editing files
Run
```
sudo nano /etc/libvirt/libvirtd.conf
```
Now, CTRL+W, and search unix_sock.
Remove # from that line, and replace root with libvirt
CTRL+W again, search for unix_sock_rw.
Remove # again.

CTRL+X, Y, ENTER, to save and exit.

Now, run
```
sudo nano /etc/libvirt/qemu.conf
```
CTRL+W to search for: user = "r
Remove the # from the line, and change root within the quotes, to your username, and remove the # from the line a little below named: group = "root", swap root for libvirt
**CTRL+X, Y, Enter**.

Finally, 
```
sudo usermod -a -G libvirt $(whoami)
sudo usermod -a -G kvm $(whoami)
sudo systemctl start libvirtd
sudo systemctl enable libvirtd
```
to add yourself to groups, and enable libvirtd.

#### Now, reboot

## GPU Bios
Some GPUs need patched firmware to work for this. In this example, we won't be patching it, since at the time of writing, I have an AMD Vega 64, which doesn't need patching, however we will be saving a bios anyway to pass through to the VM.
Go to the [Techpowerup Bios Repository](https://www.techpowerup.com/vgabios/), and search for your GPU. Download it's bios. You want to find your exact model. E.G: MSI AIR BOOST VEGA 64, as opposed to just "Vega 64". Once you have your rom, rename it to "gpubios.rom", and ensure it's in your Downloads folder. 

Then, you can just run
```
sudo mkdir /etc/libvirt/gpubios
sudo mv /home/$(whoami)/Downloads/gpubios.rom /etc/libvirt/gpubios
```
to move it into a convenient place.

I'd also suggest running
```
**sudo chmod 755 /etc/libvirt/gpubios/gpubios.rom**
```
to ensure permissions are ok.

## VM Setup
Open Virtual Machine Manager, which should've been installed earlier, click QEMU/KVM, and then the New VM button in the top left. Here are the buttons to press:
* Local Install Media
* Forward
* Browse
* Browse Local
* Downloads
* Double Click The ISO file
* Forward
* 80% of your Ram in the Memory section
* Leave CPU settings alone for later
* Forward
* Untick "Enable Storage For This Machine", we'll handle it later.
* Forward
* Set a name. Examples shown will use "Windows". Change "Windows" to whatever you call your VM if you choose something different.
* Tick customise config before install
* Finish
* Click BIOS, and change it to the /x64/OVMF_CODE.secboot option (secboot enables secureboot. If you know you don't want it, you can disable it, but it's necessary for windows 11)
* Apply
* Go to CPUs
* Click topology, and manually set topology.
* Sockets to 1, set cores to -1 from whatever your CPU has, so a 6 core processor would have 5.
* Go to a terminal and type htop. At the top you'll see a bunch of charts going horizontally at the top, which are numbered. On a Ryzen 3600, they go from 0-11, meaning there are 12 threads, or 2x the core count, meaning it has hyperthreading. If you have double as many bars as cores, set threads on your VM to 2. If it's the same as the physical cores you have, set threads to 1.

* Click add hardware in the bottom left of the virtual machine manager, and select storage, which is normally at the top
* Set however many GB you want it to have (keep in mind, the virtual disk only takes as much space as the VM is actually using, so don't worry about it immediately eating all your storage space on the Host)
* Set the Bus Type to VirtIO instead of SATA
* Add hardware to your VM again, select storage, and set the device type to CDROM Device. Click "select or create custom storage", then browse to the VirtIO iso. Press finish
* Click on SATA CDROM 2, and click browse, then browse local, and head to your downloads, where you'll double click the VirtIO drivers iso. Press apply.
* Click add hardware, this time add a TPM with the default settings
* Now, go to the top, and press begin installation.
* When you see a menu which says press any key to boot from CD, press any key. If you miss the time window, close the VM, right click it and force off, then try again.
Keep in mind, during first install, graphics and framerate will be pretty bad
* Go through installation as normal for the first bit.
* Next, install now, I don't have a product key, any version not ending in N, read and accept, next, custom install.
* Now, press load drivers, then ok, and select the driver that appears with w11 (or whatever other version of windows you're using) in the name. Click it and press next.
* Now click on Drive 0 unallocated space, and press next. Windows will install. 
* When windows restarts, you may be put into windows, but you may be put into a terminal. If in the terminal,click the dropdown near the power icon at the top of the VM, and force it off.
* Press the info icon, and press boot options, and tick VFIO disk 1. Then, go to the 2 CDROM drives, and remove + delete them.
* Now press play, and go back to the view icon next to the info icon. You'll go through the windows personalisation setup now.
* When you're on the windows desktop, shut down windows. You can X out of it in Linux afterwards.
