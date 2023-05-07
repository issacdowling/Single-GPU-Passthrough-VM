# 1gpupassvm (Fedora)

## Prerequisites
* I own an AMD GPU, and so NVIDIA users will need to look elsewhere
* I am not an expert, if this causes you issues or is in some way unideal, that's your responsibility to deal with
* Ensure above 4g decoding is disabled in the bios. Right now, it causes VM issues.
* Enable your respective virtualisation technology, for AMD, that's CSM in the bios.
* Whenever I say "run", copy whatever's in that box into the terminal, and press enter.
* Download what you'll need later, [VirtIO Drivers](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso) and [Windows ISO](https://www.microsoft.com/en-gb/software-download/windows11/)

## Update and install things
Run these to update and install what's necessary, and add yourself to the libvirt group.
```
sudo dnf upgrade -y --refresh
sudo dnf -y group install Virtualization
sudo usermod -aG libvirt,input,kvm $USER
sudo systemctl restart libvirtd
```
## Change bootloader options
Run this
```
sudo nano /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```
Then - if you have an AMD CPU - add this: (replace amd with intel if using an Intel CPU)
```
amd_iommu=on iommu=pt
```
To the line containing "GRUB_CMDLINE_LINUX="

CTRL+X, Y, Enter, to save and exit, and it'll regenerate your grub2 config.

## VM Setup
Open Virtual Machine Manager, which should've been installed earlier,
* Click edit at the top, followed by preferences
* Enable XML editing
Then, click QEMU/KVM, and then the New VM button in the top left. Here are the buttons to press:
* Local Install Media
* Forward
* Browse
* Browse Local
* Downloads
* Select the ISO file
* Forward
* 80% of your ram, leave CPU settings alone for later
* Forward
* Untick "Enable Storage For This Machine", we'll handle it later.
* Forward
* Set a name. Examples shown will use "Windows". **Change "Windows" to whatever you call your VM in the later scripts if you choose something different.**
* Tick customise config before install
* Finish
* Click BIOS, and change it to the /x64/OVMF_CODE.secboot option (secboot enables secureboot. If you know you don't want it, you can disable it, but it's necessary for windows 11)
* Apply
* Go to CPUs
* Set configuration to host-model
* Click topology, and manually set topology.
* Sockets to 1, set cores to -1 from whatever your CPU has, so a 6 core processor would have 5.
* Go to a terminal and type htop. At the top you'll see a bunch of charts going horizontally at the top, which are numbered. On a Ryzen 3600, they go from 0-11, meaning there are 12 threads, or 2x the core count, meaning it has hyperthreading. If you have double as many bars as cores, set threads on your VM to 2. If it's the same as the physical cores you have, set threads to 1.
* Click add hardware in the bottom left of the virtual machine manager, and select storage. Set the Bus Type to VirtIO instead of SATA, and pick however much storage you want (minimum 64GB for Windows 11).
* Add hardware to your VM again, select storage, and set the device type to CDROM Device. Click "select or create custom storage", then browse to the VirtIO iso. Press finish
* Now go to overview, and click the XML tab.
* Look for the line beginning with "spinlocks State"
* Paste this:
````
<vendor_id state="on" value="fedoralinux"/>
      <vpindex state="on"/>
      <runtime state="on"/>
      <synic state="on"/>
      <stimer state="on"/>
      <reset state="on"/>
      <frequencies state='on'/>
````
Then, just below the end hyperv line, paste:
````
    <kvm>
      <hidden state="on"/>
    </kvm>
````    

* Now, go to the top, and press begin installation.
* If you see a permissions error, run
```
sudo ausearch -c 'qemu-system-x86' --raw | audit2allow -M my-qemusystemx86
sudo semodule -i my-qemusystemx86.pp
```
which will tell SElinux to allow whatever it's blocking.
* When you see a menu which says press any key to boot from CD, press any key. If you miss the time window, close the VM, right click it and force off, then try again.

**Keep in mind, during first install, graphics and framerate will be pretty bad**

* Go through installation as normal for the first bit.
* Now, at the install section, press load drivers, then ok, and select the driver that appears with w11 (or whatever other version of windows you're using) in the name. Click it and press next.
* Now click on Drive 0 unallocated space, and press next. Windows will install. 
* When windows restarts, you may be put into windows, but you may be put into a terminal. If in the terminal,click the dropdown near the power icon at the top of the VM, and force it off.
* Press the info icon, and press boot options, and tick VFIO disk 1, unticking everything else. Then, go to the CDROM drive with the Windows ISO on, and remove + delete it.
* Now press play, and go back to the view icon next to the info icon. You'll go through the windows personalisation setup now.
* When you're on the windows desktop, shut down windows. You can X out of it in Linux afterwards.

## Hooks
Run 
```
sudo mkdir /etc/libvirt/hooks
sudo mkdir -p /etc/libvirt/hooks/qemu.d/Windows/prepare/begin
sudo mkdir -p /etc/libvirt/hooks/qemu.d/Windows/release/end
sudo nano /etc/libvirt/hooks/qemu
sudo chmod +x /etc/libvirt/hooks/qemu
sudo systemctl restart libvirtd
sudo nano /etc/libvirt/hooks/qemu.d/Windows/prepare/begin/start.sh
sudo nano /etc/libvirt/hooks/qemu.d/Windows/release/end/revert.sh
sudo chmod +x /etc/libvirt/hooks/qemu.d/Windows/prepare/begin/start.sh
sudo chmod +x /etc/libvirt/hooks/qemu.d/Windows/release/end/revert.sh
```
This will make directories, enter the text editor, and restart libvirtd

#### Hook file
Paste into the text editor:

```
#!/usr/bin/env bash
#
# Author: SharkWipf
#
# Copy this file to /etc/libvirt/hooks, make sure it's called "qemu".
# After this file is installed, restart libvirt.
# From now on, you can easily add per-guest qemu hooks.
# Add your hooks in /etc/libvirt/hooks/qemu.d/vm_name/hook_name/state_name.
# For a list of available hooks, please refer to https://www.libvirt.org/hooks.html

GUEST_NAME="$1"
HOOK_NAME="$2"
STATE_NAME="$3"
MISC="${@:4}"

BASEDIR="$(dirname $0)"

HOOKPATH="$BASEDIR/qemu.d/$GUEST_NAME/$HOOK_NAME/$STATE_NAME"

set -e # If a script exits with an error, we should as well.

# check if it's a non-empty executable file
if [ -f "$HOOKPATH" ] && [ -s "$HOOKPATH" ] && [ -x "$HOOKPATH" ]; then
    eval \"$HOOKPATH\" "$@"
elif [ -d "$HOOKPATH" ]; then
    while read file; do
        # check for null string
        if [ ! -z "$file" ]; then
          eval \"$file\" "$@"
        fi
    done <<< "$(find -L "$HOOKPATH" -maxdepth 1 -type f -executable -print;)"
fi
```

#### Start file
Next, paste
```
#Debugging (show currently running command)
set -x

#Stop disp manager
systemctl stop display-manager.service
#If not on Gnome, comment these, and pick X or Wayland
killall gdm-wayland-session
#killall gdm-x-session

#unbind VTconsoles
echo 0 > /sys/class/vtconsole/vtcon0/bind
echo 0 > /sys/class/vtconsole/vtcon1/bind

#Unbind efifb
#echo "efi-framebuffer.0" > /sys/bus/platform/devices/efi-framebuffer.0/driver/unbind

#load vfio
modprobe vfio-pci
```
Copy This into the text editor, and we'll edit to fit your PC.

For the VTconsoles bit, go into another terminal and run ``ls /sys/class/vtconsole/``. For however many vtcons there are, copy the lines that are already in this code, changing the number to match. Most people just have vtcon0 and 1, which is already setup.

Uncommend the unbind efifb line if not using a 6000 series+ card.

Then, leave the rest of the script alone, and **CTRL+X, Y, ENTER**

#### Revert File

Now, in the text editor, paste this:
```
#unload vfio-pci
modprobe -r vfio-pci

#Rebind VTconsoles
echo 1 > /sys/class/vtconsole/vtcon0/bind
echo 1 > /sys/class/vtconsole/vtcon1/bind

#Rebind efifb
#echo "efi-framebuffer.0" > /sys/bus/platform/drivers/efi-framebuffer/bind

#Restart Display Service
systemctl start display-manager.service
```
For VTconsoles, make the same changes, if any, you made in the start script.
Uncomment the rebind efifb line if not using a 6000 series+ card.

**Now, CTRL+X, Y, ENTER**

# We're nearly there! Give yourself a pat on the back

Open virtual machine manager, and click on your machine. Go to the info icon in the top left, and follow these steps:
* Remove Tablet
* Remove Display Spice (THIS MAY NOT WORK. IF NOT, YOU JUST HAVE TO REMOVE IT AND ANYTHING RELATED FROM THE XML IN OVERVIEW, INCLUDING USB SPICE REDIRECT)
* Remove Serial 1
* Remove Channel spice
* Remove Sound ich9
* Remove Video
* Remove anything else unnecessary if you'd like.
* Add Hardware
* USB Host device
* Find your keyboard here, select it and finish
* Repeat for your mouse
* You can try adding other things, but try them after we have a working VM, since some devices don't like it and cause a crash.
* Add Hardware
* PCI Host device
* Your GPU Video adapter
* Repeat for GPU audio adapter.

## Restart your PC, and give your VM a go!
It will likely not work, because of SELinux. 

I am not an expert, these changes have not been verified by me or anybody else not to hurt the security or reliability of your system. These are just the commands I needed to run to ensure SELINUX would allow my VM to work fully.
```
sudo ausearch -c 'qemu-system-x86' --raw | audit2allow -M my-qemusystemx86
sudo semodule -i my-qemusystemx86.pp
```
It may now work. If it doesn't, run it again. Keep going until it does (2 tries for me got it working).
## If it works, here are some extras.

### I want to pass through devices in the same IOMMU group (this is the section for people with wonky IOMMU groups from earlier)
#### This is also relevant for people who want to pass through their audio/network devices to the VM
Switch your kernel to Xanmod. Or do some other method to get the ACS patch, but I don't know how.
For Xanmod, just run
```
sudo dnf copr enable rmnscnce/kernel-xanmod -y
sudo dnf in kernel-xanmod-edge
```
Then reboot, and type
```
hostnamectl
```
to verify that you're now using Xanmod.

Then, go to your bootloader options, on grub that's 
```
sudo nano /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```
and on the line **GRUB_CMDLINE_LINUX_DEFAULT=**, *inside* the quotation marks, add:
```
pcie_acs_override=downstream,multifunction
```
CTRL+X, Y, ENTER.

Not only should this fix any GPU-related IOMMU group issues, but it should also allow you to directly passthrough things like your motherboard's NIC, or a PCIE USB card that's not cooperating. Anything PCIE should be free to pass now. Also note: passing a USB controller is MUCH PREFERRED to individually forwarding USB devices. It allows hotplug, and is more compatible

### Performance
If on ryzen, add this to your CPU section in the XML, something something performance / hyperthreading
````
<feature policy='require' name='topoext'/>
````

### Anticheat
If you want this VM to be compatible with the most annoying Anti-VM games, go to virt-manager, then the CPU section. Untick "copy host configuration", then set the model to "host-passthrough". Then, boot the VM, and just go to "Turn windows features on or off", and enable Hyper-V. There's a good reason not to have this enabled if you don't absolutely need it, since CPU performance takes a large hit, however this will allow you to bypass *most but not all* VM detection
