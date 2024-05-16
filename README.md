# alpine-linux-desktop-guide
A memorization help for myself to remember what i have done to setup my laptop workstation

Basically i run a partial disc entryption with unencrypted EFI and encrypted
root using EFI STUB. Desktop wise i use gnome-shell. I aim to make the
install reasonably secure

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
rc-update add seedrng boot
rc-update add acpid default
```

### Step seven: start powermanagement and random number generator in live session
```sh
rc-service seedrng start
rc-service acpid start
```

### Step eight: point hostname domain to local host ip
```sh
HOSTNAME=$(cat /etc/hostname)
sed -i -e "s/localhost /${HOSTNAME} localhost /" /etc/hosts
```

### Step nine: enable ntp
```sh
setup-ntp
```

### Step ten: initialize apk package manager
```sh
setup-apkrepos
# change mirror to https
sed -i -e "s/http/https/" /etc/apk/repositories

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
| 256M | ${FREE} |

EFI partition will hold bootloader, kernel and initramfs an is FAT32
Linux partition will be encrypted with LUKS later on
```sh
# overwrite first 100M with junk (could be sda instead of nvme0n1, check with lsblk)
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

cryptsetup open /dev/${DEVNAME_ENCRYPTED_PARTITION} ${DEVNAME_DECRYPTED_PARTITION}
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
As of now the `lsblk` command should yield to something like this:
```
nvme0n1                 259:0    0 931.5G  0 disk
├─nvme0n1p1             259:1    0   256M  0 part
└─nvme0n1p2             259:2    0 931.3G  0 part
  └─nvme0n1p2.dec       253:0    0 931.2G  0 crypt
    ├─nvme0n1p2.vg-swap 253:1    0     2G  0 lvm
    └─nvme0n1p2.vg-root 253:2    0 929.2G  0 lvm
```

### Step sixteen: install to disk
```sh
# mount /
mount -t ext4 /dev/mapper/${DEVNAME_DECRYPTED_VOLGROUP}-root /mnt/

# mount swap
swapon /dev/mapper/${DEVNAME_DECRYPTED_VOLGROUP}-swap

# mount the EFI partition inside /mnt/boot
mkdir -p /mnt/boot/efi
mount -t vfat -o fmask=0177,dmask=0077 /dev/${DEVNAME_EFI_PARTITION} /mnt/boot/efi/

# finally the setup
setup-disk -m sys /mnt/
```

### Step seventeen: fix swap missing in /mnt/etc/fstab
```sh
# search swap partition by UUID and add fstab entry for it
(blkid | grep swap | sed -e "s/.* UUID=\"/UUID=/" | sed -e "s/\" .*/    swap    swap    sw    0 0 /") >> /mnt/etc/fstab
# note: i am not sure if other partitions are by UUID or by path
#       all mounts without UUID should be replaced with UUID notion where
#       applicable

# delete cdrom and usbdisk entries (alpine bug - or feature :DD)
sed -i -e "/^\/dev\/cdrom/d" -e "/^\/dev/\/usbdisk/d" /mnt/etc/fstab
```

### Step eighteen: change initramfs modules to the ones needed
```sh
INITRAMFS_MODULES="base usb ext4 lvm nvme cryptsetup keymap nvme"
sed -i -e "s/features=\".*\"/features=\"${INITRAMFS_MODULES}\"/" /mnt/etc/mkinitfs/mkinitfs.conf
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
find / | grep syslinux | xargs -I% rm -rf %
find / | grep extlinux | xargs -I% rm -rf %

# add apparmor
apk add apparmor apparmor-utils


# IMPORTANT: set a root password
passwd


# cleanup chroot and its mounts
exit
umount -l /mnt/dev
umount -l /mnt/proc
umount -l /mnt/sys

# register kernel as bootloader in uefi interface (old kernel version)
UUID_ENCRYPTED_PARTITION=$(blkid | grep ${DEVNAME_PHYSICAL_DISK} | grep crypto_LUKS | sed -e "s/.* UUID=\"//" | sed -e "s/\" .*//")
UUID_DECRYPTED_LOGICAL_ROOT_PARTITION=$(blkid | grep ${DEVNAME_DECRYPTED_VOLGROUP} | grep root | sed -e "s/.* UUID=\"//" | sed -e "s/\" .*//")

# previous kernel
KERNEL_ARGS_OLD="quiet root=UUID=${UUID_DECRYPTED_LOGICAL_ROOT_PARTITION} \
modules=sd-mod,usb-storage,ext4,cryptsetup,keymap,kms,lvm \
lsm=capability,landlock,yama,apparmor \
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
lsm=capability,landlock,yama,apparmor \
cryptroot=UUID=${UUID_ENCRYPTED_PARTITION} cryptdm=${DEVNAME_DECRYPTED_PARTITION} \
initrd=\EFI\alpine\intel-ucode.img \
initrd=\EFI\alpine\initramfs-lts"

efibootmgr --create --label "ALPINE LINUX (EFI STUB) - LATEST" \
--disk /dev/${DEVNAME_PHYSICAL_DISK} --part 1 \
--loader /EFI/alpine/vmlinuz-lts \
--unicode "${KERNEL_ARGS}"
```


## Add gnome desktop

### Step one: Setup a user
```sh
setup-user
```

### Step two: Install gnome meta-package
```sh
setup-desktop gnome
```

### Step three: Add the GUI user to some groups
```sh
adduser ${USER} audio
adduser ${USER} video
# if plugdev is missing: addgroup plugdev
adduser ${USER} plugdev
```

### Step four: Enable NetworkManager with iwd
```sh
# install networkmanager
apk add netwotkmanager-wifi

# copy the config
echo "[main]
dhcp=internal
dns=none
#plugins=ifupdown,keyfile

#[ifupdown]
#managed=true

[device]
wifi.backend=iwd
# "yes" is already the default for scanning
wifi.scan-rand-mac-address=yes
# Generate a random MAC for each WiFi and associate the two permanently.
wifi.cloned-mac-address=stable

[connection]
ipv6.ip6-privacy=2" > /etc/NetworkManager/NetworkManager.conf

# stop traditional networking services
rc-service networking stop
rc-update del networking boot
rc-update del wpa_supplicant boot

# register NetworkManager and iwd
rc-update add iwd boot
rc-update add networkmanager boot

# delete config files of wpa_supplicant
rm -rf /etc/wpa_supplicant

# disable manual network configuration for every non loopback interface
# delete every line besides the first two (containing lo) in
# /etc/network/interfaces

```

## Fix inconveniences

### Issue: pam does allow weak ciphers in /etc/shadow by default
sha512 seems subobtimal, as Debian uses [yescrypt](https://www.debian.org/releases/bullseye/amd64/release-notes/ch-information.en.html#pam-default-password)
But musl does not support it (yet)
```sh
# fixed upstream
```
To apply the changes, all passwords have to be changed with `passwd`

### Issue: swap automounting is not enabled by default
```sh
su -l
rc-update add swap boot
exit
```

### Issue: swapping is not defensive
I want to keep the NVMe in good health, so the system should only
swap to disk when not inevitable.
```sh
su -l
echo "vm.swappiness = 1" > /etc/sysctl.d/swappiness.conf
exit
``` 

### Issue: EFI stub boots old kernel only, EFI/alpine is not updated automatically
As EFI Stub booting has (in my knowledge) no automatic kernel and initramfs updating
script, manual copying of the latest kernel and initramfs to the EFI partition is
necessary.
```sh
su -l
# backup old kernel and initramfs combo (has also a boot entry)
cp /boot/efi/EFI/alpine/vmlinuz-lts /boot/efi/EFI/alpine/vmlinuz-lts-old
cp /boot/efi/EFI/alpine/initramfs-lts /boot/efi/EFI/alpine/initramfs-lts-old

# copy latest kernel and initramfs combo to EFI
cp /boot/vmlinuz-lts /boot/efi/EFI/alpine/vmlinuz-lts
cp /boot/initramfs-lts /boot/efi/EFI/alpine/initramfs-lts
exit
```

### Issue: coredumps are enabled by default
coredumps can contain sensible in memory information, such as clear text passwords
```sh
su -l
echo "kernel.core_pattern=/dev/null" > /etc/sysctl.d/coredump.conf
exit
```


### Issue: openrc does not log anything
```sh
su -l
sed -i -e "s/^#rc_logger=.*/rc_logger=\"YES\"/" /etc/rc.conf
sed -i -e "s/^#rc_verbose=.*/rc_verbose=yes/" /etc/rc.conf
exit
```

### Issue: no autologin in gdm
As i use lvm on luks and am the sole user of my laptop, i have no need to
type yet another password
```sh
# su without login shell, to keep the username of ${USER}
su
sed -i "/daemon/ a # Uncomment the two lines below to enable an automatic login\\
AutomaticLoginEnable=true\\
AutomaticLogin=${USER}" /etc/gdm/custom.conf
exit
```

### Issue: nftables missing
This is the successor of iptables. Really miss [pf](https://why-openbsd.rocks/fact/pf/) of OpenBSD btw.
```sh
# install and enable nftables
su -l
apk add nftables
rc-update add nftables boot

# note: you still should review the default ruleset
exit
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

Note: The code above does not longer work. You have to fiddle with `/etc/pam.d/` files and comment out all `-auth optional pam_gnome_keyring.so` lines like so:
```sh
find /etc/pam.d/ -type f | xargs -I% sed -i -e "/pam_gnome_keyring.so/d" %
```
Just the above does not nuke this annoying service. To do this you have to remove execute bits from the following files:
```sh
chmod -x /usr/libexec/gcr-ssh-*
chmod -x /usr/bin/gnome-keyring*
```

### Issue: search only program names in gnome-shell searchbar
```sh
gsettings set org.gnome.desktop.search-providers disable-external true
```

### Issue: firefox(-esr) launches only in safe mode
As by suggestion of user ikke on [irc](https://wiki.alpinelinux.org/wiki/Alpine_Linux:IRC), you have to 
hide the safe-mode .desktop entries from gnome-shell, because it maps them
even to the regular firefox and firefox-esr executables. So in fact firefox
does not launch in safe mode at all if you don't click the safe mode icon
explicity
```sh
# for firefox -esr
(cat /usr/share/applications/firefox-esr-safe.desktop; echo Hidden=true) > ~/.local/share/applications/firefox-esr-safe.desktop

# for firefox
(cat /usr/share/applications/firefox-safe.desktop; echo Hidden=true) > ~/.local/share/applications/firefox-safe.desktop
```

### Issue: apk package manager installs or updates without preview
```sh
su -l
touch /etc/apk/interactive
exit
```

### Issue: gkt3 applications look a bit 'dated'
```
# testing repo should be active in /etc/apk/repositories
apk add adw-gtk3
```

### Issue: default Cantarell font doesn't look crisp on average DPI LCD
To reduce eyestrain i looked around to replace the default fonts. Apple provides
the excellent San-Francisco font pack. They can be applied easily.
```sh
mkdir /tmp/applefonts
cd /tmp/applefonts
curl -O https://devimages-cdn.apple.com/design/resources/download/SF-Pro.dmg
curl -O https://devimages-cdn.apple.com/design/resources/download/SF-Mono.dmg

# some unused ones
#curl -O https://devimages-cdn.apple.com/design/resources/download/NY.dmg
#curl -O https://devimages-cdn.apple.com/design/resources/download/SF-Compact.dmg

# extract the dmg
7z x SF-Pro.dmg
7z x 'SFProFonts/SF Pro Fonts.pkg'
7z x 'Payload~'
# and
7z x SF-Mono.dmg
7z x -aoa 'SFMonoFonts/SF Mono Fonts.pkg'
7z x -aoa 'Payload~'

# install it systemwide
su
cp Library/Fonts/*.otf /usr/share/fonts/OTF/
chmod 644 /usr/share/fonts/OTF/SF-*.otf
exit
```
After relogin:
```sh
# apply new fonts
gsettings set org.gnome.desktop.wm.preferences titlebar-font 'SF Pro Display 11'
gsettings set org.gnome.desktop.interface document-font-name 'SF Pro Display 11'
gsettings set org.gnome.desktop.interface font-name 'SF Pro Display 11'
gsettings set org.gnome.desktop.interface monospace-font-name 'SF Mono Medium 10'
gsettings set org.gnome.desktop.interface font-hinting 'none'
gsettings set org.gnome.desktop.interface font-antialiasing 'grayscale'
# for gnome builder
gsettings set org.gnome.builder.editor font-name 'SF Mono Medium 10'
gsettings set org.gnome.builder.terminal font-name 'SF Mono Medium 10'
# for gnome texteditior
gsettings set org.gnome.TextEditor use-system-font true
```

## Install some software

### Virtual machines with QEMU + KVM + Virt-Manager
```sh
# necessary packages
apk add virt-manager libvirt-daemon qemu-img qemu-system-x86_64 qemu-modules 

# add virtual network interface drivers to autostart on demand
su -l
echo "tun" > /etc/modules-load.d/tun.conf
exit

# non root management of virtual machines (use for personal computers only)
su -l
addgroup user libvirt
exit

# start libvirt daemon
rc-update add libvirtd boot
rc-service libvirtd start
```
