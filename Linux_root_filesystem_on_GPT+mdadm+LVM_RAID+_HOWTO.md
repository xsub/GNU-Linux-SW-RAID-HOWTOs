WRITTEN IN 2016, REDACTED 2021
pawel.suchanecki@gmail.com


#HOWTO INSTALL LINUX MINT[1] TO RAID1 WITH GPT+LVM+MDADM

[1] :(OR OTHER DEBIAN/UBUNTU BASED)

NOTE
---
I install to only one drive (/dev/hdc in example) and then allow mdadm autorecovery replicate the whole disk to the other drive (/dev/hdb) when adding it to RAID1 matrix.

TIPS
--- 
a) /boot can not reside on LVM, but you can "mirror" manually /boot partition to the other drive for resilience
b) dmraid is not mdadm -- dmraid is for cheap raid chip (use: apt remove dmraid -- can mess up grub2)  
c) this assumes


0. prepare a bootable USB stick with Debian-like distro

1. boot live distro with installer, skip installer for now

2a. parted or sgdisk (cgdisk = fdisk for GPT)
2b. create small EFI (for futre use) partition, type 0xEF00 (/dev/sdc1)
2c. create small BIOS boot (for GRUB), type 0xEF02 (/dev/sdc2)
2d. create /boot outside of RAID (/dev/sdc3)
2e. create RAID (0xfd00) for rest of drive (/dev/sdc4)

3. do `apt install mdadm` on live distro

4. create array of your choice like (RAID1 with only one device)
`mdadm --create --level 1 --raid-devices 2 --spare-devices 0 /dev/md0 missing /dev/sdc4`

4. create LVM volume group array
`pvcreate /dev/md0`
`vgcreate vg_COREBOX /dev/md0`

5. add logical volumes 
`lvcreate vg_COREBOX -n lv_HOME_STASH -s 2.5T`
`lvcreate vg_COREBOX -n lv_ROOTFS_MINT18 -s 75GB`
*(note: my system's name is `COREBOX', adding /home and / parts with 2.5TB & 75GB in size, respectively)*

6. run installer, install OS on to (need to do it custom disk setup)
/boot -> /dev/sdc3
/ -> /dev/vg_COREBOX/lv_ROOTFS_MINT18
/home -> /dev/vg_COREBOX_lv_HOME_STASH

6. when prompted, request installation of GRUB to /dev/sdc

7. run console

8. mount your newly created rootfs (need to `mkdir /t` directory first)
`mount /dev/vg_COREBOX/lv_ROOTFS_MINT18 /t`

8. prepare chroot
`cp /etc/resolv.conf /t/etc/`
`mount -o bind /dev/ /t/dev`
`mount -t proc proc /t/proc`
`mount -t sysfs sys /t/sys`
`mount -t devpts devpts /t/dev/pts`
`mkdir /t/hostrun`
`mount --bind /run /t/hostrun/`

9. chroot
`chroot /t`

IN CHROOTED ENV to new system:

10. mount the lvmetad from host into chroot
`mount --bind /hostrun/lvm /run/lvm`

10. mount new /boot
mount /dev/sdc3 /boot`

11. a. `apt install grub2` (should be installed, but we want invoke reconfigure (can use `dpkg-reconfigure grub2`)
11. b. `apt install lvm2` (should be installed)
11. c. `apt install mdadm` (should NOT be installed by default, but critical)

-- this should invoke update-initramfs to make initfs support mdadm

12. exit chrooted
12. unmount everything

13. reboot to your new system

14. copy the partition table to the other device
sgdisk --replicate=/dev/sdb3 /dev/sdc3  
(make sure your SOURCE DEVICE IS SECOND (/dev/sdc3) and argument to --replicate is your TARGET DEVICE)

15. randomize GUIDs on new part
sgdisk -G /dev/sdb

6. add the newly prepared device to /dev/md0 (which is our LVM raid backend)
mdadm --add /dev/md0 /dev/sdb3

really,you need to specify the given RAID partition not the whole disk! (sdb vs sdb3)

DONE.
