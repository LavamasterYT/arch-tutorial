# Unofficial Arch Install Guide

This will guide you into installing Arch Linux onto your system, and how to set it up.

##### It is recommended you test inside a VM before proceeding on a real machine in order to make sure what you are doing.

1. [Verifying you are in UEFI mode](#uefi)
2. [Live USB Clock](#clock)
3. [Internet](#internet)
	1. [Ethernet](#ethernet)
	2. [WiFi](#wifi)
		1. [Checking Drivers](#drivers)
			1. [Broadcom](#broadcom)
		2. [Connecting to the internet](#connecting)
		3. [Setting up DHCP](#dhcp)
	3. [Testing the connectino](#testing)
4. [Partitioning](#partitioning)
5. [Installing Arch](#installing)
6. [Setting up Arch](#settingup)
	1. [Setting up time](#settingtime)
	2. [Setting up locale's](#settinglocale)
	3. [Setting up hosts](#settinghost)
	4. [Finishing Touches](#settingdone)
7. [Installing GRUB](#grub)
8. [Booting Arch](#booting)
9. [Post Install](#postinstall)
	1. [Setting up users](#users)
	2. [Setting up sudo](#sudo)
10. [Finished](#finished)

## Verifying you are in UEFI mode <a name='uefi'></a>
If you are booted from the live USB, verify it's in UEFI mode by running:
```
$ ls /sys/firmware/efi/efivars
```
If the directory does not exist, then the USB is not booted in UEFI mode. Make sure to boot it into UEFI mode before proceeding.

## Live USB clock <a name='clock'></a>
Ensure that the clock is accurate by running:
```
$ timedatectl set-ntp true
```

## Internet <a name='internet'></a>
You need to set up internet before doing a lot of stuff in Arch. Steps are different if you use either Ethernet or WiFi

### Ethernet <a name='ethernet'></a>
> Untested

If you are booted into the live USB, then you should be good to go. Just run `ping 1.1.1.1` to make sure that you are able to access the internet. If for some reason you are unable to connect, or you are booted into Arch without the live USB, go to the [Setting up DHCP](#dhcp) section in the [WiFi](#wifi) section

### WiFi <a name='wifi'></a>
WiFi is a bit more complicated. You have to set up drivers, connect to the router, and set a DHCP thing or something.

#### Checking drivers <a name='drivers'></a>
Verify you have proper WiFi drivers working in the live shell before proceeding.

Verify that a kernel driver is loaded by running:
```
$ lspci -k
```
> Tip: If the output is way too long that it goes off screen, try piping it to less in order to go line by line. `lspci -k | less`

You should see some sort of kernel driver loaded, as shown in this example:
```
06:00.0 Network controller: Intel Corporation WiFi Link 5100
	Subsystem: Intel Corporation WiFi Link 5100 AGN
	Kernel driver in use: iwlwifi
	Kernel modules: iwlwifi
```
If not, more setup may have to be done.

##### Broadcom <a name='broadcom'></a>
If using a Broadcom card, you can load the broadcom-wl driver that is already on the live USB. To load it, we must block the conflicting drivers that are trying to grab the card.

To block, assuming you are in UEFI mode, edit the kernel boot parameters by adding:
```
module_blacklist=b43,b43legacy,ssb,bcm43xx,brcm80211,brcmfmac,brcmsmac,bcma
```
You can use systemd-boot to accomplish this by pressing E to edit the kernel boot parameters before booting into Arch.

#### Connecting to the internet. <a name='connecting'></a>
Connecting to the internet can be a hassle in Arch, however assuming that the kernel drivers are loaded correctly, the live USB already comes with tools that make connecting to the internet easy. One such tool, which we are going to use, is called `iwd`.

To start, launch the `iwd` shell by running:
```
$ iwctl
```
It should bring you to the `iwd` shell. This will be used for connecting to internet.
Next, list your network adapters by running this in the shell:
```
[iwd]# device list
```
Note down the the WiFi adapter that you are going to be using as we are going to use it for the following commands.

You can scan for networks by running:
```
[iwd]# station <device> scan
```
This will scan for networks, however won't print them. You can list the networks it found by running:
```
[iwd]# station <device> get-networks
```
Now, you can connect to the network. Type in the following. If the network requires a password, `iwd` will prompt you for it.
```
[iwd]# station <device> connect <SSID>
```
Finally, just exit by running:
```
[iwd]# exit
```
#### Setting up DHCP <a name='dhcp'></a>
Now just because you are connected to the network doesn't mean it will work. You have to set up DHCP in order to access the outside network. This is required for both ethernet and WiFi. We are going to be using a tool called `dhclient` to accomplish this.

Simply just run this command and wait for it to finish:
```
$ dhclient -v
```

### Testing the connection <a name='test'></a>
You can test if your internet is set up correctly by pinging a server:
```
$ ping 1.1.1.1
```
## Partitioning <a name='partitioning'></a>
Setting up partitions can be tedious, so I will try to explain it the best I can.

First, list your disks using this command:
```
$ fdisk -l | grep  "Disk /dev/"
```

This will list your disks and their respective "paths".
This is an example output:
```
Partition 1 does not start on physical sector boundary.  
Partition 2 does not start on physical sector boundary.  
Disk /dev/nvme0n1: 465.76 GiB, 500107862016 bytes, 976773168 sectors  
Disk /dev/sda: 476.94 GiB, 512110190592 bytes, 1000215216 sectors  
Disk /dev/sdb: 465.76 GiB, 500107862016 bytes, 976773168 sectors  
Disk /dev/sdc: 1.82 TiB, 2000398934016 bytes, 3907029168 sectors
```
> To view more information about a disk, including its partitions, run `fdisk -l /dev/<disk to examine>`

Now, we will be using a tool called `parted`  to partition our disks.

Start the parted shell by typing
```
$ parted /dev/<disk to partition>
```
Next, format the disk and convert it to a gpt disk by running:
```
(parted) mklabel gpt
```
> Changes will not be saved until you quit `parted`

Now, make an EFI partition by running this command:
```
(parted) mkpart "EFI" fat32 1MiB 300MiB
```
Time to explore what that does.
- `mkpart` is the command to create a partition
- `"EFI"` is the name of the new partition
- `fat32` is the file system of the new partition
- `1MiB` this is where it get's complicated. This is where the start of the partition is gonna be located in the device. In this case, we want the EFI partition to be the first partition, hence why it's starting at `1MiB`
- `300MiB` this is the location where the partition is gonna end in the device.

Next, set the EFI partition as the EFI partition by running
```
(parted) set 1 esp on 
```
Now, it is recommended to create a swap partition. Create one using
```
(parted) mkpart "Swap" linux-swap 300MiB 16GiB
```
Just in case you still don't get the last 2 arguments, since the EFI partition is 300MB big, and in the command, we set it to start at `1MiB` and end at `300MiB`, this partition has to start at `300MiB` since thats where the EFI partition ended, and since this swap partition is gonna be around 15.7GB big, we are gonna end it at `16GiB`.

Next, its time to create the main root partition.
Run this to create the root partition
```
(parted) mkpart "Arch" ext4 16GiB 100%
```
This will fill the rest of the disk with the root partition.
Now that we have done everything, quit `parted` to apply the changes and go back to the shell
```
(parted) quit
```
Now that we set up our partitions, we have to find their  paths.

Run this command to find out their paths:
```
$ fdisk -l /dev/<disk that was partitioned>
```

Format the EFI partition by running:
```
$ mkfs.fat -F32 /dev/<EFI partition>
```
Format the swap partition by running:
```
$ mkswap /dev/<swap partition>
$ swapon /dev/<swap parition>
```
Format the root partition by running
```
$ mkfs.ext4 /dev/<root partition>
```
Finally, mount the necessary partitions so we can work with them.
```
$ mount /dev/<root partition> /mnt
$ mount /dev/<efi partition> /mnt/efi
```
> You may need to create the directory if they fail by running `mkdir /mnt/efi`

Now, you are ready to install Arch.

## Installing Arch <a name='installing'></a>

Now we have to install Arch onto the mounted root partition. We are going to use a tool called `pacstrap` in order to accomplish this. The format for the `pacstrap` command is the following
```
$ pacstrap <root partition> <packages>
```
Now lets install Arch to the root partition. Install the following packages to the root partition like so:
```
$ pacstrap /mnt base linux linux-firmware
```
This will install Arch onto root partition. After that is done you may want to install some other basic tools before changing root. This can include a text editor or other utilities as the `base` doesn't provide all the utilities. You can install other tools later with the `pacman` command, but for now, install the essential stuff like a text editor.
```
$ pacstrap /mnt vim
```
> You may want to install `emacs` and `vi` alongside `vim` as other applications like `sudo` might look for them instead of `vim` or another text editor you install

Now you have to generate the `fstab` file for the system. Generate it by running the following
```
$ genfstab -U /mnt >> /mnt/etc/fstab
```
Check the resulting file in case of any error
```
$ cat /mnt/etc/fstab
```

## Setting up Arch <a name='settingup'></a>
Now it's time to set up the Arch system. Change root to the root partition by running
```
$ arch-chroot /mnt
```
This will essentially locally "ssh" into the system. This is a very useful tool, more information about it will be provided in the [troubleshooting]() section.

### Setting up time <a name='settingtime'></a>

Change the timezone to central time by running
```
$ ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
```
Then synchronise the hardware clock by running
```
$ hwclock --systohc
```

### Setting up locale's <a name='settinglocale'></a>
**Don't mess this up.** Trust me it's gonna save a lot of headaches later.

Generate the locale by simply running
```
$ locale-gen
```
Now set the `LANG` variable by running
```
$ echo "LANG=en_US.UTF-8" >> /etc/locale.conf
```
Now edit the `/etc/locale.gen` file and uncomment the following lines (most likely on line 13 and 14)
```
/etc/locale.gen
----------------------
 13| en_US ISO-8859-1
 14| en_US.UTF-8 UTF-8
```
Finally, regenerate the locale by running
```
$ locale-gen
```
### Setting up hosts <a name='settinghost'></a>

Set up the hostname by running the following
```
$ echo "<hostname>" > /etc/hostname
```
The hostname is basically the PC name, and is used to uniquely identify the device on the network. See [RFC-1178](https://datatracker.ietf.org/doc/html/rfc1178) for choosing a hostname. Examples could be `arch-pc`, `arch`, `gaming-pc`, etc.

Now setup the hosts file by running
```
$ echo -e "127.0.0.1\tlocalhost\n::1\t\tlocalhost" >> /etc/hosts
```

### Finishing Touches <a name='settingdone'></a>

Change the root password by running the following. It will prompt you for a new password.
```
$ passwd
```

Now install any extra packages you might want/need. For example, `dhclient` for DHCP, `sudo`, any WiFi drivers you might need, etc. You can do this with the `pacman` command.
```
$ pacman -S <packages>
```
> If once you have booted the Arch install and you might have some packages missing, you can boot back into the Arch live USB and chroot back into the system. Just remount the root partition as we did in the [partitioning](#partitioning) section and run the `arch-chroot` command like we did in the [setting up](#settingup) section.

## Installing GRUB <a name='grub'></a>
Finally, before we boot, we have to install a boot loader and install CPU microcode. To do this run the following command

```
$ pacman -S grub efibootmgr
```

Now install the microcode, if you have a AMD CPU, install `amd-ucode`, if you have an Intel CPU, install `intel-ucode`
```
$ pacman -S <microcode package>
```

Now install the GRUB bootloader to the EFI drive by running
```
$ grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
```
And then create the GRUB configuration file by running
```
$ grub-mkconfig -o /boot/grub/grub.cfg
```
You can make any changes by editing the grub configuration file located in `/etc/default/grub`, just be sure to rerun the `grub-config` command again just like above.

Finally, you can exit out of the shell by just typing in `exit`, and it should put you right back into the live USB shell.
```
$ exit
```

## Booting Arch <a name='booting'></a>
Before shutting down the live USB, make sure to unmount the drives in case of any error

Run in the following order:
```
$ umount -R /mnt/efi
$ umount -R /mnt
```
Check for any busy errors after each command. If all goes well, you can finally shutdown the system and boot into Arch!

## Post install <a name='postinstall'></a>

### Network <a name='network'></a>
Now that you are booted into Arch, connect to the internet. Refer to the [internet](#internet) section from before.

### Setting up users <a name='users'></a>
Now it's time to set up users. This is necessary since being logged in as root at all times is dangerous and some applications wont support being ran by root.

Create a new user by running
```
$ useradd -m <username>
```

And now set it's password by running
```
$ passwd <username>
```
### Setting up sudo <a name='sudo'></a>
Add the user to the `wheel` group by running
```
$ usermod -aG wheel <username>
```
Then, edit the sudoers file and uncomment this line (most likely line 84):
```
/etc/sudoers
-------------------------
 84| %wheel ALL=(ALL) ALL
```
> You may have to use a application called `visudo` which is installed alongside `sudo` to edit the `sudoers` file. This app requires `vi` to be installed.

Now test out if sudo works by typing in `exit` to back to the login prompt. Login to the new user you create and run this command
```
$ sudo whoami
```
Type in the root password and check for any errors. If the output says `root` then it's set up.

## Finished <a name='finished'></a>
You have now set up Arch! Congrats, you can now set up KDE or the Aura package manager or do whatever you want.

## Ref (ignore this part)
> parted /dev/<device to be partitioned>
> mklabel gpt
> mkpart "EFI" fat32 1MiB 300MiB
> set 1 esp on
> mkpart "Swap" linux-swap 300MiB 16GiB
> mkpart "Arch" ext4 16GiB <entire disk size>GiB
> fdisk -l      # use this to find name of devices
> mkswap /dev/<swap partition>
> mkfs.ext4 /dev/<root partition>
> mount /dev/<root partition> /mnt
> mkfs.fat -F32 /dev/<efi parition>
> swapon /dev/<swap parition>
> mount /dev/<efi parition> /mnt/efi

------------------

installing and setting up linux stuff:

> pacstrap /mnt base linux linux-firmware
> pacstrap /mnt sudo ntfs-3g neovim
> genfstab -U /mnt >> /mnt/etc/fstab
> cat /mnt/etc/fstab
> arch-chroot /mnt
> ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
> hwclock --systohc
> locale-gen
** dont fuck these up **
> echo "LANG=en_US.UTF-8" > /etc/locale.conf
> nvim /etc/locale.gen
uncomment these lines (lines 13 and 14):
  en_US ISO-8859-1
  en_US.UTF-8 UTF-8
> locale-gen
> echo "arch" > /etc/hostname
> echo "127.0.0.1\tlocalhost\n::1\t\tlocalhost" > /etc/hosts
> passwd
change user password
>
> grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
> grub-mkconfig -o /boot/grub/grub.cfg
umount -R /mnt/efi
umount -R /mnt

------------------

restart system, boot into arch, and run these:

> ping 1.1.1.1
if there is an connection error then reconnect to wifi again
> fdisk -l
> mount /dev/<root partition> /mnt
> arch-chroot /mnt

------------------

creating users and setting up aura:

> su
> useradd -m <username>
> passwd <username>
> usermod -aG wheel <username>
> nvim /etc/sudoers
uncomment this line (line 82):
# %wheel ALL=(ALL) ALL
> sudo pacman -S base-devel
> git clone https://aur.archlinux.org/aura-bin.git
> makepkg

------------------

setting up plasma:

> sudo pacman -S plasma-desktop xorg-init
> sudo pacman -S konsole firefox dolphin     # install anything you need
> nvim ~/.xinitrc
add these lines if they arent already there:
export DESKTOP_SESSION=plasma
exec startplasma-x11
> sudo pacman -S mesa vulkan-radeon alsa-utils pavucontrol

------------------

**you might want to double check for errors after each command if you are gonna use this
