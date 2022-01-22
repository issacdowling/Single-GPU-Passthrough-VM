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
