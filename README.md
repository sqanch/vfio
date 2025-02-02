# vfio
VFIO passthrough for Ubuntu 24.04 and an AMD CPU inspired by this [guide](https://mathiashueber.com/passthrough-windows-11-vm-ubuntu-22-04/).

I used Ubuntu Server 24.04 as the host OS and Ubuntu Desktop 24.04 as guest. Here is the used Hardware
 - GPU: Nvidia RTX 2070 Super
 - CPU: AMD Ryzen 3700x
 - Motherboard: Gigabyte X570 Aorus Ultra
 - RAM: Corsair Vengeance 32 GB
 - Storage Host: Crucial MX500 500GB M.2
 - Storage Guest: WD_BLACK 250GB SN770 NVMe

# BIOS settings
```
SVM Mode ->  Enable
IOMMU    ->  Enable
Initial Display Output -> PCIe 2 Slot # only if you want to pass the primary GPU to the VM
```

# Install necesssary packages
```
sudo apt install qemu-kvm qemu-utils libvirt-daemon-system libvirt-clients bridge-utils virt-manager ovmf
```

# Find Device IDs
Execute this [script](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Ensuring_that_the_groups_are_valid), find your GPU, USB controller (and storage) you want to pass to the VM. Write down the device IDs (XXXX:XXXX).

# Edit grub config
Edit `/etc/default/grub` and add the `iommu=pt` as well as the device IDs of your GPU (in my case 4 IDs), the other IDs are not needed.
```
GRUB_CMDLINE_LINUX_DEFAULT="iommu=pt vfio-pci.ids=XXXX:XXXX,YYYY:YYYY,..."
```
Afterwards run `sudo ùpdate-grub` and reboot.

Run `lspci -nnv`, the kernel driver for your GPU should be `vfio-pci`.


# Create bridge network
Edit `/etc/netplan/50-cloud-init.yaml` and add:
```
    bridges:
      br0:
        dhcp4: yes
        interfaces:
          - enp5s0 # your network interface
```
Apply the settings with `sudo netplan apply`.

# Disable AppArmor
I had to disable AppArmor for librivt because it wasn't able to load the profile. Edit `/etc/libvirt/qemu.conf` and add `security_driver = "none"`.

# Create VM
Finally you can create the VM. As I used Ubuntu Server as the host I started virt-manager on another machine and connected it to the host via ssh (File -> Add Connection...). Create a new VM, choose local ISO and select an ISO, specify RAM and CPU, disable storage if you want to pass a drive to the VM, on the last screen make sure to click "Customize configuration before install" and set the network to the previously created bridge `br0`.
In the overview settings make the following changes: Chipset: “Q35”, Firmware: “OVMF_CODE_4M.secboot.fd”.
In the CPU settings you can change the topology to match your physical CPU.
At last click on "Add hardware" and add all the PCIe devices (GPU, USB controller, storage). You may need to enable the storage under Boot Options.

Now you can click on "Begin Installation" and install your guest OS. The installation is done in the Spice Display in virt-manager. Once you are done shutdown the VM and go back to the settings. Remove the "SATA CDROM", "Video QXL" and "Tablet". Afterwards run `virsh edit [VM name]` and remove everything related to spice. Now you can restart your VM with `virsh start [VM name]` or virt-manager, plug in a display to the VM GPU and see the system on the display.
