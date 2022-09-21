![alt tag](https://raw.githubusercontent.com/lateralblast/kvm-nvidia-passthrough/master/images/cat_monitor.jpg)


Nvidia GPGPU Pass-Through with KVM
==================================

Introduction
------------

This is a quick example of how to configure Nvidia GPGPU pass through to a Linux KVM VM.

This guid is for Intel hardware, but a similar approach can be take for AMD.

If successful, as a simple example, you should be able to do something like this from with the KVM VM:

```
$ nvidia-smi
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 390.138                Driver Version: 390.138                   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla M2090         Off  | 00000000:04:00.0 Off |                    0 |
| N/A   N/A    P0    74W /  N/A |      0MiB /  5301MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   1  Tesla M2090         Off  | 00000000:42:00.0 Off |                    0 |
| N/A   N/A    P0    77W /  N/A |      0MiB /  5301MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

Resources
---------

Here are some links that while not providing me with the solution,
gave me some information to point me in the right direction.

Configure GPU Passthrough for Virtual Machines:

https://www.server-world.info/en/note?os=CentOS_7&p=kvm&f=10

KVM: Testing cloud-init locally using KVM for an Ubuntu cloud image:

https://fabianlee.org/2020/02/23/kvm-testing-cloud-init-locally-using-kvm-for-an-ubuntu-cloud-image/

PCI passthrough:

https://pve.proxmox.com/wiki/Pci_passthrough

Virtual machines with PCI passthrough on Ubuntu 20.04, straightforward guide for gaming on a virtual machine:

https://mathiashueber.com/pci-passthrough-ubuntu-2004-virtual-machine/

Cloud Config Examples:

https://cloudinit.readthedocs.io/en/latest/topics/examples.html

Using Cloud Images With KVM:

https://serverascode.com/2018/06/26/using-cloud-images.html

Requirements
------------

Required hardware:

- Nvidia Cuda cable GPU
- Intal / AMD IOMMU Enablement in BIOS
- Intel / AMD IOMMU Enablement in grub

Required software:

- KVM
- Nvidia Drivers
- Nvidia Toolkit and Samples (optional)

A reboot is required to install Nvidia drivers

Overview
--------

This example does the following:

- Determine Nvidia PCI Device IDS
- Enable IOMMU and pass management of devices to vfio
- Disable Nvidia drivers on host
- Install KVM
- Determine Nvidia PCI IDs
- Create KVM VM
- Install Nvidia drivers in KVM VM

Example
-------

This requires you enable IOMMU ot VT-d in BIOS.

Determine Nvidia PCI IDs:

```
$ lspci  |grep -i nvidia
04:00.0 3D controller: NVIDIA Corporation GF110GL [Tesla M2090] (rev a1)
04:00.1 Audio device: NVIDIA Corporation GF110 High Definition Audio Controller (rev a1)
42:00.0 3D controller: NVIDIA Corporation GF110GL [Tesla M2090] (rev a1)
42:00.1 Audio device: NVIDIA Corporation GF110 High Definition Audio Controller (rev a1)
```

Determine Nvidia PCI Device IDs

```
$ lspci -ns 04:00.0 |awk '{print $3}'
10de:1091
$ lspci -ns 04:00.1 |awk '{print $3}'
10de:0e09
```

Enable IOMMU and pass management of devices to vfio in grub:

```
cat /etc/default/grub |grep iommu
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt intremap=no_x2apic_optout vfio-pci.ids=10de:1091,10de:0e09"
```

The key parts in this are:

```
intel_iommu=on iommu=pt vfio-pci.ids=10de:1091,10de:0e09 
```

You may need additional handling to deal with X2APIC alerts/messages:

```
intremap=no_x2apic_optout
```

Update GRUB:

```
$ sudo update-grub
```

Disable nvidia drivers:

```
$ sudo sh -c 'echo "blacklist nvidia" >> /etc/modprobe.d/blacklist.conf'
$ sudo sh -c 'echo "blacklist nvidia_uvm" >> /etc/modprobe.d/blacklist.conf'
$ sudo sh -c 'echo "blacklist nvidia_drm" >> /etc/modprobe.d/blacklist.conf'
$ sudo sh -c 'echo "blacklist nvidia_modeset" >> /etc/modprobe.d/blacklist.conf'
$ sudo sh -c 'echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf'
```

Update vfio config:

```
$ sudo sh -c 'echo "options vfio-pci ids=10de:1091,10de:0e09 disable_vga=1" > /etc/modprobe.d/vfio.conf'
```

Reboot:

```
$ sudo reboot
```

After reboot you should see vfio mention the PCI ID of the Nvidia GPU in dmesg or /proc/interrupts:

```
$ sudo dmesg |grep vfio |grep add
[    9.601364] vfio_pci: add [10de:1091[ffffffff:ffffffff]] class 0x000000/00000000
[    9.653456] vfio_pci: add [10de:0e09[ffffffff:ffffffff]] class 0x000000/00000000
```

If devices are not handled by vfio you may need to allow unsafe config/interrupts:

```
$ sudo echo "options vfio_iommu_type1 allow_unsafe_interrupts=1" > /etc/modprobe.d/iommu_unsafe_interrupts.conf
```

Now we need to install KVM and associated packages:

```
$ sudo apt install -y qemu qemu-kvm libvirt-daemon libvirt-clients bridge-utils virt-manager
```

Add user to appropriate groups (e.g. kvm, libvirt, libvirt-qemu)

Grab a cloud image:

```
$ cd /var/lib/libvirt/images
$ sudo wget http://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
```

Copy Image and resize it:

```
$ sudo qemu-img convert -f qcow2 -O qcow2 focal-server-cloudimg-amd64.img nvubuntu2004vm01.img
$ sudo qemu-img resize nvubuntu2004vm01.img 50G
```

Create a cloud init config:

```
$ sudo cat nvubuntu2004vm01.cfg
#cloud-config
hostname: nvubuntu2004vm01
groups:
  - nvadmin: nvadmin
users:
  - default
  - name: nvadmin
    gecos: NvAdmin
    primary_group: nvadmin
    groups: users
    shell: /bin/bash
    passwd: PASSWORDHASH
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: false
packages:
  - qemu-guest-agent
  - net-tools
  - software-properties-common
  - nvidia-driver-390
  - freeglut3
  - freeglut3-dev
  - libxi-dev
  - libxmu-dev
  - gcc-7
  - g++-7
growpart:
  mode: auto
  devices: ['/']
power_state:
  mode: reboot
```

Create static IP config:

```
$ cat nvubuntu2004vm01_network.cfg
version: 2
ethernets:
  enp1s0:
     dhcp4: false
     # default libvirt network
     addresses: [ 192.168.122.91/24 ]
     gateway4: 192.168.122.1
     nameservers:
       addresses: [ 8.8.8.8,8.8.4.4 ]
```

Create config image/ISO for install:

```
$ sudo cloud-localds --network-config nvubuntu2004vm01_network.cfg nvubuntu2004vm01_cloud.img nvubuntu2004vm01.cfg
```

Build VM:

```
$ sudo virt-install --name nvubuntu2004vm01 --cpu host-passthrough --os-type linux \
--os-variant ubuntu20.04 --host-device 04:00.0 --features kvm_hidden=on --machine q35 \
--disk ./nvubuntu2004vm01.img,device=disk,bus=virtio \
--disk ./nvubuntu2004vm01_cloud.img,device=cdrom --graphics none --virt-type kvm \
--network network=default,model=virtio --import --memory 4096
```

The key things here are to pass the host device to the VM, and hide KVM 
(Nvidia drivers will fail to load if it determines it is a virtualised environment):

```
--host-device 04:00.0 --features kvm_hidden=on
```

