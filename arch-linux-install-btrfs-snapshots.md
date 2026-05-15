# ARCH LINUX ON ROOT WITH BTRFS AND SNAPPER SNAPSHOTS GUIDE

Set device variable

```bash
export DISK=/dev/nvme0n1
export ROOT_DEVICE="/dev/${DISK}p2"
export EFI_DEVICE="/dev/${DISK}p1"
```

Partition the disk

```bash
cfdisk $DISK
```

we need 2 partitions:

- EFI partition: 10GB
- ROOT parittion: rest of the drive

Format the partitions

```bash
mkfs.fat -F 32 $EFI_DEVICE
mkfs.btrfs $ROOT_DEVICE
```

Create root subvolumes

```bash
mount $ROOT_DEVICE /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@tmp
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@log

umount /mnt
```

Create and mount internal core directories to root subvolumes

```bash
mount -o subvol=@ $ROOT_DEVICE /mnt
mkdir -p /mnt/{home,.snapshots}
mkdir -p /mnt/var/{log,cache,tmp}
mount -o subvol=@home $ROOT_DEVICE /mnt/home
mount -o subvol=@snapshots $ROOT_DEVICE /mnt/.snapshots
mount -o subvol=@cache $ROOT_DEVICE /mnt/var/cache
mount -o subvol=@tmp $ROOT_DEVICE /mnt/var/tmp
mount -o subvol=@log $ROOT_DEVICE /mnt/var/log
```

Check if everything is right with

```bash
btrfs subvolume list /mnt
```

Create boot dir

```bash
mkdir /mnt/boot
mount $EFI_DEVICE /mnt/boot
```

Update mirrors

```bash
reflector --latest 10 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```

Install the system and essential packages

```bash
pacstrap -K /mnt base base-devel linux linux-firmware amd-ucode networkmanager btrfs-progs efibootmgr limine
```

## Install Arch

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

chroot into the newly created arch root

```bash
arch-chroot /mnt
```

## Postinstall

Set time

```bash
ln -sf /usr/share/zoneinfo/Area/Location /etc/localtime
hwclock --systohc
```

Set localizations

```bash
# uncomment locales to generate
nano /etc/locale.gen 

# generates locales
locale-gen

# configure locales
echo "LANG=en_GB.UTF-8" > /etc/locale.conf

# set console keymap
echo 'KEYMAP=us' > /etc/vconsole.conf
```

Set hostname

```bash
echo 'yourhostname' > /etc/hostname
```

## Generate kernel image

run mkinitcpio

```bash
mkinitcpio -P
```

## User configurations

Set root password

```bash
passwd
```

Create user with admin credentials

```bash
useradd -G power,ftp,games,network,power,i2c,video,storage,input,audio,wheel -s /bin/bash username
passwd username
```

## Configure Limine bootloader

Install base Limine bootloader into EFI

```bash
mkdir -p $EFI_DEVICE/EFI/arch-limine
cp /usr/share/limine/BOOTX64.EFI $EFI_DEVICE/EFI/arch-limine/

efibootmgr \
      --create \
      --disk $EFI_DEVICE \
      --part Y \
      --label "Arch Linux Limine Boot Loader" \
      --loader '\EFI\arch-limine\BOOTX64.EFI' \
      --unicode
```

Configure Limine boot entries

```bash
cat > /boot/limine.conf << EOF
timeout: 5

/Arch Linux
    protocol: linux
    kernel_path: boot():/vmlinuz-linux
    kernel_cmdline: root=UUID=$(findmnt -no UUID /) rw rootflags=subvol=@ initrd=/amd-ucode.img
    module_path: boot():/initramfs-linux.img
EOF
```
