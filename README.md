# OpenWRT Build Instructions for WatchGuard Firebox T30 / T30-W

This guide outlines the process to build / flash OpenWRT on the WatchGuard Firebox T30 / T30-W without needing to aquire the partition table of the device.

The reason its hard is because, before, you had to somehow aquire a working sd card with the partition table intact, and copy it to a new sd card.

But heres a way that I found to do it without needing a working sd card.

<br>

## Step 1: Gather Required Materials

- The WatchGuard Firebox T30 comes in 2 variants: 
The standard model and the "W" model, which includes Wi-Fi capabilities. If you have the standard one you will need a AP (Access Point) to provide wireless connectivity, as I did.

- You will also need a SD card with at least 8GB of storage, you can use the one from the T30 device itself, but its not recommended as it may be faulty.

- Its also good to have a serial to usb cable, this can used to access the console of the T30 during the flashing process, This is the one I used: https://www.amazon.com/dp/B075V1RGQK <br> Its not necessary, but it can be helpful for debugging and troubleshooting.

<br>

## Step 2: Clone the OpenWRT repository

Next, we need a openwrt build that is compatible with the WatchGuard Firebox T30. Thankfully, a user by the name of "[dmascord](https://github.com/dmascord)" has already done the work of creating a compatible build. You can find the pull request at the following link:

https://github.com/openwrt/openwrt/pull/4697

It is still a work in progress, but it works well enough for basic functionality. At least it hasn't given me any major issues so far. 😂

You can clone the fork by using the following command:

```bash
git clone https://github.com/dmascord/openwrt.git
```

<br>

## Step 3: Build the firmware

You will need a linux machine to build the firmware.

Now that you have the OpenWRT source code, you can start the build process.
It is the same instructions outlined in the openwrt README.md file either in the link for the pull request or in the cloned repository.

### Now lets download the firmware

First, navigate to the OpenWRT directory:

```bash
cd openwrt
```

Next, you will want to install your build dependencies, you can do that by following [install build system](https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem) instructions from openwrt.

### Update the Feeds

Next, update the feeds:

```bash
./scripts/feeds update -a
./scripts/feeds install -a
```

### Next you'll need to configure the build

This can be done using the following command:

```bash
make menuconfig
```

If you want, you can use my `.config` file from this repo. 

My config contains some extra stuff like the LuCI web interface and some additional packages.

but if you would like to do it yourself the main things you need to change are these 3 values, just match whats in this image:

<img src="https://github.com/DevGamer9991/openwrt-t30-build-instructions/blob/main/images/menuconfig-example.png?raw=true" alt="Menuconfig Settings" width="600"/>

I would also HIGHLY recommend enabling the "LuCI" web interface and the "OpenSSH" server in the "Network" and "Services" sections, respectively.

Once you have made your changes, save the configuration and exit the menu.

Then, run the following command to start the build process, the extra `-j` flag will speed up compilation by utilizing all available CPU cores:

```bash
make -j$(($(nproc)+1))
```

This will take some time, so be patient. Once the build process is complete, you will find the firmware images in the `bin/targets/mpc85xx/p1010` directory.

You will see multiple files, but the 2 you want are the ones with the `.bin` extension.

<img src="https://github.com/DevGamer9991/openwrt-t30-build-instructions/blob/main/images/build-files.png?raw=true" alt="Build Files" width="600"/>

<br>

These are the two files we will be using to flash the firmware onto the device.

<br>

## Step 4: Flashing the firmware

Now we get to the tricky part, flashing the firmware onto the device.

### Copy the fw files to your linux instance

First we need to copy the firmware images to a linux instance, it can be the same one or a different one it just needs to have some way to connect/write to the sd card. 

I will be using `WSL` because its easiest for me.

The files need to be copied with the correct names (These will be important later).

```bash
cp ./openwrt-mpc85xx-p1010-watchguard_t30-w-initramfs-kernel.bin /tmp/uImage_t30

cp ./openwrt-mpc85xx-p1010-watchguard_t30-w-squashfs-sysupgrade.bin /tmp/fw.bin
```

### Instert SD Card Into Linux

I would recommend opening the device now since its easier for testing.

But now you can insert your SD card into your Linux machine, and run this command to get the device id:

```bash
lsblk
```

and write down the device ID that matches the size of your sd card.

It will be used quite a bit later.

### Extract DTB File

So a little bit on how the Firebox T30's bootloader works:

The bootloader, uboot in this case, is looking for 2 specific files on partition 3 of the SD card:

- `t30.dtb`
- `uImage_t30`

These files need to be in the correct partition and have the correct names in order for the bootloader to find them.

So we are going to be "spoofing" them.

To do that we will need to retrieve the Device Tree Blob (DTB) out of the `fw.bin` file

From what I found the `dtb` file can be found at the offset `2048` and has a length of about `14848` bytes

And we can extract it using the following command:

```bash
sudo dd if=/tmp/fw.bin of=/tmp/t30.dtb bs=1 skip=2048 count=14848 status=progress
```

That will output the `t30.dtb` file to the `/tmp` directory.

### Building the MBR

Next, we need to build the Master Boot Record (MBR) for the SD card.

The MBR is a special type of boot sector that contains information about the partitions on the disk and the bootloader.

This can be done using the following commands:

```bash
# create /tmp/mbr (local file, not yet written to device)
sudo dd if=/dev/zero of=/tmp/mbr bs=512 count=1

# clear partition 3 entry bytes (p3 begins at offset 478)
printf '\x00\x00' | sudo dd of=/tmp/mbr bs=1 seek=478 conv=notrunc status=none

# set partition type 0x83 (Linux) at byte 482
printf '\x83' | sudo dd of=/tmp/mbr bs=1 seek=482 conv=notrunc status=none

# CHS start (3 bytes) left zeroed (bytes 483-485)
printf '\x00\x00\x00' | sudo dd of=/tmp/mbr bs=1 seek=483 conv=notrunc status=none

# LBA start = 32768 -> little-endian: 00 80 00 00 (bytes 486-489)
printf '\x00\x80\x00\x00' | sudo dd of=/tmp/mbr bs=1 seek=486 conv=notrunc status=none

# LBA count = 204800 -> 0x00032000 -> little-endian: 00 20 03 00 (bytes 490-493)
printf '\x00\x20\x03\x00' | sudo dd of=/tmp/mbr bs=1 seek=490 conv=notrunc status=none

# boot signature 0x55AA (bytes 510-511)
printf '\x55\xAA' | sudo dd of=/tmp/mbr bs=1 seek=510 conv=notrunc status=none
```

### Write the MBR (destructive)

This is the part that matters, the last commands only generated the mbr, now we are going to write it to the sd card.

Get your device ID from the previous section, and in the next command replace `/dev/sde` with your device ID.

```bash
sudo dd if=/tmp/mbr of=/dev/sde bs=512 count=1 conv=fsync status=progress
```

Then sync it:

```bash
sync
```

### Format Partition 3 and Mount It

Now that we wrote the mbr to the sd card. We are going to format partition 3 and mount it.

And again replace `/dev/sde` with your device ID, but keep `/mnt/sd3` the same.

And also make sure you add the 3 to the end of your device id, as thats for partition 3.

That can be done using these commands:

```bash
sudo mkfs.ext2 -F /dev/sde3
sudo mkdir -p /mnt/sd3
sudo mount -t ext2 /dev/sde3 /mnt/sd3
ls -l /mnt/sd3
```

### Copy Files to Partition 3

Now comes the part of installing the kernel.

The way that we are going to do this is we are going to copy the files to the partition we mounted earlier:

```bash
sudo cp /tmp/uImage_t30 /mnt/sd3/uImage_t30
sudo cp /tmp/t30.dtb /mnt/sd3/t30.dtb
sudo cp /tmp/fw.bin /mnt/sd3/fw.bin
```

Then sync it:

```bash
sync
```

And un-mount it:

```bash
sudo umount /mnt/sd3
```

Thats all we need to do for now.

## Step 5: Setup the Device

So now we have the sd card prepared, now insert it into the device.

Next, if you have one, connect your serial to USB cable to the device, And use a software like PuTTY or minicom to connect to the serial console.

Then power on the device.

You should see it booting up and displaying that it found the `t30.dtb` and the `uImage_t30` files:

<img src="https://github.com/DevGamer9991/openwrt-t30-build-instructions/blob/main/images/booting.png?raw=true" alt="Booting Serial Console" width="400"/>

Now connect your pc to the lan port of the device.

Now, the way the device is booted is that it is running the kernel from RAM, and it not yet written the sysupdate to the sd card.

Therefore, any changes made to the system will be lost on reboot.

We will fix that now

### With LuCI

If you built LuCI into the image using the `.config` file I mentioned above.

The setup is more straightforward.

You should be able to access the device from your web browser at `http://192.168.1.1`. And the username and password are both set to `root` by default.

Then you will be shown a error saying its booted in initramfs mode:

<img src="https://github.com/DevGamer9991/openwrt-t30-build-instructions/blob/main/images/web-panel.png?raw=true" alt="Initramfs Error" width="400"/>

You can fix that by going clicking on `Go to firmware upgrade...`

Then scroll to the bottom of the page and click on the `Flash Image...` button.

<img src="https://github.com/DevGamer9991/openwrt-t30-build-instructions/blob/main/images/flash-firmware.png?raw=true" alt="Initramfs Error" width="400"/>

Then upload the `openwrt-mpc85xx-p1010-watchguard_t30-w-squashfs-sysupgrade.bin` file.

Make sure its the correct file, it MUST end in `squashfs-sysupgrade.bin`

<img src="https://github.com/DevGamer9991/openwrt-t30-build-instructions/blob/main/images/flash-firmware2.png?raw=true" alt="Initramfs Error" width="400"/>

Then click `Upload` and then click `Continue` on the next screen and wait for about 3 minutes to give it time to flash.

Then the webui should come back up and the error should be gone.

### Without LuCI

If you didnt install LuCI, the setup is a little bit more complicated but isnt too difficult.

When the device comes up you should be able to ssh into it with the username of `root` and no password at the ip address `192.168.1.1`.

Then you will be greeted by the console of the router.

Before, when you copied the `uImage_t30` and other files onto partition 3 of the sd card, I also had you copy the `fw.bin` file.

This is the file that we will flash to the device.

This can be done my mounting the partition and using the `sysupgrade` command.

First, we need to mount the partition:

Make sure the directory exists:
```bash
mkdir -p /mnt/sd3
```

Then mount the partition:

```bash
mount /dev/mmcblk0p3 /mnt/sd3
```

Then we can use the `sysupgrade` command to flash the firmware:

```bash
sysupgrade /mnt/sd3/fw.bin
```

Now, wait for about 3 minutes to give it time to flash.

Then you should be able to ssh back into it.

## Step 5: Enjoy!

Congratulations! Your WatchGuard Firebox T30 / T30-W should now be running OpenWRT. You can further customize your setup by installing additional packages, configuring network interfaces, and exploring the many features OpenWRT offers.

If you encounter any issues, check the OpenWRT forums or the GitHub pull request for troubleshooting tips. Enjoy your revitalized hardware!

Happy networking!

## Special Thanks

A huge thank you to "[dmascord](https://github.com/dmascord)" for the initial T30 guide and for creating the fork. Thanks also to everyone who has contributed documentation, troubleshooting tips, and encouragement on the forums and GitHub. Your work makes projects like this possible!