(c) 2016-2021 pawel.suchanecki@gmail.com


# Debian-based GNU/Linux on RAID 1 with LVM on MDADM & GPT 

TARGET
---
This assumes you have Linux (admin) experience with CLI, semi-manual installs and using chroot environment.

NOTE
---
I install to only one drive (/dev/hdc in example) and then allow mdadm autorecovery replicate the whole disk to the other drive (/dev/hdb) when adding it to RAID1 matrix.

TIPS
--- 

* Although this is for Debian-like distributions, RAID setup steps are generic, "distro-agnostic".

* /boot can not reside on LVM, but you can "mirror" manually /boot partition to the other drive for resilience

* dmraid is not mdadm -- dmraid is for cheap RAID chip (use: `apt remove dmraid` -- can mess up grub2)  


## STEPS


I. PREP INSTALLER
---
 
*Context: FLASHING OS IMAGE*

0. prepare a bootable USB stick with Debian-like distro


II. PREP DRIVES
---

*Context: CLI OF LIVE OS*

1. boot live distro with installer, skip installer for now

2. prepare partition layout (need to do for just 1 drive in RAID1 set)

   * `parted` or `sgdisk` (`cgdisk` = fdisk for GPT)
   
   * create small EFI (for futre use) partition, type 0xEF00 (/dev/sdc1)

   * create small BIOS boot (for GRUB), type 0xEF02 (/dev/sdc2)

   * create /boot outside of RAID (/dev/sdc3)

   * create RAID (0xfd00) for rest of drive (/dev/sdc4)

3. do `apt install mdadm` on live distro

4. create array of your choice like (RAID1 with only one device)

  * `mdadm --create --level 1 --raid-devices 2 --spare-devices 0 /dev/md0 missing /dev/sdc4`

5. create LVM volume group array

  * `pvcreate /dev/md0`

  * `vgcreate vg_COREBOX /dev/md0`

6. add logical volumes 

   * `lvcreate vg_COREBOX -n lv_HOME_STASH -s 2.5T`

   * `lvcreate vg_COREBOX -n lv_ROOTFS_MINT18 -s 75GB`

*note: my system's name is `COREBOX', adding /home and / parts with 2.5TB & 75GB in size, respectively*


III. PREP OS
---

*Context: OS INSTALLER*

7. run installer, install OS on to (need to do it custom disk setup)

mount point | target partition 
------------|--------------------
/boot | /dev/sdc3
/ | /dev/vg_COREBOX/lv_ROOTFS_MINT18
/home | /dev/vg_COREBOX_lv_HOME_STASH

8. when prompted, request installation of GRUB to `/dev/sdc`

9. open console

10. mount your newly created rootfs (need to `mkdir /t` directory first)

  * `mount /dev/vg_COREBOX/lv_ROOTFS_MINT18 /t`

11. prepare chroot

  * `cp /etc/resolv.conf /t/etc/`

  * `mount -o bind /dev/ /t/dev`

  * `mount -t proc proc /t/proc`

  * `mount -t sysfs sys /t/sys`

  * `mount -t devpts devpts /t/dev/pts`

  * `mkdir /t/hostrun`

  * `mount --bind /run /t/hostrun/`

12. chroot

   * `chroot /t`


IV. OS CONFIG 
---

*Context: CHROOTED TO THE NEW SYSTEM*

13. mount the `lvmetad` from host into chroot

   * `mount --bind /hostrun/lvm /run/lvm`

14. mount new /boot

   * `mount /dev/sdc3 /boot`

15. Add software packages for setting GRUB and RAID 
  
   * `apt install grub2` (should be installed, but we want invoke reconfigure (can use `dpkg-reconfigure grub2`)

   * `apt install lvm2` (should be installed)

   * `apt install mdadm` (should NOT be installed by default, but critical)

   *note: this should invoke update-initramfs to add initfs support for mdadm*

16. exit chrooted & unmount everything

17. reboot to your new system


V. CLONE DRIVE(s)
---

*Context: BOOTED INTO THE NEW OS*

18. copy the partition table to the other device

  * `sgdisk --replicate=/dev/sdb3 /dev/sdc3`

*note: make sure your SOURCE DEVICE IS SECOND (/dev/sdc3) and argument to --replicate is your TARGET DEVICE*

19. randomize GUIDs on new part

  * `sgdisk -G /dev/sdb`

20. add the newly prepared device to /dev/md0 (which is our LVM raid backend)

   * `mdadm --add /dev/md0 /dev/sdb3`

*note: really, you need to specify the given RAID partition not the whole disk! (sdb vs sdb3)*


**DONE**
