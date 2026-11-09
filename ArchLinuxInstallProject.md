---
title: Arch Linux Install Project
---

> **The big idea**  
> The slides use a VMware VM. I'm using **KVM through virt-manager** instead, so the VM setup is different from the slides, but the Arch install itself follows the same wiki steps. The main change is that my disk is **`/dev/vda`**, not `/dev/sda`.

---
## Step 0
Boot the virtual machine but because I'm using KVM through virt-manager instead of VMware, the setup is different from the slides.
### What each piece does

| Piece            | Job                                                                                                                                                                                                  |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **KVM**          | Part of the Linux kernel. Turns the kernel into a hypervisor and uses the CPU's virtualization extensions (VT-x on my Intel chip) |
| **QEMU**         | Runs each VM and provides its virtual hardware, like the disk, network card and display                                                                                                              |
| **libvirt**      | Background service that stores VM settings and starts and stops VMs                                                                                                                                  |
| **virt-manager** | The desktop app I click through. It tells libvirt what to do                                                                                                                                         

### 1. Check that the laptop supports KVM

```bash
lscpu | grep Virtualization   # should show VT-x (Intel) or AMD-V (AMD)
ls -l /dev/kvm                # should exist
```

### 2. Install the packages

```bash
sudo pacman -S --needed qemu-desktop libvirt virt-manager dnsmasq edk2-ovmf
```

`dnsmasq` gives VMs their IP addresses and DNS on the virtual network. `edk2-ovmf` is the UEFI firmware for VMs.

### 3. Start libvirt and allow my user to use it

```bash
sudo systemctl enable --now libvirtd.service
sudo usermod -aG libvirt $USER
```

Log out and back in so the new group applies.

### 4. Turn on the virtual network

```bash
sudo virsh net-list --all 
# Should say  default   active   yes    yes
```

### 5. Download and verify the ISO

Download the ISO and its `.sig` file from the [Arch download page](https://archlinux.org/download/), then check the signature:

```bash
pacman-key -v archlinux-2026.11.01-x86_64.iso.sig
```

Look for `Good signature`. The slides also say to compare the SHA256 checksum:

```bash
sha256sum archlinux-2026.11.01-x86_64.iso
```

### 6. Create the VM in virt-manager
0. [Documentation](https://virt-manager.org/screenshots.html)
1. Open virt-manager and click **Create a new virtual machine**.
2. Choose **Local install media (ISO image)**.
3. Browse to the ISO. If it asks to fix search permissions, click **Yes**.
4. Set the operating system to **Arch Linux** if it wasn't detected.
5. Set memory and CPUs, then the disk size, using the table below.
6. Name it `archlinux`, tick **Customize configuration before install**, and click **Finish**.
7. In **Overview**, check the firmware is **UEFI** without secure boot and the chipset is **Q35**.
8. Click **Begin Installation**.

| Setting  | My VM                   | Slide minimum |
| -------- | ----------------------- | ------------- |
| Memory   | 4 GiB                   | 2 GB          |
| CPUs     | 2                       |               |
| Disk     | 30 GiB, `qcow2`, VirtIO | 20 GB         |
| Firmware | UEFI, secure boot off   | UEFI          |
| Network  | `default` (NAT), VirtIO |               |
| Display  | SPICE, VirtIO video     |               |

The disk file grows as it's used, so it doesn't take the full 30 GiB right away. It's at `/var/lib/libvirt/images/archlinux.qcow2`.

### 7. Boot the ISO and check the VM

Choose the Arch install option in the boot menu. Press `Ctrl+Alt` if the VM captures the mouse. Then run:

```bash
cat /sys/firmware/efi/fw_platform_size   # should print 64, which means UEFI
lsblk                                    # should show vda at 30G
ip -br a                                 # the network card should have a 192.168.122.x address
ping -c 3 ping.archlinux.org             # should get replies
```

---

## What's different from the slides

| Topic           | Slides (VMware)                                         | Mine (KVM)                                                                                        |
| --------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| **Disk name**   | `/dev/sda`                                              | `/dev/vda`                                                                                        |
| **Partitions**  | `sda1` (EFI), `sda2` (OS)                               | `vda1` (EFI), `vda2` (OS)                                                                         |
| **UEFI**        | May need `firmware="efi"` in the `.vmx` file on Windows | Set in virt-manager, so no file edit                                                              |
| **Internet**    | Through VMware                                          | Through libvirt's NAT network, so no Wi-Fi setup in the VM                                        |
| **Guest tools** | VMware tools                                            | Optional, once a desktop is installed: `spice-vdagent` for clipboard sharing and display resizing |

The disk is `vda` because it's a **VirtIO** disk. VirtIO is paravirtualized, so the guest knows it's virtual and works with the hypervisor instead of QEMU imitating real SATA hardware.

> **Things that can catch me out**  
> - **Keep the ISO attached until the install is done.** The VM's saved settings boot from the disk only and don't include the ISO. They replace the install settings when the VM first stops, which can look like the VM just turned off. If that happens early, go to **Details → Add Hardware → Storage**, add a **CDROM** with the ISO, then put it first under **Boot Options**.
> - **Install a network manager before rebooting into Arch.** This is the Arch slides' warning. A wired connection in the live ISO doesn't mean the installed system can connect.

## Step 1: The Arch wiki installation steps

### 1.7 Connect to the internet

1. run `ip link`
2. It gives me a `enp1s0` along with an `lo`
3. Try to ping:
   ```bash
   ping ping.archlinux.org
   ```
4. Get the error `temporary failure in name resolution` after looking this up it seems that the VM was having problems getting through the firewall and I had to allow it by running

```bash
sudo ufw allow in on virbr0 to any port 67 proto udp   # let the VM request an IP address (DHCP)
sudo ufw allow in on virbr0 to any port 53             # let the VM use libvirt's DNS
sudo ufw route allow in on virbr0                      # let the VM's traffic out to the internet
```

### 1.8 Update system clock

```bash
timedatectl
```

### 1.9 Partition the disks

> **Note:**  
> I am using KVM and because of that the disk is called `/dev/vda` rather than `/dev/sda`

**List the partitions**

```bash
fdisk -l /dev/vda
```

> **Tip:**  
> Make a snapshot before partitioning because you'll probably make a mistake

**Create a partition table and partitions**

1. Create a new GPT partition table with `g`
2. Partition 1: EFI
   - `n` to start a new partitions with number `1`
   - `2048` first sector, `+1G` for the last sector
   - then change the partition type with `t` to `uefi`
3. Partition 2: The SWAP partition
   - `n` to start a new partition with number `2`
   - default first sector, `+4G` for the last sector
   - `t` to change partition 2 to `swap`
4. Partition 3: The Arch OS
   - `n` to start a new one and then press `enter` 3 times to just fill with default answers
5. Check that they exist with `p` then save with `w` (just like vim)
6. Finally check with:
   ```bash
   lsblk /dev/vda
   ```

### 1.10 Format the Partitions

> **Tip:**  
> Create a snapshot EX: After partition

```bash
mkfs.fat -F 32 /dev/vda1   # Formats your EFI partition to be fat 32
mkfs.ext4 /dev/vda3        # Formats you root partition to Ext4
mkswap /dev/vda2           # formats your swap partition to swap
```

### 1.11 Mount the file system

> **Tip:**  
> Create a snapshot EX: Post format

```bash
mount /dev/vda3 /mnt                 # this will mount the root volume
mount --mkdir /dev/vda1 /mnt/boot    # This will mount the boot partition to the root volume
swapon /dev/vda2                     # Enable the swap with this
```

---

## Step 2: Installation

### 2.1 Check mirror list

Don't need to change anything unless you really care

### 2.2 Install Essential packages

To simplify it:

```bash
pacstrap -K /mnt base linux linux-firmware networkmanager vim
```

| Package          | Why                                                                                                                                                                       |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `base`           | fundamentally needed                                                                                                                                                      |
| `linux`          | The kernel                                                                                                                                                                |
| `linux-firmware` | This is the firmware that is necessary such as WiFi and graphic cards. Technicality don't need this because I'm using KVM and that is already builtin to the base linux that im already running |
| `networkmanager` | What actually allows for connecting to the internet                                                                                                                       |
| `vim`            | This is just preference but I like vim more than nano                                                                                                                     |

---

## Step 3: Configure the System

### 3.1 Fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab   # checks the file lists in vda3, vda2, vda1 by UUID
```

### 3.2 Chroot

To actually be able to interact with the new system environment, tools and configuration in the next steps

### 3.3 Time

```bash
ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
hwclock --systohc
```

### 3.4 Localization

```bash
locale-gen   # Remove the comment under en_US UTF-8
```

```bash
vim /etc/locale.conf   # Creates a file and you should put:
```

```
LANG=_en_US.UTF-8
```

`/etc/vconsole.conf`: Creates a file and you should put:

```
KEYMAP=us
```

### 3.5 Network configuration

```bash
vim /etc/hostname   # Put your hostname EX: saxonarchinstall
systemctl enable NetworkManager   # Necessary for wifi
```

### 3.7 Root password

```bash
passwd
```

### 3.8 Bootloader

```bash
pacman -S grub efibootmgr                                                # install GRUB and its UEFI tool
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB   # install it to the EFI partition
grub-mkconfig -o /boot/grub/grub.cfg                                     # create the boot menu
```

---

## Step 4: Reboot

```bash
exit             # exit the chroot environment
umount -R /mnt   # Optional
```

Power off

Because I'm using KVM I need to remove the ISO with:

```bash
virsh -c qemu:///system change-media archlinux sda --eject --config
```

---

## Step 5: Post Install

### 5.1 GUI-based desktop environment

I decided to install Xfce because it seemed interesting and lightweigh

```bash
pacman -S xorg xfce4 xfce4-goodies lightdm lightdm-gtk-greeter
systemctl enable lightdm   # Enable the lightbox
```

Reboot the system

### 5.2 User accounts

Create user account for me and codi with sudo permissions

```bash
useradd Saxon
usermod -aG wheel Saxon
useradd Codi
usermod -aG wheel Codi
```

### 5.3 Zsh shell install

```bash
pacman -S sudo                # Had to download sudo
sudo pacman -S zsh            # Download the new shell
chsh -s /usr/bin/zsh Saxon    # Set Zsh as default shell
echo $SHELL                   # Restart shell and check whether it worked
```

### 5.4 Install ssh

```bash
sudo pacman -S openssh   # Install openssh
```

### 5.5 Color coding

```bash
sudo pacman -S zsh grml-zsh-config   # install the default coloring package
chsh -s /usr/bin/zsh                 # Make zsh default shell
```

### 5.6 Install some aliases

| Alias                              | what it does                                       |
| ---------------------------------- | -------------------------------------------------- |
| `alias ll='ls -lah'`               | Long listing that includes hidden files, with size |
| `alias update='sudo pacman -Syu'`  | Update the whole system                            |
| `alias please='sudo $(fc -ln -1)'` | rerun the last command with sudo                   |

### 5.7 Install a web browser

I installed Firefox because it's in the official Arch repos, so pacman installs and updates it like everything else

```bash
sudo pacman -Syu                        # Update the system first
sudo pacman -S firefox                  # Install Firefox
sudo pacman -S noto-fonts ttf-dejavu    # Fonts so web pages display correctly
firefox &                               # Open it from the terminal
```

It also shows up in the Xfce menu under **Applications → Internet → Firefox**

> [!tip]
> Create a snapshot EX: After browser install

---

## Step 6: Problems that had to resolved

### 6.1 User Saxon and Codi didnt get home directories

```bash
mkdir /home/Saxon                 # create the home folder
cp -r /etc/skel/. /home/Saxon     # copy the default starter files
chown -R Saxon:Saxon /home/Saxon  # make Saxon the owner
chmod 700 /home/Saxon             # only Saxon can access it
```

### 6.2 sudo wasn't recognized as a command

Using `sudo` as my user gave `sudo: command not found`, because the base install doesn't include sudo.

```bash
su -             # switch to root
pacman -S sudo   # install sudo
```

### 6.3 In the sudo group but still couldn't use sudo

Arch's sudoers file only has a rule for the `wheel` group, and it's commented out, so being in a `sudo` group did nothing.

```bash
EDITOR=vim visudo   # open the sudoers file safely (it checks the syntax before saving)
```

Uncomment this line:

```
%wheel ALL=(ALL:ALL) ALL
```

Then log out and back in so the group change applies, and test it:

```bash
groups        # wheel should be in the list
sudo whoami   # should print root
```

