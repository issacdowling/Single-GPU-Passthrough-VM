# 1gpupassvm (Arch)
## Prerequisites
* I own an AMD GPU, and so NVIDIA users will need to look elsewhere
* I am not an expert, if this causes you issues or is in some way unideal, that's your responsibility to deal with
* Ensure above 4g decoding is disabled in the bios. Right now, it causes VM issues.
* Enable your respective virtualisation technology, for AMD, that's CSM in the bios.
* Be on Arch. This branch is for Arch. [Click here for Fedora.](https://github.com/IssacDowling/1gpupassvm/tree/fedora)
* Whenever I say "run", copy whatever's in that box into the terminal, and press enter.
* Download what you'll need later, [VirtIO Drivers](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso) and [Windows ISO](https://www.microsoft.com/en-gb/software-download/windows11/)


## Update things
```
sudo pacman -Syu
```

## Change bootloader options
### If using grub:
```
sudo nano /etc/default/grub
```
Then, on the line **GRUB_CMDLINE_LINUX_DEFAULT=**, *inside* the quotation marks, add these if you've got an AMD cpu:
```
iommu=pt amd_iommu=on
```
Of course, on intel it's:
```
iommu=pt intel_iommu=on
```

CTRL+X, Y, ENTER, to save and exit.

Now, run
```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### If using systemd-boot
```cd /boot/loader/entries/```
Type "sudo nano", then press TAB, which will select your one entry. If you have multiple, it'll show all of them, select the right one. Then press enter.

Go to the end of the line beginning with **options**, and paste this for AMD:
```
iommu=pt amd_iommu=on
```
This for intel:
```
iommu=pt intel_iommu=on
```
CTRL+X, Y, ENTER, to save and exit.


### If not using the above:
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
First, ensure the multilib repo is enabled.
Run
```
sudo sed -i "/\[multilib\]/,/Include/"'s/^#//' /etc/pacman.conf
sudo pacman -Syu
```

Run this. It downloads necessary packages, and enables services. It will likely ask you for your password multiple times, however this is why it's important that you can see exactly the commands we're running.
```
sudo pacman -S virt-manager qemu vde2 ebtables iptables-nft nftables dnsmasq bridge-utils ovmf swtpm wget
```

## Editing files
Run
```
sudo nano /etc/libvirt/libvirtd.conf

```
Now, CTRL+W, and search unix_sock.
Remove # from that line, CTRL+W again, unix_sock_rw, remove #.

CTRL+X, Y, ENTER, to save and exit.

Finally, 
```
sudo usermod -aG libvirt $USER
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
sudo chmod 755 /etc/libvirt/gpubios/gpubios.rom
```

## VM Setup
Open Virtual Machine Manager, which should've been installed earlier, click QEMU/KVM, and then the New VM button in the top left. Here are the buttons to press:
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
* Finish (press yes if it asks you anything about networks/permissions)
* Click BIOS, and change it to the /x64/OVMF_CODE.secboot option (secboot enables secureboot. If you know you don't want it, you can disable it, but it's necessary for windows 11)
* Apply
* Go to CPUs
* Click topology, and manually set topology.
* Sockets to 1, set cores to -1 from whatever your CPU has, so a 6 core processor would have 5.
* Go to a terminal and type htop. At the top you'll see a bunch of charts going horizontally at the top, which are numbered. On a Ryzen 3600, they go from 0-11, meaning there are 12 threads, or 2x the core count, meaning it has hyperthreading. If you have double as many bars as cores, set threads on your VM to 2. If it's the same as the physical cores you have, set threads to 1.
* Click add hardware in the bottom left of the virtual machine manager, and select storage. Set the Bus Type to VirtIO instead of SATA, and pick however much storage you want.
* Add hardware to your VM again, select storage, and set the device type to CDROM Device. Click "select or create custom storage", then browse to the VirtIO iso. Press finish
* Click on SATA CDROM 2, and click browse, then browse local, and head to your downloads, where you'll double click the VirtIO drivers iso. Press apply.
* Add hardware, TPM, and set version to 2. If that fails, go to the **"For Windows 11 only"** section later when installing, continue normally for now.
* Now, go to the top, and press begin installation.
* When you see a menu which says press any key to boot from CD, press any key. If you miss the time window, close the VM, right click it and force off, then try again.
* If this still doesn't work, you can go to the boot menu section in virt-manager and tick the CD.

**Keep in mind, during first install, graphics and framerate will be pretty bad**

* Go through installation as normal for the first bit.
### For Windows 11 with TPM issues only

* Due to some potential emulated TPM issues, we'll be telling Windows to ignore it. Once you're onto the windows version select screen, press Shift+F10, then type regedit.exe . Now, go to localmachine, system, right click setup, and make a new key called LabConfig.
* Inside LabConfig, make a new 32bit value called BypassTPMCheck. Double click it, set the value to one, then continue setup as normal by selecting a windows 11 edition.

### Back to any Windows version

* Now, press load drivers, then ok, and select the driver that appears with w11 (or whatever other version of windows you're using) in the name. Click it and press next.
* Now click on Drive 0 unallocated space, and press next. Windows will install. 
* When windows restarts, you may be put into windows, but you may be put into a terminal. If in the terminal,click the dropdown near the power icon at the top of the VM, and force it off.
* Press the info icon, and press boot options, and tick VFIO disk 1. Then, go to the 2 CDROM drives, and remove + delete them.
* Now press play, and go back to the view icon next to the info icon. You'll go through the windows personalisation setup now.
* When you're on the windows desktop, shut down windows. You can X out of it in Linux afterwards.

## Hooks
Run 
```
sudo mkdir /etc/libvirt/hooks
sudo wget 'https://raw.githubusercontent.com/PassthroughPOST/VFIO-Tools/master/libvirt_hooks/qemu' -O /etc/libvirt/hooks/qemu
sudo chmod +x /etc/libvirt/hooks/qemu
sudo systemctl restart libvirtd
sudo mkdir -p /etc/libvirt/hooks/qemu.d/Windows/prepare/begin
sudo mkdir -p /etc/libvirt/hooks/qemu.d/Windows/release/end
```
This will download files to do with VM hooks, and make directories, along with restarting libvirtd

#### Start file
Run
```
sudo nano /etc/libvirt/hooks/qemu.d/Windows/prepare/begin/start.sh
```
Paste
```
#Stop disp manager
systemctl stop display-manager.service
#If on Gnome, uncomment one of these if on X or Wayland
#killall gdm-wayland-session
#killall gdm-x-session

#unbind VTconsoles
echo 0 > /sys/class/vtconsole/vtcon0/bind
echo 0 > /sys/class/vtconsole/vtcon1/bind

#Unbind efifb
echo "efi-framebuffer.0" > /sys/bus/platform/devices/efi-framebuffer.0/driver/unbind

#Avoid race condition
sleep 5

#load vfio
modprobe vfio
modprobe vfio_pci
modprobe vfio_iommu_type1
```
Copy This into the text editor, and we'll edit to fit your PC.

For the VTconsoles bit, go into another terminal and run ``ls /sys/class/vtconsole/``. For however many vtcons there are, copy the lines that are already in this code, changing the number to match. Most people just have vtcon0 and 1, which is already setup.

The avoid race condition section basically just waits to ensure previous bits of code are excecuted before continuing. For most people, this does not need to be 10, but for some it does. Start at 10, and then once we're done, try lowering it and seeing what works. Personally, I use 0.5

Then, press **CTRL+X, Y, ENTER**

Now, run
```
sudo nano /etc/libvirt/hooks/qemu.d/Windows/release/end/revert.sh
```
In the text editor, paste this:
```
#unload vfio-pci
modprobe -r vfio_pci
modprobe -r vfio_iommu_type1
modprobe -r vfio

#Rebind VTconsoles
echo 1 > /sys/class/vtconsole/vtcon0/bind
echo 1 > /sys/class/vtconsole/vtcon1/bind

#Rebind efifb
echo "efi-framebuffer.0" > /sys/bus/platform/drivers/efi-framebuffer/bind

#Restart Display Service
systemctl start display-manager.service
```
The first 4 sections should be left alone always. 

For VTconsoles, make the same changes, if any, you made in the start script.

**Now, CTRL+X, Y, ENTER**


Finally, *ensure these scripts are executable!
```
sudo chmod +x /etc/libvirt/hooks/qemu.d/Windows/prepare/begin/start.sh
sudo chmod +x /etc/libvirt/hooks/qemu.d/Windows/release/end/revert.sh
```
# We're nearly there! Give yourself a pat on the back

Open virtual machine manager, and click on your machine. Go to the info icon in the top left, and follow these steps:
* Remove Tablet
* Remove Display Spice (may break. If so, go to the bottom of your XML, and remove audio spice things)
* Remove Serial 1
* Remove Channel spice
* Remove Sound ich9
* Remove Video QXL
* Remove anything else unnecessary if you'd like.
* Add Hardware
* USB Host device
* Find your keyboard here, select it and finish
* Repeat for your mouse
* You can try adding other things, but try them after we have a working VM, since some devices don't like it and cause a crash.
* Add Hardware
* PCI Host device
* Your GPU Video adapter
* Repeat for GPU audio adapter. You can identify either of these using the IDs from before.
* Close the VM window for a sec, go back to the first Virtual Machine Manager window.
* Click edit at the top, followed by preferences
* Enable XML editing, then click back into your VM
* Click on one of the PCIE devices, click xml, and make a line below the line 'source'
* Paste in this line:
```
  <rom file="/etc/libvirt/gpubios/gpubios.rom"/>
```
* Copy that line to the other PCIE device too.
* Now go to overview, and click the XML tab.
* Look for the line beginning with "Spinlock State"
* If you're on NVIDIA, remember to patch your bios like I mentioned earlier, there are guides online, and paste this line in:
````
<vendor_id state="on" value="archlinux"/>
      <vpindex state="on"/>
      <runtime state="on"/>
      <synic state="on"/>
      <stimer state="on"/>
      <reset state="on"/>

````
Then, just below the hyperv line, paste:
````
    <kvm>
      <hidden state="on"/>
    </kvm>
````    

## Restart your PC, and give your VM a go!
## Networking may not work, here's how to fix that.
Enable ACS patching and pass through your ethernet card. Honestly unsure why it isn't working properly sometimes right now.
## If it works, here are some extras.
### Performance
If on ryzen, add this to your CPU section in the XML, something something performance / hyperthreading
````
<feature policy='require' name='topoext'/>
````
For CPU pinning, first replace "Domain Type = KVM" with:
```<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>```
Then, replace the line ending in vcpu with:
```
<vcpu placement="static">8</vcpu>
<iothreads>2</iothreads>
<cputune>
    <vcpupin vcpu="0" cpuset="4"/>
    <vcpupin vcpu="1" cpuset="5"/>
    <vcpupin vcpu="2" cpuset="6"/>
    <vcpupin vcpu="3" cpuset="7"/>
    <vcpupin vcpu="4" cpuset="8"/>
    <vcpupin vcpu="5" cpuset="9"/>
    <vcpupin vcpu="6" cpuset="10"/>
    <vcpupin vcpu="7" cpuset="11"/>
    <emulatorpin cpuset="0-1"/>
    <iothreadpin iothread="1" cpuset="2-3"/>
    </cputune>
 ```
 To modify it, first change the number of vcpus to be the equal to 2x the cores we'll be passing through. Keep 2 cores free in this case, instead of just 1 which i'd do if not pinning. Then, use LSTOPO to match groups of cores. So, vcpu 0 I have = to cpuset 4, then vcpu 1 is cpuset 5. This is because one of my physical cores shows as 'owning' 4 and 5. That's the best explanation I can think of. For emulatorpin, change to one other free core, then set iothread to the final free core.


### I want to pass through devices in the same IOMMU group (this is the section for people with wonky IOMMU groups from earlier)
Well, switch your kernel to Linux-Zen. Or do some other method to get the ACS patch, but I like Linux-Zen. Then, go to your bootloader options, on grub that's 
```
sudo nano /etc/default/grub
```
Then, on the line **GRUB_CMDLINE_LINUX_DEFAULT=**, *inside* the quotation marks, add:
```
pcie_acs_override=downstream,multifunction
```
And run:
```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```
Not only should this fix any GPU-related IOMMU group issues, but it should also allow you to directly passthrough things like your motherboard's NIC, or a PCIE USB card that's not cooperating. Anything PCIE should be free to pass now. Also note: passing a USB controller is MUCH PREFERRED to individually forwarding USB devices. It allows hotplug, and is more compatible


### Anticheat
If you want this VM to be compatible with the most annoying Anti-VM games, go to virt-manager, then the CPU section. Untick "copy host configuration", then set the model to "host-model". Then, boot the VM, and just go to "Turn windows features on or off", and enable Hyper-V. There's a good reason not to have this enabled if you don't absolutely need it, since CPU performance takes a large hit, however this will allow you to bypass *most but not all* VM detection
