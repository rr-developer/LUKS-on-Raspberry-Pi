## LUKS on Raspberry Pi

***Last updated: February 2021.***


This guide explains how to encrypt the root partition of an SD Card with Raspbian with LUKS. The process requires a Raspberry Pi running Raspbian on the SD Card and a USB memory with the same capacity as the SD Card at least. It is important to have a backup of the SD Card, in case anything goes wrong, because it will be overwritten, and of the USB memory as well.

Full disk encryption is quite easy to perform with modern Linux distributions. Raspberry Pi is an exception because the boot partition does not include most of the needed programs and kernel modules. On the other hand, it is important to use disk encryption with Raspberry Pi because the SD card can be extracted from the unit and its content read quite easily.

### Requirements

Linux kernel 5.0 or later. You can check it with this command:
```
uname -s -r
```
cryptsetup 2.0.6 or later. You can check it with this command:
```
cryptsetup --version
```
You should install the programs needed:
```
sudo apt install busybox cryptsetup initramfs-tools
```
The microprocessors of Raspberry Pi computers do not include AES acceleration. That means that a software implementation of AES is needed, which is slow with modest hardware. Fortunately, there is a recent alternative, Adiantum, that runs fast in software.
Linux kernel 5.0 or later includes the cryptographic kernel modules needed for Adiantum. You can check that every module is present and loaded with this command:
```
cryptsetup benchmark -c xchacha20,aes-adiantum-plain64
```
The output, if everything is all right, will be like this:
```
# Tests are approximate using memory only (no storage IO).
#            Algorithm |       Key |      Encryption |      Decryption
xchacha20,aes-adiantum        256b       111.2 MiB/s       114.6 MiB/s
```
If the execution shows an error message complaining about the cipher not being available, either the kernel modules are not present or are not loaded. You can load the necessary kernel modules for that cipher with ‘modprobe’:
```
sudo modprobe xchacha20
sudo modprobe adiantum
sudo modprobe nhpoly1305
```
If ‘modprobe’ complains about the module not being found, the kernel module is not present. A possible cause cab be that the kernel version is not 5.0 or later.
You can check AES and see the speed difference with Adiantum:
```
# Tests are approximate using memory only (no storage IO).
# Algorithm |   Key |      Encryption |      Decryption
aes-xts        256b        88.7 MiB/s        86.2 MiB/s
```

### Preparing Linux

‘initramfs’ has to be recreated when a new kernel is installed. We need to create a new file:
```
/etc/kernel/postinst.d/initramfs-rebuild
```
and it should have this content:
```bash
#!/bin/sh -e

# Rebuild initramfs.gz after kernel upgrade to include new kernel's modules.
# https://github.com/Robpol86/robpol86.com/blob/master/docs/_static/initramfs-rebuild.sh
# Save as (chmod +x): /etc/kernel/postinst.d/initramfs-rebuild

# Remove splash from cmdline.
if grep -q '\bsplash\b' /boot/cmdline.txt; then
  sed -i 's/ \?splash \?/ /' /boot/cmdline.txt
fi

# Exit if not building kernel for this Raspberry Pi's hardware version.
version="$1"
current_version="$(uname -r)"
case "${current_version}" in
  *-v7+)
    case "${version}" in
      *-v7+) ;;
      *) exit 0
    esac
  ;;
  *+)
    case "${version}" in
      *-v7+) exit 0 ;;
    esac
  ;;
esac

# Exit if rebuild cannot be performed or not needed.
[ -x /usr/sbin/mkinitramfs ] || exit 0
[ -f /boot/initramfs.gz ] || exit 0
lsinitramfs /boot/initramfs.gz |grep -q "/$version$" && exit 0  # Already in initramfs.

# Rebuild.
mkinitramfs -o /boot/initramfs.gz "$version"
```
The file should be made executable:
```
sudo chmod +x /etc/kernel/postinst.d/initramfs-rebuild
```
We also need to specify some programs that need to be included in ‘initramfs’. We do that with a new file at:
```
/etc/initramfs-tools/hooks/luks_hooks
```
and it should have this content:
```bash
#!/bin/sh -e
PREREQS=""
case $1 in
        prereqs) echo "${PREREQS}"; exit 0;;
esac

. /usr/share/initramfs-tools/hook-functions

copy_exec /sbin/resize2fs /sbin
copy_exec /sbin/fdisk /sbin
copy_exec /sbin/cryptsetup /sbin
```
The programs are ‘resize2fs’, ‘fdisk’ and ‘cryptsetup’.
The file should be made executable:
```
sudo chmod +x /etc/initramfs-tools/hooks/luks_hooks
```
‘initramfs’ does not include kernel modules for LUKS and encryption. We need to configure the kernel modules to add. This file has to be edited:
```
/etc/initramfs-tools/modules
```
and the following lines added:
```
algif_skcipher
xchacha20
adiantum
aes_arm
sha256
nhpoly1305
dm-crypt
```
Now we need to build the new ‘initramfs’:
```
sudo -E CRYPTSETUP=y mkinitramfs -o /boot/initramfs.gz
```
You can ignore the warning about ‘cryptsetup’ if the next checking is correct.

We can check that the programs are present in ‘initramfs’ with the following command:
```
lsinitramfs /boot/initramfs.gz | grep -P "sbin/(cryptsetup|resize2fs|fdisk)"
```
We can check that the kernel modules are present in ‘initramfs’ with the following command:
```
lsinitramfs /boot/initramfs.gz | grep -P "(algif_skcipher|chacha|adiantum|aes-arm|sha256|nhpoly1305|dm-crypt)"
```

### Preparing Boot
We need to modify some files before rebooting the Rasperry Pi. These changes are meant to tell the boot process to use an encrypted root filesystem. After the previous changes, the Raspberry Pi will boot correctly. After changing the following files, the Raspberry Pi will not boot to Desktop until the whole process of encrypting the root partition and configuring LUKS is completed. If after modifying the next four files and before the root partition is encrypted anything goes wrong, the changes in the four files can be reverted and the Raspberry Pi would boot normally. It is a good idea to make a copy of those file before modifying them.

**File: /boot/config.txt**
The next line has to be appended at the end of the file:
```
initramfs initramfs.gz followkernel
```

**File: /boot/cmdline.txt**
It contains one line with parameters. One of then is ‘root’, that specifies the location of the root partition. For Raspberry Pi is usually ‘/dev/mmcblk0p2’, but it can also be other device (or the same) specified as “PARTUUID=xxxxx”. The value of ‘root’ has to be change to ‘/dev/mapper/sdcard’. For example, if ‘root’ is:
```
root=/dev/mmcblk0p2
```
it should be changed to:
```
root=/dev/mapper/sdcard
```
also, at the end of the line, separated by a space, this text should be appended:
```
cryptdevice=/dev/mmcblk0p2:sdcard
```

**File: /etc/fstab**
The device for root partition (‘/’) should be changed to the mapper.
For example, if the device for root is:
```
/dev/mmcblk0p2
```
it should be changed to:
```
/dev/mapper/sdcard
```

**File: etc/crypttab**
At the end of the file a new line should be appended with the next content:
```
sdcard	/dev/mmcblk0p2	none	luks
```

Everything is ready now to reboot. After rebooting, Raspbian will fail to start because we have configured a root partition that does not exist yet. After several equal messages indicating the failure, the ‘initramfs’ shell will show.

### Encrypting the root partition

In the ‘initramfs’ shell, we can check that we can use ‘cryptsetup’ with the kernel module ciphers:
```
cryptsetup benchmark -c xchacha20,aes-adiantum-plain64
```
If the test is completed, everything is all right. Now we have to copy the root partition of the SD Card to the USB memory. The idea is to have a copy of the root partition of the SD Card in the USB memory, create an encrypted volume in the root partition (the content will be lost) and copy the root partition in the USB memory back to the encrypted partition of the SD Card. Take into account that the previous content of the USB memory will be lost. Before doing that, we are going to check the root partition and reduce the size of its filesystem to the minimum possible, so that the copy operation takes less time. The command to check it and correct possible errors is:
```
e2fsck -f /dev/mmcblk0p2
```
After finishing, we have to resize the filesystem. It is very important to take note of the size of the filesystem after resizing it, the number of blocks of 4k:
```
resize2fs -fM -p /dev/mmcblk0p2
```
An example of the output of the program is:
```
Resizing the filesystem on /dev/mmcblk0p2 to 2345678 (4k) blocks.
```
The number to take note of is ‘2345678’ in the example.
Once the resizing is finished we are going to get a checksum value of the whole partition. We will do the same after every copy operation between SD card and USB memory. Id the checksum is the same, the copy will be equal. Let’s create the checksum for the root partition to copy. Remember to substitute ‘XXXXX’ with the number of 4k blocks you got after resizing the filesystem. It is useful to add “time ” before “dd” to obtain how long it takes the operation. When the rest of checksums are calculate it, it allows to know, more or less, how much time I will take:
```
time dd bs=4k count=XXXXX if=/dev/mmcblk0p2 | sha1sum
```
Take note of the SHA1 value. 
Connect now the USB memory to the Raspberry Pi. Some information about the USB memory will be printed. If there is no other USB memory connected, the device for the memory will probably be /dev/sda. It can be checked with:
```
fdisk -l /dev/sda
```
We are going to use the whole memory, /dev/sda, but it is possible to mount the memory and create the copy of the root filesystem in the SD Card in a file of the mounted filesystem, rather than in the device.
This operation will copy the root filesystem of the SD Card into the USB memory device, deleting all its content (remember to substitute ‘XXXXX’ with the number of 4k blocks you got after resizing the filesystem):
```
time dd bs=4k count=XXXXX if=/dev/mmcblk0p2 of=/dev/sda
```
While copying to or from SD cards and USB memories a message might appear indicating that task worker has been block for several seconds. It can be ignored if the checksums are correct.

Now we have the original root filesystem and the copy. We also have the checksum of the original. We have to calculate the checksum of the copy and compare them. We can consider they are a exact copy if the checksums coincide. The command to calculate the checksum of the copy is (remember to substitute ‘XXXXX’ with the number of 4k blocks you got after resizing the filesystem):
```
time dd bs=4k count=XXXXX if=/dev/sda | sha1sum
```
Assuming that the checksums are correct, now it is time to encrypt the root filesystem of the SD Card, to create the LUKS volume using ‘cryptsetup’. The are many parameter and possible values for the encryption. This is the command I have chosen:
```
cryptsetup --type luks2 --cipher xchacha20,aes-adiantum-plain64 --hash sha256 --iter-time 5000 –keysize 256 --pbkdf argon2i luksFormat /dev/mmcblk0p2
```
More information about the parameters can be found here: https://man7.org/linux/man-pages/man8/cryptsetup.8.html
The command will ask for a passphrase twice (for confirmation). It is important that the passphrase is long and uses different characters sets (letter, numbers, symbols, uppercase, lower case, etc.).
After creating th LUKS volumen, we have to open it and copy the content of the root filesystem into it. The command to open the LUKS volume is:
```
cryptsetup luksOpen /dev/mmcblk0p2 sdcard
```
It will ask the passphrase we chose in the previous stage. Once opened, we copy the root filesystem in the USB memory into the encrypted volume (remember to substitute ‘XXXXX’ with the number of 4k blocks you got after resizing the filesystem):
```
time dd bs=4k count=XXXXX if=/dev/sda of=/dev/mapper/sdcard
```
After copied, we have to calculate the checksum of the copy once more to validate it. If the checksums coincide, the copy is correct (remember to substitute ‘XXXXX’ with the number of 4k blocks you got after resizing the filesystem):
```
time dd bs=4k count=XXXXX if=/dev/mapper/sdcard | sha1sum
```
In addition to the checksum check, we check the the filesystem of the LUKS volume:
```
e2fsck -f /dev/mapper/sdcard
```
We copied a reduced filesystem. Now we have to expand it to the size of the SD Card:
```
resize2fs -f /dev/mapper/sdcard
```
The process is nearly finished now. The USB memory can be extracted because is not needed anymore. We have to exit for the boot process to continue:
```
exit
```
The boot process will enter into initramfs shell again. At this moment we have to open the LUKS volume we just created to make the root filesystem available:
```
cryptsetup luksOpen /dev/mmcblk0p2 sdcard
```
After opening the LUKS volumen we have to exit again and Raspian will start normally:
```
exit
```

### Booting
We do not want to enter into initramfs shell every time we switch on our Raspberry Pi. We can make Raspbian ask for the password and that’s it. For that, we need to build initramfs once more and reboot:
```
sudo mkinitramfs -o /tmp/initramfs.gz
sudo cp /tmp/initramfs.gz /boot/initramfs.gz
```

After rebooting, a prompt message should appear, something like “Please unlock disk...”. Perhaps the prompt asking for a password will get lost between some star-up messages, but you can enter your passphrase anyway and it should work.

