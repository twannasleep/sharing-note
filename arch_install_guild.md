# Arch Linux Installation Guide for ThinkPad X1 Extreme Gen 5

## Prerequisites
- Arch Linux ISO: `archlinux-2025.04.01-x86_64.iso`
- Ventoy USB drive

## Step 1: Create Bootable USB
1. Copy the Arch Linux ISO to your Ventoy USB drive
2. Boot from USB (press F12 during startup)
3. Select "Arch Linux install medium" from Ventoy menu

## Step 2: Verify Boot Mode
```bash
cat /sys/firmware/efi/fw_platform_size
```
Should return "64" for UEFI mode.

## Step 3: Connect to Wi-Fi
```bash
iwctl
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect YOURNETWORK
exit
ping -c 3 archlinux.org
```

## Step 4: Update System Clock
```bash
timedatectl set-ntp true
```

## Step 5: Partition the Disk
```bash
lsblk                   # Identify your NVMe drive (/dev/nvme0n1)
gdisk /dev/nvme0n1
```

Create partitions:
- EFI partition: 500MB, type `ef00`
- Swap partition: 16GB, type `8200`
- Root partition: Remainder, type `8300`

## Step 6: Format Partitions
```bash
mkfs.fat -F32 /dev/nvme0n1p1
mkswap /dev/nvme0n1p2
swapon /dev/nvme0n1p2
mkfs.ext4 /dev/nvme0n1p3
```

## Step 7: Mount Partitions
```bash
mount /dev/nvme0n1p3 /mnt
mkdir -p /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

## Step 8: Install Essential Packages
```bash
pacstrap -K /mnt base linux linux-firmware intel-ucode sudo vim networkmanager
```

## Step 9: Generate Fstab
```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

## Step 10: Chroot into New System
```bash
arch-chroot /mnt
```

## Step 11: Set Timezone
```bash
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc
```

## Step 12: Localization
```bash
vim /etc/locale.gen     # Uncomment en_US.UTF-8 UTF-8
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

## Step 13: Network Configuration
```bash
echo "thinkpad-x1" > /etc/hostname
systemctl enable NetworkManager
```

## Step 14: Set Root Password
```bash
passwd
```

## Step 15: Create User
```bash
useradd -m -G wheel twan
passwd twan
```

## Step 16: Configure Sudo
```bash
EDITOR=vim visudo      # Uncomment %wheel ALL=(ALL) ALL
```

## Step 17: Install Bootloader
```bash
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

## Step 18: Install Graphics Drivers
```bash
pacman -S xorg-server nvidia nvidia-utils
```

## Step 19: Finish and Reboot
```bash
exit
umount -R /mnt
reboot
```

## Post-Installation: Install Desktop Environment
```bash
sudo pacman -S gnome
sudo systemctl enable gdm
sudo systemctl start gdm
```

## Additional Packages to Consider
- Web browser: `firefox`
- Terminal emulator: `alacritty` or `gnome-terminal`
- Text editor: `code` (VS Code)
- File manager: `nautilus` (GNOME) or `dolphin` (KDE)
- Office suite: `libreoffice-fresh`
- Media player: `vlc`

## ThinkPad-Specific Packages
- TLP for battery optimization: `tlp`
- Fingerprint reader: `fprintd`
- Trackpoint/touchpad: `xf86-input-libinput`
