# alpine-linux-desktop-guide
A memorization help for myself to remember what i have done to setup my laptop workstation

Basically i run a partial disc entryption with unencrypted EFI and encrypted
root using EFI STUB. Desktop wise i use gnome-shell.

## Installing to disk
This notes take some implicit assumptions regarding disk naming and the use
of an uefi capable machine as well as booting with EFI STUB.

The installation process contains some variables. These made my work easier

### Step zero: create the installation media
Gather the install media from [here](https://alpinelinux.org/downloads/)
and dd it to your storage. For example
```sh
su -l
dd if=/tmp/alpine-standart.iso of=/dev/sdc; sync
```

### Step two: register the preferred keyboard language
```sh
setup-keymap
```

### Step three: set the hostname
```sh
setup-hostname
```

### Step four: change the timezone
```sh
setup-timezone
```

### Step five: configure networking
```sh
setup-interfaces
rc-service networking restart
```

### Step six: some service related stuff
```sh
rc-update add networking boot
rc-update add urandom boot
rc-update add acpid default
```

### Step seven: start powermanagement in live session
```sh
rc-service acpid start
```

### Step eight: point hostname domain to local host ip
```sh
HOSTNAME=$(echo /etc/hostname)
sed -e "s/localhost /${HOSTNAME} localhost /" /etc/hosts > /tmp/hosts
cp /tmp/hosts /etc/hosts
```

### Step nine: enable ntp
```sh
setup-ntp
```

### Step ten: initialize apk package manager
```sh
setup-apkrepos
# change mirror to https
sed -e "s/http/https/" /etc/apk/repositories > /tmp/repositories
cp /tmp/repositories /etc/apk/repositories

apk update
apk upgrade -a
```

### Step eleven: install needed tools
```sh
# intel-ucode may not be needed for you
apk add lvm2 util-linux cryptsetup e2fsprogs dosfstools cfdisk mkinitfs efibootmgr intel-ucode
```

### Step twelve: raw partitioning
First of all a gpt partition is needed with following layout
| first partition | second partition |
| --- | --- |
| EFI (topmost item) | Linux (Default) |

EFI partition will hold bootloader, kernel and initramfs an is FAT32
Linux partition will be encrypted with LUKS later on
```sh
# overwrite first 100M with junk (could be sda instead of nvme0n1)
DEVNAME_PHYSICAL_DISK="nvme0n1"
dd if=/dev/urandom/ of=/dev/${DEVNAME_PHYSICAL_DISK} count=200000
# partitioning tool (could be automated)
cfdisk /dev/${DEVNAME_PHYSICAL_DISK}
```

### Step thirteen: raw partition formatting
```sh
DEVNAME_EFI_PARTITION=""
DEVNAME_ENCRYPTED_PARTITION=""
# nvme drives have partition names pX instead of X
case ${DEVNAME_PHYSICAL_DISK} in
  *nvme*)
    DEVNAME_EFI_PARTITION=${DEVNAME_PHYSICAL_DISK}p1
    DEVNAME_ENCRYPTED_PARTITION=${DEVNAME_PHYSICAL_DISK}p2
  ;;

  *)
    DEVNAME_EFI_PARTITION=${DEVNAME_PHYSICAL_DISK}1
    DEVNAME_ENCRYPTED_PARTITION=${DEVNAME_PHYSICAL_DISK}2
  ;;
esac
    

# optimized for security of encrypted partition
cryptsetup -v -c aes-xts-plain64 -s 512 --hash sha512 --pbkdf pbkdf2 --iter-time 5000 --use-random luksFormat /dev/${DEVNAME_ENCRYPTED_PARTITION}
mkfs.fat -F32 /dev/${DEVNAME_EFI_PARTITION}
```

### Step fourteen: create lvm backed root and swap on the encrypted partition
```sh
# note: some naming scheme that i like
DEVNAME_DECRYPTED_PARTITION=${DEVNAME_ENCRYPTED_PARTITION}.dec
DEVNAME_DECRYPTED_VOLGROUP=${DEVNAME_ENCRYPTED_PARTITION}.vg

cryptsetup open /dev/nvme0n1p2 ${DEVNAME_DECRYPTED_PARTITION}
pvcreate /dev/mapper/${DEVNAME_DECRYPTED_PARTITION}
vgcreate ${DEVNAME_DECRYPTED_VOLGROUP} /dev/mapper/${DEVNAME_DECRYPTED_PARTITION}

lvcreate -L 2G ${DEVNAME_DECRYPTED_VOLGROUP} -n swap
lvcreate -l 100%FREE ${DEVNAME_DECRYPTED_VOLGROUP} -n root
```

### Step fifteen: virtual partition formatting
```sh
mkswap /dev/mapper/${DEVNAME_DECRYPTED_VOLGROUP}-swap
mkfs.ext4 /dev/mapper/${DEVNAME_DECRYPTED_VOLGROUP}-root
```

### Step sixteen: install to disk
```sh
# mount /
mount -t ext4 /dev/mapper/${DEVNAME_DECRYPTED_VOLGROUP}-root /mnt/

# mount swap
swapon /dev/mapper/${DEVNAME_DECRYPTED_VOLGROUP}-swap

# mount the EFI partition inside /mnt/boot
mkdir -p /mnt/boot/efi
mount -t vfat /dev/${DEVNAME_EFI_PARTITION} /mnt/boot/efi/

# finally the setup
setup-disk -m sys /mnt/
```

### Step seventeen: fix swap missing in /mnt/etc/fstab
```sh
# search swap partition by UUID and add fstab entry for it
(blkid | grep swap | sed -e "s/.* UUID=\"/UUID=/" | sed -e "s/\" .*/    swap    swap    defaults    0 0 /") >> /etc/fstab
# note: i am not sure if other partitions are by uuid or by path
#       all mounts without UUID should be replaced with UUID notion where
#       applicable
```

### Step eighteen: change initramfs modules to the ones needed
```sh
INITRAMFS_MODULES="base usb ext4 lvm nvme cryptsetup keymap nvme"
sed -e "s/features=\".*\"/features=\"${INITRAMFS_MODULES\"/" /mnt/etc/mkinitfs/mkinitfs.conf
# note: your range may differ
mkinitfs -c /mnt/etc/mkinitfs/mkinitfs.conf -b /mnt/ $(ls /mnt/lib/modules/)
```

### Step nineteen: use EFI stub as bootloader
alpine as by the time of writing only supports the automatic install of
grub. This does not fit my usecase, so some adjustments have to be made

when a kernel update comes, the copy steps of vmlinuz and initramfs and 
intel-ucode to EFI partition have to be done by hand (as of time of writing)
```sh
# create dirs in the EFI partition
mkdir -p /mnt/boot/efi/EFI/alpine

# setup the chroot
mount -t proc /proc /mnt/proc
mount --rbind /dev /mnt/dev
mount --make-rslave /mnt/dev
mount --rbind /sys /mnt/sys

# enter the freshly installed system with chroot
chroot /mnt
source /etc/profile
export PS1="(chroot) $PS1"

# copy finished config, kernel and initramfs to EFI partition
cp /boot/vmlinuz-lts /boot/efi/EFI/alpine/
cp /boot/initramfs-lts /boot/efi/EFI/alpine/
cp /boot/intel-ucode.img /boot/efi/EFI/alpine/

cp /boot/efi/EFI/alpine/vmlinuz-lts /boot/efi/EFI/alpine/vmlinuz-lts-old
cp /boot/efi/EFI/alpine/initramfs-lts /boot/efi/EFI/alpine/initramfs-lts-old
cp /boot/efi/EFI/alpine/intel-ucode.img /boot/efi/EFI/alpine/intel-ucode-old.img

# delete grub components
apk del grub grub-efi syslinux
find / | grep grub | xargs -I% rm -rf %

# cleanup chroot and its mounts
exit
umount -l /mnt/dev
umount -l /mnt/proc
umount -l /mnt/sys

# register kernel as bootloader in uefi interface (old kernel version)
UUID_ENCRYPTED_PARTITION=$(blkid | grep ${DEVNAME_PHYSICAL_DISK} | grep crypto_LUKS | sed -e "s/.* UUID=\"//" | sed -e "s/\" .*//")
UUID_DECRYPTED_LOGIGAL_ROOT_PARTITION=$(blkid | grep ${DEVNAME_DECRYPTED_VOLGROUP} | grep root | sed -e "s/.* UUID=\"//" | sed -e "s/\" .*//")

# previous kernel
KERNEL_ARGS_OLD="quiet root=UUID=${UUID_DECRYPTED_LOGICAL_ROOT_PARTITION} \
modules=sd-mod,usb-storage,ext4,cryptsetup,keymap,kms,lvm \
cryptroot=UUID=${UUID_ENCRYPTED_PARTITION} cryptdm=${DEVNAME_DECRYPTED_PARTITION} \
initrd=\EFI\alpine\intel-ucode-old.img \
initrd=\EFI\alpine\initramfs-lts-old"

efibootmgr --create --label "ALPINE LINUX (EFI STUB) - PREVIOUS" \
--disk /dev/${DEVNAME_PHYSICAL_DISK} --part 1 \
--loader /EFI/alpine/vmlinuz-lts-old \
--unicode "${KERNEL_ARGS_OLD}"

# latest kernel
KERNEL_ARGS="quiet root=UUID=${UUID_DECRYPTED_LOGICAL_ROOT_PARTITION} \
modules=sd-mod,usb-storage,ext4,cryptsetup,keymap,kms,lvm \
cryptroot=UUID=${UUID_ENCRYPTED_PARTITION} cryptdm=${DEVNAME_DECRYPTED_PARTITION} \
initrd=\EFI\alpine\intel-ucode.img \
initrd=\EFI\alpine\initramfs-lts"

efibootmgr --create --label "ALPINE LINUX (EFI STUB) - LATEST" \
--disk /dev/${DEVNAME_PHYSICAL_DISK} --part 1 \
--loader /EFI/alpine/vmlinuz-lts \
--unicode "${KERNEL_ARGS}"
```


## Post install

### Issue: no autologin in gdm
As i use lvm on luks and am the sole user of my laptop, i have no need to
type yet another password
```sh
# su without login shell, to keep the username of ${USER}
su
sed "/daemon/ a # Uncomment the two lines below to enable an automatic login\\
AutomaticLoginEnable=true\\
AutomaticLogin=${USER}" /etc/gdm/custom.conf > /etc/gdm/custom.conf
```

### Issue: nftables missing
This is the successor of iptables. Really miss [pf](https://why-openbsd.rocks/fact/pf/) of OpenBSD btw.
```sh
# install and enable nftables
su -l
apk add nftables
rc-update add nftables boot

# note: you still should review the default ruleset
```

### Issue: gnome-keyring is auto-launching on login
As i prefer to do authentication based stuff in console, i want to disable
the gnome keyring functionality otherwise leads to an popup, which is blocking
the desktop till you have entered the password. This is a usability drawback 
for me.
```sh
(cat /etc/xdg/autostart/gnome-keyring-ssh.desktop; echo Hidden=true) > ~/.config/autostart/gnome-keyring-ssh.desktop
(cat /etc/xdg/autostart/gnome-keyring-pkcs11.desktop; echo Hidden=true) > ~/.config/autostart/gnome-keyring-pkcs11.desktop
(cat /etc/xdg/autostart/gnome-keyring-secrets.desktop; echo Hidden=true) > ~/.config/autostart/gnome-keyring-secrets.desktop
```

### Issue: firefox(-esr) launches only in safe mode
As by suggestion of user ikke on [irc](https://wiki.alpinelinux.org/wiki/Alpine_Linux:IRC), you have to 
hide the safe-mode .desktop entries from gnome-shell, because it maps them
even to the regular firefox and firefox-esr executables. So in fact firefox
does not launch in safe mode at all if you click the safe mode icon explicity
```sh
# for firefox -esr
(cat /usr/share/applications/firefox-esr-safe.desktop; echo Hidden=true) > ~/.local/share/applications/firefox-esr-safe.desktop

# for firefox
(cat /usr/share/applications/firefox-safe.desktop; echo Hidden=true) > ~/.local/share/applications/firefox-safe.desktop
```

