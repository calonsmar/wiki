# Using USB Storage as Rootfs (extroot)

*This tutorial is on how to use the USB storage device as the rootfs. If you simply want to use the USB storage device to store files (i.e., you won't be installing anything to it), then read [[Tutorials/Using-USB-Storage]]*.

The Omega comes with 16MB of flash storage. While this is enough for the majority of hardware-related tasks, having extra storage space is always better. Even though you can plug in external USB storage devices to teh Omega, but because application packages are typically designed to be installed in the root filesystem (rootfs), those of you who want to install larger `opkg` packages would still be limited by the 16MB flash size.

Luckily, with extroot, you can mount your USB storage device to the rootfs. There are two approaches to this:
* With **pivot-overlay**, you can mount your USB storage device to `/overlay`, which is the writable part of your filesystem that's merged with `/rom` (the read-only part of your filesystem) to generate `/`.
* With **pivot-root**, you can mount your USB storage device to both `/overlay` and `/rom`, which essentially allows you to replace Omega's on-board flash with yoru USB storage device.

[[_TOC_]]

## extroot with pivot-overlay

Pivot-overlay is the recommended extroot implementation because it is easier to set up and future firmware upgrade will still be written to Omega's flash (instead of your USB storage device).

### Step 1. Prerequisites

Using extroot requires the following packages, all of them should already be installed on the default Omega firmware. But just in case you are using your own firmware, here they are again:
* `block-mount`
* `kmod-fs-[filesystem of choice]`, typically `kmod-fs-ext4`
* `kmod-usb-storage-extras`

Install the packages with `opkg`:

```
opkg update
opkg install block-mount kmod-fs-ext4 kmod-usb-storage-extras
```

You also need to have a USB storage device formatted to ext4 (or another filesystem you chose above).

### Step 2. Mounting the USB Storage Device

Next, plug in your USB storage device. If your device is detected correctly by the Omega, it will show up at a device under the `/dev` directory, usually `/dev/sda1`.

Then, create a mount point for your device if you don't already have one with the following command:

```
mkdir /mnt/sda1
```

Finally, mount the device:

```
mount /dev/sda1 /mnt/sda1
```

Your USB storage device should now be accessible at `/mnt/sda1`.

### Step 3. Duplicating the `/overlay` Directory

Now, we are ready to move the `/overlay` directory into the USB storage device. 

```
mount /dev/sda1 /mnt ; tar -C /overlay -cvf - . | tar -C /mnt -xf - ; umount /mnt
```

### Step 4. Setting up the `/overlay` Directory to Automount on Startup

First, we are going to generate the `fstab` template:

```
block detect > /etc/config/fstab
```

Then, we are going to edit the `/etc/config/fstab` file, which we have just created, to enable auto-mounting the `/overlay` directory. Look for the line

```
option  target  '/mnt/sda1'
```

and change it to:

```
option target '/overlay'
```

Then, look for the line: 

```
option  enabled '0'
```

and change it to 

```
option  enabled '1'
```

Save the file and restart the Omega:

```
reboot
```

And voilà! Your Omega should automatically mount the `/overlay` directory. From this point on, all changes to your filesystem will be made on your USB storage device. Please remember that if you startup your Omega with out plugging in the USB storage device, some settings may be different because all the settings will be saved on your USB device.


## exroot with pivot-root

Pivot-root is the extroot implementation that completely replaces the flash storage on the Omega with your USB storage device. Using pivot-root method means that all fugure firmware upgrade (unless it is done by uboot) will be done to your USB storage device.

### Step 1. Prerequisites

Using extroot requires the following packages, all of them should already be installed on the default Omega firmware. But just in case you are using your own firmware, here they are again:
* `block-mount`
* `kmod-fs-[filesystem of choice]`, typically `kmod-fs-ext4`
* `kmod-usb-storage-extras`

Install the packages with `opkg`:

```
opkg update
opkg install block-mount kmod-fs-ext4 kmod-usb-storage-extras
```

You also need to have a USB storage device formatted to ext4 (or another filesystem you chose above).

### Step 2. Mounting the USB Storage Device

Next, plug in your USB storage device. If your device is detected correctly by the Omega, it will show up at a device under the `/dev` directory, usually `/dev/sda1`.

Then, create a mount point for your device if you don't already have one with the following command:

```
mkdir /mnt/sda1
```

Finally, mount the device:

```
mount /dev/sda1 /mnt/sda1
```

Your USB storage device should now be accessible at `/mnt/sda1`.

### Step 3. Duplicating your rootfs

Next, we must make a complete copy of the of the root filesystem to the external rootfs device. To do this, we will first create a temporary directory that acts as the mount point for the current `/` directory:

```
mkdir -p /tmp/cproot
```

Then, we will create a bind mount of `/` at `/tmp/cproot`:

```
mount --bind / /tmp/cproot
```

Now, we will make a complete copy of the rootfs to your USB storage device:

```
tar -C /tmp/cproot -cvf - . | tar -C /mnt/sda1 -xf -
```

Finally, we will unmount the bind mount:

```
umount /tmp/cproot
```

Great! Now you have created a complete copy of your current rootfs on your USB storage device!

### Step 4. Configure Omega to Automount USB Storage Device on Boot

We will now configure the Omega to automatically mount your USB storage device to `/` on startup. This essentially means that we will be using the firmware on Omega's own flash storage as a "bootloader" that run the filesystem on your USB storage as rootfs. To do this, we will open up `/etc/config/fstab`, and add the following lines to it:

```
config mount
    option target        /
    option device        /dev/sda1
    option fstype        ext4
    option options       rw,sync
    option enabled       1
    option enabled_fsck  0
```

Save the file and reboot:

```
reboot
```

And voilà! Your Omega should automatically mount your USB storage device as its rootfs. From this point on, all changes to your filesystem, including firmware upgrades, will be made on your USB storage device. If you change your mind and want to revert back to running the firmware off Omega's flash storage, simply unplug the USB storage and reboot the Omega.

Happy hacking!
