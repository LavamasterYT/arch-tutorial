# Unofficial Arch Install Guide
### Notes:
<> means that you need to change it with your own text (don't include the <>)

**Backup any data before proceeding**

**Read this guide entirely before proceeding to perform this**

On commands that use `sudo`, it can be omitted if you are logged in as the root user.

`$ <command>` represents the live USB shell, whereas `#$ <command>` represents the root partition shell. If neither is there, then it can be applied to both shells. If there is a different prefix, then it is a shell from an application.

## Verifying you are in UEFI mode
If you are booted from the live USB, verify it's in UEFI mode by running
```
$ ls /sys/firmware/efi/efivars
```
If the directory does not exist, then the USB is not booted in UEFI mode. Make sure to boot it into UEFI mode before proceeding.

## Live USB clock
Ensure that the clock is accurate by running this command
```
$ timedatectl set-ntp true
```

## Internet
You need to set up internet before doing a lot of stuff in Arch. Steps are different if you use either Ethernet or WiFi

### Ethernet
If you are booted into the live USB, then you should be good to go. Just run `ping 1.1.1.1` to make sure that you are able to access the internet. If for some reason you are unable to connect, or you are booted into Arch without the live USB, go to the **Setting up DHCP** section in the **WiFi** section

### WiFi
WiFi is a bit more complicated. You have to set up drivers, connect to the router, and set a DHCP thing or something.

#### Connecting to the internet.
We are going to use a tool called `iwd` to do this.

Install the iwd package and start/enable the service, if you are in the live USB, all of this is done for you. If you are booted to Arch, simply run these 2 commands. You only need to run them once as the first one makes it auto start at boot.
```
#$ sudo systemctl enable iwd.service
#$ sudo systemctl start iwd.service
```

First step, open the shell for `iwd` by using this command:
```
sudo iwctl
```
It should bring you to another shell. This will be used for connecting to internet.
Next, list your devices by running this in the shell:
```
[iwd]# device list
```
Now that you know which device is the correct WiFi device, scan for networks using this command:
```
[iwd]# station <device> scan
```
Let it scan for networks. You can list the networks it found by typing:
```
[iwd]# station <device> get-networks
```
Now, you can connect to the network. Type in the following. If the network requires a password, `iwd` will prompt you for it.
```
[iwd]# station <device> connect <SSID>
```
Finally, just exit by typing this:
```
[iwd]# exit
```
#### Setting up DHCP
Now just because you are connected to the network doesn't mean it will work. You have to set up DHCP in order to access the outside network. This is required for both ethernet and WiFi. We are going to be using a tool called `dhclient` to accomplish this. If you are in the live USB, it should be installed. If not, install it to your system from the live USB.

Simply just run this command and wait for it to finish:
```
sudo dhclient -v
```

### Testing the connection
You can test if your internet is set up correctly by running
```
ping 1.1.1.1
```
## Partitioning
Setting up partitions is hard but easy if you understand.

First, list your disks using this command:
```
$ sudo fdisk -l | grep  "Disk /dev/"
```
> If you are logged in as root, you can omit sudo for any of the commands that use sudo

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
In this case, `/dev/nvme0n1` is my main 500GB NVMe SSD, so when I run the `parted` command later on, this is the path I'm gonna use.

Now, we will be using a tool called `parted`  to partition our disks.

Start the parted shell by typing
```
$ sudo parted <path to disk>
```
Remember, the path to disk was the paths that was listed in the `fdisk` output.
Next, format the disk by typing this:
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
Now that we set up our partitions, we have to find their specified paths.

Run this command to find out their paths:
```
$ sudo fdisk -l | grep "<disk path>"
```
A example output:
```
Partition 1 does not start on physical sector boundary.  
Partition 2 does not start on physical sector boundary.  
Disk /dev/sda: 476.94 GiB, 512110190592 bytes, 1000215216 sectors  
/dev/sda1 2048 614399 612352 299M Microsoft basic data  
/dev/sda2 614400 41943039 41328640 19.7G Linux swap  
/dev/sda3 41943040 1000214527 958271488 456.9G EFI System
```
In this case, we need to focus on the last 3 lines, the ones with the disk path and a number.
`/dev/sda1` in my case is my EFI partition since its 299MB or close to 300MB big as shown by `299M` in the line.
`/dev/sda2` is my swap since its 19.7G big and it says Linux swap
`/dev/sda3` is my root partition since its the biggest one at 456.9G
Save these, we will use them in a bit.

Format the EFI partition by running
```
$ mkfs.fat -F32 <path to EFI partition>
```
Format the swap partition by running
```
$ sudo mkswap <swap partition>
$ swapon <swap parition>
```
Format the root partition by running
```
$ mkfs.ext4 <root partition>
```
Finally, mount the necessary partitions so we can work with them.
```
$ mount <root partition> /mnt
$ mount <efi partition> /mnt/efi
```
> If the last command fails, you may need to create the directory by running `mkdir /mnt/efi`

Now, you are ready to install Arch.

## Installing Arch

Now we have to install Arch onto the formatted root partition. We are going to use a tool called `pacstrap` in order to accomplish this. The format for the `pacstrap` command is the following
```
$ pacstrap <root partition> <packages>
```
Now lets install Arch to the root partition. Install it by installing these packages onto the `/mnt` folder where the root partition is mounted.
```
$ pacstrap /mnt base linux linux-firmware
```
This will install Arch onto the drive. After that is done you may want to install some other basic tools before changing root. This can include a text editor or other utilities as the `base` doesn't provide all the utilities. You can install other tools later with the `pacman` command, but for now, install the essential stuff like a text editor.
```
$ pacstrap /mnt neovim ntfs-3g
```
> You may want to install `vim` and `vi` alongside `neovim` as other applications might look for them instead of `neovim` or another text editor you install

Now you have to generate the `fstab` file for the system. Generate it by running the following
```
$ genfstab -U /mnt >> /mnt/etc/fstab
```
Check the resulting file in case of any error
```
$ cat /mnt/etc/fstab
```

## Setting up Arch
Now it's time to set up the Arch system. Change root to the root partition by running
```
$ arch-chroot /mnt
```
This will essentially "ssh" into the system.

### Setting up time

Change the timezone to central time by running
```
#$ ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
```
Then synchronise the hardware clock by running
```
#$ hwclock --systohc
```

### Setting up locale's
**Don't mess this up.** Trust me it's gonna save a lot of headaches later.

Generate the locale by simply running
```
#$ locale-gen
```
Now set the `LANG` variable by running
```
#$ echo "LANG=en_US.UTF-8" > /etc/locale.conf
```
Now edit the `/etc/locale.gen` file. If you are using `neovim`, simply run
```
#$ nvim /etc/locale.gen
```

Now uncomment the following lines (most likely on line 13 and 14)
```
/etc/locale.gen

 13| en_US ISO-8859-1
 14| en_US.UTF-8 UTF-8
```
Finally, regenerate the locale by running
```
#$ locale-gen
```
### Setting up hosts

Set up the hostname by running the following
```
#$ echo "<hostname>" > /etc/hostname
```
The hostname is basically the PC name and cannot have spaces. Try to linit yourself with only dashes, numbers, and letters. Examples could be `arch-pc`, `arch`, `gaming-pc`, etc.

Now setup the hosts file by running
```
#$ echo "127.0.0.1\tlocalhost\n::1\t\tlocalhost" > /etc/hosts
```

### Almost done

Change the root password by running the following. It will promot you for a new password.
```
#$ passwd
```

Now install any extra packages you might want/need. For example, `dhclient` for DHCP, `sudo`, `iwd` for connecting to WiFi, any WiFi drivers you might need, etc. You can do this with the `pacman` command.
```
#$ sudo pacman -S <packages>
```

### Notes
If once you have booted the Arch install and you might have some packages missing, you can boot back into the Arch live USB and change root to the drive. Just remount the root partition as we did a while ago and run the `arch-chroot` command like we did.

## Installing GRUB
Finally, before we boot, we have to install a boot loader and install CPU microcode. To do this run the following command

```
#$ pacman -S grub efibootmgr
```

Now install the microcode, if you have a AMD CPU, install `amd-ucode`, if you have an Intel CPU, install `intel-ucode`
```
#$ sudo pacman -S <microcode package>
```

Now install the GRUB bootloader to the EFI drive by running
```
#$ sudo grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
```
And then create the GRUB configuration file by running
```
#$ sudo grub-mkconfig -o /boot/grub/grub.cfg
```
Finally, you can exit out of the shell by just typing in `exit`, and it should put you right back into the live USB shell.
```
#$ exit
```

## Booting Arch
Before shutting down the live USB, make sure to unmount the drives in case of any error

Run it in the following order:
```
$ umount -R /mnt/efi
$ umount -R /mnt
```
Check for any busy errors after each command. If all goes well, you can finally shutdown the system and boot into Arch!

## Post install

### Network
Now that you are booted into Arch, connect to the internet. Refer to the **Internet** section from before.

### Setting up users
Now it's time to set up users. This is necessary since being logged in as root at all times is dangerous and some applications wont support being ran by root.

Create a new user by running
```
#$ useradd -m <username>
```
> Only numbers, letters, symbols,  allowed just to be safe

And now set it's password by running
```
#$ passwd <username>
```
### Setting up sudo
Add the user to the `wheel` group by running
```
#$ usermod -aG wheel <username>
```
Then, edit the sudoers file and uncomment this line (most likely line 84
```
/etc/sudoers

 84| %wheel ALL=(ALL) ALL
```
Now test out if sudo works by typing in `exit` to back to the login prompt. Login to the new user you create and run this command
```
#$ sudo whoami
```
Type in the root password and check for any errors. If the output says `root` then it's set up.

## Done
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
