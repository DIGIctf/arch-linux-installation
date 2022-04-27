# Installing Arch Linux #
04/11/2022

### Downloading ISO ###
[https://archlinux.org/download/](https://archlinux.org/download/) provides a list of downloads for Arch Linux categorized by countries. I selected a link under 'United States' which downloaded the file 'archlinux-2022.04.05-x86_64.iso'.

After the download was complete, I verified the integrity of the ISO file using the following command:
```
sha256sum archlinux-2022.04.05-x86_64.iso
```
This provided the following result:
```
SHA256: 5934a1561f33a49574ba8bf6dbbdbd18c933470de4e2f7562bec06d24f73839b
```
I verified this output from the link I downloaded the ISO file from, under 'checksum'. The output matches. 

### Setting up ISO on VMware ###
Open up VMware
File->New Virtual Machine
Typical->Browse for ISO->Next->Linux
Maximum disk size (GB): 20.0

**Enabling UEFI**
On VMware, right click the VM you created, in my case it's 'Arch Linux Install'->settings. Click the options tab. Next, select 'advanced' and make sure the UEFI is selected. 
![[Pasted image 20220411122849.png]]

Click 'OK' and power on the VM.
![[Pasted image 20220411121909.png]]

My screen looks like this:
![[Pasted image 20220411122107.png]]

The first thing I did was ping google.com to confirm I have an internet connection. I am using ethernet for my internet connection.
```
ping google.com
```

*Tip: ![[Pasted image 20220411130616.png]]At this point I recommend creating a snapshot on your VM.* 

### Partitions ###
Time to setup the partitions. 

We need to find the device name of the hard drive. 
```
fdisk -l
```
This will print a list of hard disks attached to the VM.
![[Pasted image 20220411124128.png]]
The one I will be using is **/dev/sda**
```
fdisk /dev/sda
```

The first partition we will want to create is a GPT partition with 500MB
![[Pasted image 20220411124555.png]]

We need to change the partition type by pressing **t** for the command.
![[Pasted image 20220411124853.png]]
If you want to view a full list of the partition type options, type L at the second prompt. We will need EFI System, which is option 1. 

We will create the next partition very similiar to the last one. Press 'enter' for Partition number (we will be using the default 2) as well as the First & Last sector. 

For the partition type we will want to select **Linux LVM** which is option 43 for me. 
![[Pasted image 20220411125438.png]]

You should have two partitions at this point; 500M EFI System & (remaining space - in my case it's 19.5 GB) Linux LVM.

**IMPORTANT REMINDER**
This last step is VERY important. In order for these changes to take affect we need to write them by entering **w**.
![[Pasted image 20220411125959.png]]

*Tip: ![[Pasted image 20220411130616.png]]At this point I recommend creating a snapshot on your VM.* 

### Setting up LVM ###
Format the first partition
```
mkfs.fat -F32 /dev/sda1
```
Create a physical volume for LVM by using the following command by pointing to the second partition:
```
pvcreate --dataalignment 1m /dev/sda2
```
Next we will create a volume group which is a container of logical volumes. This will be a namespace that we will use to refer to our LVM configuration.
```
vgcreate volgroup0 /dev/sda2
```
Note: You can choose any name where I have *volgroup0*. I chose this name for easy naming convention. 
```
lvcreate -L 14GB volgroup0 -n lv_root
```
Create the root and home directories by the following commands:
```
lvcreate -L 15GB volgroup0 -n lv_root
```
Keep in mind, my VM is 20GB so that is why I chose 15GB for the lv_root. 
```
lvcreate -l 100%FREE volgroup0 -n lv_home
```
By changing the parameter from uppercase -L to lowercase -l we can use 100%FREE left of space to use for the lv_home.

Now, we need to activate it by loading a kernel module into the kernel.
```
modprobe dm_mod
```
We need to scan all the volume groups available on our VM. 
```
vgscan
```
Activate it by:
```
vgchange -ay
```

**Format the root file system**
```
mkfs.ext4 /dev/volgroup0/lv_root

```
Now we need to mount it:
```
mount /dev/volgroup0/lv_root /mnt
```
Next, we need to format lv_home:
```
mkfs.ext4 /dev/volgroup0/lv_home
```
We need to make the home directory before we can mount it:
```
mkdir /mnt/home
```
Now we can mount the directory:
```
mount /dev/volgroup0/lv_home /mnt/home
```

**fstab**
Create the fstab file which will tell the distrubution will find the partitions at boot time. First, let's make a directory etc to store the files.
```
mkdir /mnt/etc
genfstab -U -p /mnt >> /mnt/etc/fstab
```
Verify this is completed by running the following command:
```
cat /mnt/etc/fstab
```
It should show lv_root and lv_home with unique UUID numbers along with the paths.
![[Pasted image 20220411140729.png]]

*Tip: ![[Pasted image 20220411130616.png]]At this point I recommend creating a snapshot on your VM.* 

### Installing ###
Install the base packages.
```
pacstrap -i /mnt base
```
*Note: This may take a little while before it completes*

Configure our installation:
```
arch-chroot /mnt
```
*Note: You should notice a change of your root prompt.*

Make the installation independent of the boot media. Right now our installation is a distribution but not a distribution of Linux because we haven't added Linux yet. We need to install the kernel still.
```
pacman -S linux linux-headers linux-lts linux-lts-headers

```
*Note: This may take a little while to install the kernel and it's dependencies.*

**Installing nano text editor**
My personal favorite Linux text editor is nano. To install:
```
pacman -S nano
```

Add LVM support:
```
pacman -S lvm2
```

We need to make sure our boot process supports our configuration:
```
nano /etc/mkinitcpio.conf
```
Look for HOOKS and add the following change. Add **lvm2** between *block* and *filesystems* as shown below:
![[Pasted image 20220411143858.png]]
To make those options take effect we will want to run each command.
```
mkinitcpio -p linux
mkinitcpio -p linux-lts
```

Update locale settings to your location:
```
nano /etc/locale.gen
```
Once you find your locale, you will want to uncomment it. For the United States, scroll down until you see the following and uncomment it by simply deleting the # before it.
```
en_US.UTF-8 UTF-8
```
Save the file and generate the locale by the following command:
```
locale-gen
```

**Change password** 
It's recommended to set a password for root and create a user for your new system. 
``` 
passwd
``` 
**Create a user**
``` 
useradd -m -g users -G wheel <username_here>
```
*-m creates a user home directory. -g puts the user in a group, which is why we have users. -G wheel  adds the user to administrative tools.*

Check to make sure sudo is installed:
```
pacman -S sudo
which sudo
```

Associate the wheel group with sudo:
```
EDITOR=nano visudo
```
Scroll all the way down and uncomment the following line and save the file:
```
%wheel ALL=(ALL) ALL
```
*This allows the user to use the sudo command and run administrative tasks*

*Tip: ![[Pasted image 20220411130616.png]]At this point I recommend creating a snapshot on your VM.* 

### GRUB ###
Time to make the system bootable and independant of the installation media.

I had to update the system before advancing to adding the packages:
```
pacman -Sy
pacman -S grub dosfstools os-prober mtools
pacman -S grub efibootmgr dosfstools os-prober mtools
mkdir /boot/EFI
mount /dev/sda1 /boot/EFI
grub-install --target=x86_64-efi --bootloader-id=grub_uefi recheck
grub-mkconfig -o /boot/grub/grub.cfg
```

After successfully running the above commands comes the moment of truth:
```
reboot
```

*Tip: ![[Pasted image 20220411130616.png]]At this point I recommend creating a snapshot on your VM.* 

### Post install tweaks ###
Select the correct timezone from the list and set it:
```
timedatectl list-timezones
timedatectl set-timezone America/Denver
systemctl enable systemd-timesyncd
```

Set hostname
```
hostnamectl set-hostname <name_here>
```

### GUI ###
You have several options for the GUI (graphical user interface). I chose XFCE4 after having errors with GNOME. 

```
pacman -S xfce4 xfce4-goodies
<enter> for default Repository extra
```

Enable a login manager:
```
pacman -S lightdm lightdm-gtk-greeter

```
For more information about lightdm, https://wiki.archlinux.org/title/LightDM.

Enable lightdm:
```
systemctl enable lightdm
```

This concludes my write up for installing Arch Linux with XFCE4.
