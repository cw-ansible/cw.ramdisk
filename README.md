<!--

---
lang: american
---
-->

[![Build Status](https://travis-ci.org/cw-ansible/cw.ramdisk.svg?branch=master)](https://travis-ci.org/cw-ansible/cw.ramdisk)
[![Tweet this](http://img.shields.io/badge/%20-Tweet-00aced.svg)](https://twitter.com/intent/tweet?tw_p=tweetbutton&via=renard_0&url=https%3A%2F%2Fgithub.com%2Fcw-ansible%2Fcw.ramdisk&text=Run%20linux%20from%20a%20%23tmpfs%20filesystem%20in%20%23RAM.)
[![Follow me on twitter](http://img.shields.io/badge/Twitter-Follow-00aced.svg)](https://twitter.com/intent/follow?region=follow_link&screen_name=renard_0&tw_p=followbutton)


# Ramdisk

Create an image of current installation for a RAM only system.


## Usage

Include `cw.ramdisk` role to your playbook.

## Description

This module will create an image of the current suitable for a RAM only
machine running the OS from a *tmpfs*.

## Configuration

See specific documentation in `defaults/main.yml`

## How does it work?

This role deploy in `/boot/ramdisk` the necessary files to create a *RAMdisk*:

- `update-ramdisk`: A script used to create/update the *RAMdisk* image.
- `files-to-exclude`: A list of files or directory to exclude from the
*RAMdisk* image. This list is must be suitable for `tar` `--exclude-from`
and `--anchored` options (please refer to `tar` man page).
- `ramdisk.conf` the configuration file used by `update-ramdisk`.

A complete PXELinux environment is generated into `/tmp/tftpboot/` you can
copy these files into a PXE server. This environment includes:

    |-+ pxelinux.cfg
    | `-- default
    |-- pxelinux.0
    |-- vmlinuz
    |-- initrd.img
    `-- $(hostname -s).tar.gz

If you don't want to use a PXE boot you still can use the *RAMdisk* from local
boot. You just have to copy the `$(hostname -s).tar.gz` file into
`/boot/tmpfs`.


### Internals

Behind the curtain some other files are required (mainly in
`/etc/initramfs-tools`) but you don't need to care about except if you want
see the internals.

- `hooks/tmpfs`: ensures that all required binaries are included into the
  *init* RAM disk file. It also installs the `ssh` authorized keys (as defined
  in the `ramdisk_dropbear_users` variable) for the *initrd* `root`
  user. Therefore you will be able to log into the system during the boot
  process for debugging purposes or to enter the passphrase to decrypt the
  *RAMdisk* image if you have used encryption (see `ramdisk_tar_ext` variable)

- `scripts/tmpfs`: This script does all the job. It is called using the
  `boot=` kernel option.

- `scripts/init-bottom/00-ensure-fstab`: If no `/etc/fstab` file exists the
  system hangs and refuse to boot properly. This script makes sure it does
  exist.

- `scripts/init-bottom/00-ensure-wtmp`: make sure `/var/log/wtmp` exist so
  that the `last` command will work properly.

- `scripts/init-bottom/01-fix-network-interface`: Substitute `allow-hotplug`
  by `auto` in `/etc/network/interfaces`. This make sure that network can be
  restarted correctly.

### `tmpfs` init script internals

This part is a bit tricky. Basically it is responsible for:

- Starting `udev`.
- Setup the network configuration.
- Mounting the root file system from the *RAMdisk* image.


#### Network setting

Following actions are performed for all interfaces as listed by the
`ifconfig` command output:

- Bring the interface up.
- Wait for the interface to be `up` (for a maximum of `$DEVICE_MAX_TRIES`
  defaulted to 10s). for the
- If the interface is UP run the `configure_networking` function. By default
  a DHCP request is done. If you need a static IP please refer to `ip=`
  kernel parameter (see
  https://www.kernel.org/doc/Documentation/filesystems/nfs/nfsroot.txt)

#### Mount the root file system

The root file system is mounted using a *tmpfs*. By default its size is 50%
of avalable memory but you can change it using the `root_tmpfs_size` (don't
use the `tmpfs_size` since it is already use to create a *tmpfs* for the
initrd image).

The *RAMdisk* as given by the `root=` kernel parameter is decompressed into
the root file system. If a passphrase is required you can either:

- enter it from the console
- enter it using ssh:

        ssh root@$host 'echo "PASSPHRASE" > /lib/cryptsetup/passfifo'

- enter using the `tmpfs_password=` kernel parameter. THIS METHOD IS
  STRONGLY DISCOURAGED FOR SECURITY REASONS SINCE THE PASSPHRASE WILL BE
  READABLE FROM `/proc/cmdline`.

If you are booting from a local drive, you need to pass a `tmpfs_boot`
variable to the kernel in order to find the `/boot`. The *RAMdisk* image is
searched into the `tmpfs/${ROOT}`. if the file you passed on the kernel
parameter is not found, a fallback image is searched into the *tmpfs*
directory of the boot partition. If no fallback file is found, the system
tries to boot from the standard local method. If the standard local method
fails, the system panics.

## Additional kernel parameters

- `drop_to_initrd`: if this parameter is `yes` the system panics at early
  stage. This may be useful for debugging. Note that the network is not set
  yet, thus you need a console access to the server.
- `root_tmpfs_size`: the size of the root file system in RAM suitable for
  `mount` `size` option for `tmpfs`, see `mount(8)`. (default `50%`).
- `tmpfs_boot`: the device of where the `/boot` resides (such as
  `/dev/sda1`). `tmpfs_boot` also supports `LABEL=`, `UUID=`, and on some
  Linux versions `PARTLABEL` and `PARTUUID`.
- `ROOT`: the URL of the *RAMdisk* image suitable for curl.
- `init`: should be set to `tmpfs`.
- `tmpfs_password`: The passphrase for encrypted *RAMdisks*. DO NOT USE THIS.

Notes:

- `tmpfs_boot_uuid` takes precedence on `tmpfs_boot` if both are given.

## Deploy *RAMdisk* to a remote system

Assuming the remote system has already been booted from a live system
(*LiveCD* or an other *RAMdisk* system) and the hard drive is `/dev/vda`

We first need to create partitions with following layout:

- `/dev/vda1`: a 2G partition for `/boot`.
- `/dev/vda2`: a big partition for persistent data.
- `/dev/vda3`: a small partition for `grub_bios`.

        parted /dev/vda -s 'mklabel gpt'
        parted -a optimal /dev/vda -s 'mkpart boot xfs 2 2G'
        parted -a optimal /dev/vda -s 'set 1 boot on'
        parted -a optimal /dev/vda -s 'mkpart data xfs 2G 100%'
        parted -a optimal /dev/vda -s 'mkpart bios_grub ext2 1 2'
        parted -a optimal /dev/vda -s 'set 3 bios_grub on'
        mkfs.xfs /dev/vda1
        mount /dev/vda1 /boot

Synchronize both `/boot` and `/etc/grub.d/09-ramdisk` from the master (or an
other *RAMdisk*) server:

    sudo -E rsync -ai --progress  --rsync-path 'sudo rsync' \
	    /boot/* user@target:/boot/
    sudo -E rsync -ai --progress  --rsync-path 'sudo rsync' \
	    /etc/grub.d/09-ramdisk user@target:/etc/grub.d

Finally just install grub to `/dev/vda`, for that we have to make sure that
`grub-mkconfig` won't fail since it doesn't recognize `/dev/root` as a valid
device path for the `/` directory.

	sed -i '/set -e/d' /usr/sbin/grub-mkconfig
	update-grub
	grub-install /dev/vda

And *voilà* now you can safely reboot the server.

## Copyright

Author: Sébastien Gross `<seb•ɑƬ•chezwam•ɖɵʈ•org>` [@renard_0](https://twitter.com/renard_0)

License: WTFPL, grab your copy here: http://www.wtfpl.net
