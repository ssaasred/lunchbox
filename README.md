# lunchbox

Project Lunchbox is a resource for deploying Red Hat technologies on a single server that literally fits in a lunchbox.

# Building a Custom ISO for the initial RHEL+KVM Utility server

In order to utilize Lunchbox, you must install the base kvm image.
This server will serve as the Ansilbe Host for which all the automation 
will execute from. It will also serve as a hypervisor
for some of the necessary infrastructure VMs (like IDM and RHOSP Director).

The procedures below will work to generate a custom ISO file which can be
written to a bootable USB device or could be served from a PXE server.

## Custom ISO Build Process

> NOTE: To start with the official RHEL 7 ISO you will need an active
> RHEL subscription

1. Download the [latest RHEL DVD ISO](https://access.redhat.com/downloads/content/69/ver=/rhel---7/latest/x86_64/product-software)
2. Make some directories to utilize during the process
```
# mkdir -p /mnt/{iso,working}
```
3. Mount the RHEL-DVD.ISO file
```
# mount -t iso9660 <path to rhel-dvd.iso> /mnt/iso
```
4. Copy all of the contents of the .iso to the working directory
```
# cp -rPf /mnt/iso/* /mnt/working
```
5. Create a true repo inside the “Packages” directory that will be used to
install certain packages that are listed in the custom kickstart file
> NOTE: This next step requires installation of the createrepo packages
> ON RHEL/CentOS, yum install createrepo. On Fedora, dnf install createrepo
```
# cd /mnt/working/Packages
# createrepo .
```
8. Copy the custom kickstart file into the /mnt/working directory
> NOTE: the one we are using here is called
> [lunchbox-ks.cfg](https://github.com/RedHatGov/lunchbox/blob/master/lunchbox-ks.cfg)
9. Edit the lunchbox-ks.cfg file for whatever changes are needed in your environment
> NOTE: You will want to change the root password and most the network configs
10. Once all the info is in the kickstart file, you are ready to create your
custom ISO file
```
# cd /mnt/working
# mkisofs -r -T -J -V "lbadmin" -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -eltorito-alt-boot -e images/efiboot.img -no-emul-boot -o ../lbadmin.iso .
```
11. Use the isohybrid command to specify this .iso file is used for UEFI
```
# cd ../
# isohybrid --uefi lbadmin.iso
```
12. Using the ‘dd’ command to copy the .iso over to a usable USB device
```
# dd if=lbadmin.iso of=/dev/sd<usb device location> bs=4M status=progress conv=fdatasync
```
> NOTE: You can watch the buffer get written out with the following command
```
# watch grep -e Dirty: -e Writeback: /proc/meminfo
```
13. Mount the new usb - there should be 2 partitions (ex. /dev/sdc1 & /dev/sdc2)
14. Mount the second device (should be smaller like 8M)
```
# mkdir /mnt/partition2
# mount /dev/sd<device name/number> /mnt/partition2
# cd /mnt/partition2
```
15. You should see a EFI/BOOT/grub.cfg file in the 2nd partition - edit the file
```
#vi /mnt/partition2/EFI/BOOT/grub.cfg

*Look for the search line and make it look like below*
search --no-floppy --set=root -l 'lbadmin'

*Look for the 'linuxefi line for the first installation option
and make it look like below*
linuxefi /images/pxeboot/vmlinuz inst.stage2=hd:LABEL=lbadmin inst.ks=hd:LABEL=lbadmin:/lunchbox-ks.cfg quiet
```
16. After this is verified/changed - you can unmount the usb device
and use it to boot

## Improvements

Anyone trying this process that sees room for improvement, please submit a PR
Harware info: https://www.supermicro.com/manuals/superserver/mini-itx/MNL-2094.pdf

## Prerequisites

You need to configure an ansible host to execute these playbooks
- Ansible >= 2.6
- Git
- pip and pipenv
- screen or tmux (optional and your preference)

## Clone and configure the repository

1. Clone the lunchbox repository
```
$ git clone https://github.com/RedHatGov/lunchbox.git
```
2. Setup your vars file
```
$ cd lunchbox/
$ cp vars/vars.example.yml vars/vars.yml
$ vi vars/vars.yml
```
> NOTE: The vars example file only exposes the variables you must care about. If you need to dig deeper you'll have to dig into the roles.

3. Install the required roles from Ansible Galaxy
```
$ ansible-galaxy install -r playbooks/requirements.yml
```

## Configure pipenv

1. Install pipenv
```
$ pip install pipenv --user
```

2. Initialize the pipenv project
```
$ cd lunchbox/
$ pipenv --python 3.7
$ pipenv install
```

## To deploy the initial RHEL+KVM admin host

1. Follow the [instructions to create a bootable custom ISO](https://github.com/RedHatGov/hattrick/tree/master/admin-iso)
to install the base operating system for what will become the initial RHEL+KVM utility server
2. Verify your networking is the way you expect. We recommend one bridge for the LAN network (192.0.2.0/24) named br1
3. Run the kvm playbook from the lunchbox directory
```
$ ansible-playbook -i inventory/inventory.yml -e @vars/vars.yml playbooks/lunchbox/kvm.yml
```
