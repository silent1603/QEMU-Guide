# QEMU-Guide-in-Windows
QEMU guide for create img , install , run vm

## Getting Started
* download executable file in https://www.qemu.org/download/

## Create Image
Create a disk image in QCOW2 format (qcow2 is slower than the raw format, but uses less disk space). 
- qemu-img create -f qcow2 isoname.cow 40G

## Install the OS
- window host : 
```
qemu-system-x86_64 -m 8G -accel whpx,kernel-irqchip=off -cdrom C:\{USERS}\tamgi\Downloads\archlinux-2024.05.01-x86_64.iso -drive file=arch.cow,format=qcow2  -boot order=d
```
- linux host :
```
qemu-system-x86_64 -m 4G -enable-kvm -cdrom ~/Downloads/ubuntu-21.10-desktop-amd64.iso -drive file=ubuntu.cow,format=qcow2 -boot order=d
```
where:
- enable-kvm increases performances dramatically but it requires the host to load some modules. See the Arch page for more details.
- m 4G allocates 4G of RAM, in place of the 128M otherwise available
You can skip this passage and download prebuilt images, such as, for instance:

- Arch Linux boxes, from: https://gitlab.archlinux.org/archlinux/arch-boxes/
- Guix boxes, from: https://guix.gnu.org/en/download/

## Run the machine
```
export MACHINE=ubuntu.cow
qemu-system-x86_64 -m 4G -smp 2 -enable-kvm \
                   -nic user,hostfwd=tcp::60022-:22 \
                   $MACHINE
```
```
export MACHINE=ubuntu.cow
qemu-system-x86_64 -m 4G -smp 2 -enable-kvm \
                   -display none \
                   -nic user,hostfwd=tcp::60022-:22 \
                   $MACHINE
```
where:

- display none hides the graphical display
- smp 2 tells qemu to use 2 CPUs
- nic user,... makes port 60022 on the host point to port 22 (ssh) on the guest. This allows to ssh on the machine with ssh localhost -p 60022, is sshd is running on the guest.

Where VT-d is supported, this further increases performance:
```
export MACHINE=Arch-Linux-x86_64-basic.qcow2
qemu-system-x86_64 -enable-kvm -device intel-iommu -machine q35 \
                   -m 4G -smp 2 -nic user,hostfwd=tcp::60022-:22 \
                   $MACHINE
```

## Write changes to temporary files
Taken from the [Gentoo QEMU/Options page](https://wiki.gentoo.org/wiki/QEMU/Options#PCI_pass-through%20):
```
qemu-system-x86_64 -snapshot img1.cow
```

## Use an overlay image (Gentoo)
Mostly taken from the [Gentoo QEMU/Options page](https://wiki.gentoo.org/wiki/QEMU/Options#PCI_pass-through%20):
```
# create an overlay image
qemu-img create -b arch-base.qcow2 -f qcow2 overlay.qcow2 -F qcow2

# run it
qemu-system-x86_64 -m 8G -enable-kvm -hda overlay.qcow2
```

## TODO Use an Overlay image (Arch)

It does not seem to work as it is.

Taken from the [ArchWiki QEMU Page](https://wiki.archlinux.org/title/QEMU).

```
# create the overlay image
qemu-img create -o backing_file=img1.raw,backing_fmt=raw -f qcow2 img1.cow

# run the machine and store changes in the overlay
qemu-system-x86_64 img1.cow
```

If paths change, you need to rebase (the example shows an unsafe rebase):
```
qemu-img rebase -u -b /new/img1.raw /new/img1.cow
```

## Share Files with the Host
### Using Virtfs
This is taken from [Stack Exchange - How to share a directory with the host without networking in QEMU?](https://superuser.com/questions/628169/how-to-share-a-directory-with-the-host-without-networking-in-qemu)

Invoke the virtual machine specifying which directory you want to share:
```
export LOCAL_PATH=/home/adolfo/Public/
export MOUNT_TAG=host_public
export MACHINE=ubuntu.cow

qemu-system-x86_64 \
    -virtfs local,path=${LOCAL_PATH},mount_tag=${MOUNT_TAG},security_model=mapped-xattr,id=host0 \
    -enable-kvm -device intel-iommu -machine q35 -m 8G \
    $MACHINE
```

Make sure the host_public device is specified in the /etc/fstab of the Guest:
```
cat /etc/fstab
host_public   /local_directory    9p      trans=virtio,version=9p2000.L   0 0
```

Or, alternately, mount it is with mount:
```
sudo su
mkdir public
mount -t 9p -o trans=virtio host_public public
```

### Using SSHD
Mount a guest directory on the host (assuming sshd is running on the guest):
```
sshfs remote_user@localhost:/home/remote_user ./tmp/ -C -p 60022
```

## VGA
### Use a different driver

Taken from the [Gentoo QEMU/Options](https://wiki.gentoo.org/wiki/QEMU/Options#PCI_pass-through%20) page.

QEMU can emulate several graphics cards:

- vga cirrus - Simple graphics card. Every guest OS has a built-in driver.
- vga std - Support resolutions >= 1280x1024x16. Linux, Windows XP and newer guest have a built-in driver.
- vga vmware - VMware SVGA-II, more powerful graphics card. Install x11-drivers/xf86-video-vmware in Linux guests, VMware Tools in Windows XP and newer guests.
- vga qxl - More powerful graphics card for use with SPICE.
To get more performance use the same color depth for your host as you use in the guest.

### Increase Memory
Add the following option:
```
- device vga,vgamem_mb=256
```
### Serve Web pages
Similar to the previous example, we also forward port 80 to 8080 on the host:
```
qemu-system-x86_64 -m 4G -smp 2 -enable-kvm  ubuntu.cow \
                   -display none \
                   -nic user,hostfwd=tcp::60022-:22,hostfwd=tcp::8080-:80
```
We assume, of course, a web server runs on the guest.