# Linux write blocker
The kernel patch and userspace tools to enable Linux software write blocking. Useful for computer forensics, incident response and data recovery.

## Background
At present, there are no universal ways to mount a file system truly read-only in Linux. For example, mounting a file system with the "ro" option (command line example: *mount -o ro /dev/sda1 /mnt/sda1/*) doesn't guarantee that a kernel driver will not write to a corresponding block device. According to the man page (*man 8 mount*, the "-r" option):
>Note that, depending on the filesystem type,  state  and  kernel
>behavior,  the  system may still write to the device.  For exam‐
>ple, ext3 and ext4 will replay the journal if the filesystem  is
>dirty.   To  prevent  this kind of write access, you may want to
>mount an ext3  or  ext4  filesystem  with  the  ro,noload  mount
>options  or  set  the block device itself to read-only mode, see
>the blockdev(8) command.

Various file system drivers support additional mount options to disable recovery and/or journaling operations. For example, one can use the combination of "ro" and "noload" mount options to disable journal recovery of Ext3/4 file systems (as mentioned above, command line example: *mount -o ro,noload /dev/sda1 /mnt/sda1/*); but these options don't protect Ext3/4 file systems against automatic orphan inode deletion performed by a kernel driver. Similarly, you can't completely disable journaling in ReiserFS (the support of "nolog" mount option wasn't fully implemented in the driver). Other data alteration issues may be hidden in the Linux source code as well.

Yet another way to mount a file system read-only is to mark a corresponding block device as read-only (using *blockdev* or *hdparm*, both programs do the same system call to mark a block device as read-only; command line examples: *blockdev --setro /dev/sda1; blockdev --setro /dev/sda* and *hdparm -r1 /dev/sda1; hdparm -r1 /dev/sda*). According to the man page (*man 8 hdparm*, the "-r" option):
>-r     Get/set read-only flag for the device.  When set,  Linux  disallows write operations on the device.

Unfortunately, this facility wasn't implemented in a forensically sound way: it's up to a file system driver to check whether an underlying block device is marked as read-only and to refuse writing to that device. If a file system driver lacks these checks (e.g. due to a bug), a write request sent by this driver will hit a physical drive successfully passing through the block device driver. Several issues of this kind were discovered in the past: a kernel was writing the XFS superblock to a read-only block device, a kernel was removing orphan inodes from Ext3/4 file systems on read-only block devices. At present, it's known that Ext3/4 drivers will transfer the error code recorded in the journal (indicating a fatal error occurred within a file system) to the superblock even if the underlying block device is read-only. Many file system drivers supporting the batch discard feature allow userspace programs to discard the unused (free) space of a file system even when an underlying block device is read-only (this results in Trim commands hitting a physical drive).

Moreover, mounting a file system isn't the only dangerous action threatening the integrity of digital evidence. Modern Linux distributions automatically activate Linux RAID, fake RAID and LVM volumes. These actions are subject to the same issues as described above: for example, LVM driver will sync two RAID1-like volumes if they're out of sync, and this will happen to read-only block devices too. Note this when doing data recovery!

The most successful attempt to neutralize possible writes to a block device was to use read-only loop devices. A loop device is a block device managed by the separate kernel driver: all I/O requests originating from a file system driver will first hit the loop device driver and then the block device driver (a loop device acts as a proxy here, translating I/O requests and replies between a file system driver and a block device). Fortunately, the loop device driver will stop all write and discard requests from going to a read-only block device. Unfortunately, there is no easy way to substitute a loop device for a real block device. For a file system residing on a single block device (e.g. */dev/sda1* only), one can use the combination of "ro" and "loop" mount options for the *mount* command from *util-linux* (command line example: *mount -o ro,loop /dev/sda1 /mnt/sda1/*) – both options used together tell the *mount* command to create a loop device for a specified block device, switch this loop device to the read-only mode and then use it to mount a file system (this is similar to the following set of commands: *losetup -r /dev/loop0 /dev/sda1*; *mount -o ro /dev/loop0 /mnt/sda1/*). But there are file systems natively supporting multiple devices configurations: for example, Ext3/4 file systems support external journals (a file system is split between two block devices – one with files and directories, while another one is used for journaling) and Btrfs supports RAID1-like configurations (natively, i.e. without additional hardware or software layers like Linux RAID). In multiple devices configurations, a file system driver is obtaining pointers to other block devices required to mount a file system from a corresponding mount option (e.g. "logdev=" for XFS when external journaling is in place) or from a superblock of a block device specified to the *mount* command. In the latter case, there is no universal way to force a file system driver to use a loop device instead of a block device specified in a superblock. Also, configuration files for Linux RAID and LVM require modifications to enable the activation of volumes residing on loop devices only (and a script is required to locate Linux RAID and LVM block devices and to create a read-only loop device for every block device attributed to Linux RAID and LVM).

In order to support file systems in native multiple devices configurations, we need a better write blocking approach.
 
## Concepts
We utilize the existing facility of marking a block device as read-only and add read-only checks to a common spot in the block device driver. This allows us to block all write and discard requests originating from kernel drivers ignoring the read-only mode of a block device; this also allows us to keep the variety of physical storage device drivers untouched. Our modification is removing the need of the loop device driver, because its primary advantage, the ability to stop write and discard requests from hitting the physical drive, will be replicated in the block device driver.

In particular, we modify the *generic_make_request_checks* function and insert the code used to end I/O processing when a write or discard request is issued to a read-only block device. This modification allows us to intercept write and discard requests at the lowest level possible in the block device driver.

## How to install
Apply the patch to the source code of your kernel and recompile it.

*The patch was written for Linux 3.15.1, but it remains compatible with newer versions of Linux.*

## How to use
Simply run the following command: _blockdev --setro /dev/sdb*_ (assuming */dev/sdb* is the evidence to be write protected).

## Possible drawbacks
### Race conditions
An operating system may write to a drive before you mark a corresponding block device as read-only. Consider disabling automatic mounting of file systems and automatic activation of Linux RAID, fake RAID and LVM volumes.

### Issues with Live CDs and Live USBs
Many live distributions automatically mount file systems on various drives during the boot (in order to locate a bootable medium by searching for specific signatures, e.g. a file with a specific name in a certain directory or a UUID within a specific text file, in a file system of a drive being probed, because a boot loader isn't passing a pointer to a bootable medium to a kernel, and an operating system can be loaded from a medium different from a drive with a boot loader). Consider marking all available block devices as read-only before they (or at least some of them) get mounted by a script in an initial RAM file system (initramfs). Note that even if you don't see a block device mounted after an operating system is loaded, this block device could be mounted and unmounted earlier during the boot.

*Pay attention to this when building a live distribution based on Debian or Ubuntu!*

### Annoying behavior
The patch successfully blocks flush requests going to a device. Some programs (e.g. *parted*) don't like this and throw annoying but harmless exceptions; you can safely ignore them.

If you experience any problems with mounting a file system using the *ntfs-3g* driver, make sure that you include the "ro" mount option (use *mount -t ntfs-3g -o ro /dev/sda1 /mnt/sda1/* instead of *mount -t ntfs-3g /dev/sda1 /mnt/sda1/*).

## Userspace helpers
There are several userspace helpers included to this repository:
- *01wrtblk_all*: the script to mark all available block devices (except loop devices) as read-only;
- *wrtblk*: the script to mark a specified block device (e.g. "sdb1" or "sdc") and its parent block device as read-only;
- *wrtblk-disable*: the script to mark a specified block device and its parent block device as *read-write*;
- *01-forensic-readonly.rules*: the *udev* rule to mark new block devices appearing in a system as read-only using the *wrtblk* script (warning: hard-coded path inside).

## Performance degradation
No performance degradation was detected after applying the patch.

## Debugging
This repository contains the simple *forensic-tracer* kernel module that will log all write and discard requests hitting the *generic_make_request_checks* function (thanks to the *Kprobes* system). Tapping this function is better than intercepting requests inside the *submit_bio* void (like the *block_dump* interface does).