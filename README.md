# Installazione Gentoo ZFS

## 0.1 Settare la tastiera
```shell
loadkeys it
```

## 0.2 Settare le variabili
DISK=/dev/disk/by-id/nvme-INTEL_SSDPEKKW512G8_BTHH812205BA512D
MOUNT=/mnt/gentoo
ZPOOL=zroot
STAGE3=[stage3 mirror address]
```
## 0.3 Controllare che la rete funzioni
```shell
ping -c 2 gentoo.org
```
	Per le schede ethernet
	
```shell
dhcpcd enp3s0 (per il nome della scheda lo potete controllare con ip a)
```
	Per le schede WiFi
```shell
iwctl --passphrase PASSWPA2 station wlp2s0 connect YOURSSID
```
## 0.4 Aggiorna l'orologio di sistema 
```shell
timedatectl set-ntp true
```
## 1. Preparazione del disco
### 1.1 Cancellazione delle firme del disco
```shell
wipefs -af ${DISK}
```
### 1.2 Scrivere sulla tabella delle partizioni GPT
```shell
sgdisk -Zo ${DISK}
```

### 1.3.1 Usare Parted per creare la tabella delle partizioni
```shell
parted --script -a optimal ${DISK} \
unit mib \
mklabel gpt \
mkpart esp 1 1025 \
mkpart rootfs 1025 100% \
set 1 boot on
```

### 1.4 Formattazione della partizione di Boot
```shell
mkfs.vfat -F32 -n EFI ${DISK}-part1
```

### 1.5 Creazione di un pool zpool
```shell
zpool create -f \
-o ashift=12 \
-o cachefile=/etc/zfs/zpool.cache \
-O normalization=formD \
-O compression=zstd \
-O acltype=posixacl \
-O relatime=on \
-O atime=off \
-o spacemap_histogram=enabled \
-o large_blocks=enabled \
-o bookmarks=enabled \
-o embedded_data=enabled \
-o empty_bpobj=enabled \
-o filesystem_limits=enabled \
-O xattr=sa \
-m none -R ${MOUNT} ${ZPOOL} ${DISK}-part2
```

### 1.6 Creare un dataset
```shell
zfs create -o mountpoint=none -o canmount=off ${ZPOOL}/ROOT
zfs create -o mountpoint=none -o canmount=off ${ZPOOL}/data
zfs create -o mountpoint=/ ${ZPOOL}/ROOT/default
zfs create -o mountpoint=/home ${ZPOOL}/data/home
zfs create -o mountpoint=/var/db/repos ${ZPOOL}/data/ebuild
zfs create -o mountpoint=/var/cache/distfiles ${ZPOOL}/data/distfiles
zfs create -o mountpoint=/var/cache/ccache ${ZPOOL}/data/ccache
zfs create -V 32G -b 8192 -o logbias=throughput -o sync=always -o primarycache=metadata ${ZPOOL}/SWAP
```

### 1.7 zpool bootfs
```shell
zpool set bootfs=${ZPOOL}/ROOT/default ${ZPOOL}
```
### 1.8 Montaggio del filesystem (i filesystem zfs sono autogestiti e si montano nella directory specificata quando vengono creati)
```shell
mkdir -p ${MOUNT}/boot
mount ${DISK}-part1 ${MOUNT}/boot
mkswap /dev/zvol/${ZPOOL}/SWAP
swapon /dev/zvol/${ZPOOL}/SWAP
```
### 1.9 Copiare la cache zfs
```shell
mkdir -p ${ZPOOL}/etc/zfs
cp /etc/zfs/zpool.cache ${ZPOOL}/etc/zfs/zpool.cache
```

## 2. Installazione del sistemi di base
###2.1 Scaricare lo stage 3
```shell
cd ${MOUNT}
wget ${STAGE3}
tar xvJpf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```
###2.2 Montare Boot per Sistemi EFI
```shell
mkdir -p /mnt/boot/efi
mount -t vfat /dev/disk/by-id/usb...-part1 /mnt/boot/efi
```
###2.3 Generare Fstab
Generare fstab con il tool archlinux genfstab
```shell
genfstab -U -p /mnt | grep boot > /mnt/etc/fstab
```
Copiare il resolv.conf per l'impostazione corretta dns in gentoo chroot :
 ```shell
cp -L /etc/resolv.conf ${MOUNT}/etc/
```
### 2.3 make.conf
Modificare il file make.conf
```shell
vim /mnt/etc/portage/make.conf
```

```shell
# MAKEOPTS is equal to CPU CORE|Thread you have + 1,  
# If need see how munch CPU core you have, launch the command `nproc`
# -j2 : 2 is the output of nproc, set -j1 with only 2GB RAM !
# -l2 : 2 is the output of nproc
MAKEOPTS="-j2 -l2"

# --jobs2 : 3 is the output of nproc
# --load-average=2 is the output of nproc
COMMON_FLAGS="-O2 -march=native -pipe"
CHOST="x86_64-pc-linux-gnu"
EMERGE_DEFAULT_OPTS="--keep-going --jobs=2 --load-averoage=2 --autounmask-write=y --with-bdeps y --complete-graph y"
CPU_FLAGS_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt rdrand sha sse sse2 sse3 sse4_1 sse4_2 sse4a ssse3"
AUTO_CLEAN="yes"
ACCEPT_KEYWORDS="amd64"
ACCEPT_LICENSE="*"
GRUB_PLATFORMS="efi-64"
VIDEO_CARDS="i965 iris"
# Later for Xorg
INPUT_DEVICES="libinput" 

# Accept stable and unstable packages
ACCEPT_KEYWORDS="amd64"
```
### 2.4 Settare i CPU_FLAGS
```shell

emerge -av app-portage/cpuid2cpuflags
echo "*/* $(cpuid2cpuflags)" >> /etc/portage/package.use/00cpuflags
```
### 2.5 Configurazione emerge con la verifica gpg 
```shell
mkdir --parents /mnt/etc/portage/repos.conf
cp /mnt/usr/share/portage/config/repos.conf /mnt/etc/portage/repos.conf/gentoo.conf
```
Modificare /mnt/etc/portage/repos.conf/gentoo.conf, aggiungere linee che mancano:
```shell
[DEFAULT]
sync-allow-hardlinks = yes

[gentoo]
location = /var/db/repos/gentoo
sync-type = webrsync
#sync-type = rsync
```
### 2.6 Montate altri filesystem, se necessario
```shell
mount --rbind /dev dev
mount --rbind /proc proc
mount --rbind /sys sys
mount --make-rslave dev
mount --make-rslave proc
mount --make-rslave sys 
```
### 2.7 Entrare nell'ambiente chroot
```shell
chroot ${MOUNT} /bin/bash
source /etc/profile
export PS1="(chroot) $PS1"
```
### 2.8 Aggiornamento gentoo ebuild
```shell
emaint sync --auto
```

### 2.9 Cambiare i Mirror
```shell
emerge -av mirrorselect
mirrorselect -i -o >> /etc/portage/make.conf
```

### 2.10 Rieditare il file gentoo.conf
```shell
vim /etc/portage/repos.conf/gentoo.conf
```
modificare sync-type
```shell 
[gentoo]
#sync-type = webrsync
sync-type = rsync
```
Rialnciare update 
```shell
emaint sync --auto
```

## 3 Installare Gentoo
### 3.1 CPU FLAGS
elenco completo dei flag supportati può essere trovato con cat /proc/cpuinfo ma dobbiamo installare app-portage/cpuid2cpuflags per trovare quelle usate da gentoo: 
```shell
emerge -av app-portage/cpuid2cpuflags
echo "*/* $(cpuid2cpuflags)" >> /etc/portage/package.use/00cpuflags
```
### 3.2 Aggiungere directory e file a portage
```shell
 mkdir -p -v /etc/portage/package.use
 mkdir -p -v /etc/portage/package.accept_keywords
 mkdir -p -v /etc/portage/package.unmask
```
e i file
```shell
 touch /etc/portage/package.use/zzz_via_autounmask
 touch /etc/portage/package.accept_keywords/zzz_via_autounmask
 touch /etc/portage/package.unmask/zzz_via_autounmask
 ```
 
 ### 3.3 CFLAGS e altre Opzioni
 Troovare architettura CPU
```shell
gcc -c -Q -march=native --help=target | grep march
```
aggiungere ai common flag anche queste 
-fstack-protector-strong
-Wl,-z,relro
-Wl,-z,now

### 3.4 Ricompilare Il sistema
```shell
source /etc/profile && env-update 
emerge --ask --update --deep --newuse --with-bdeps=y @world 
```
una volta finito eliminare i pacchetti vecchi con
```shell
emerge -av --depclean
eclean -d distfiles
```
### 3.5 Gentoolkit
```shell
emerge -av gentoolkit
```
disabilitare IPV5
```shell
euse -D ipv
```
### 3.6 Selezionare il profilo
```shell
eselect profile list
eselect profile set 5
```

### 3.7 Configurazione del Kernel
```shell
echo "sys-kernel/gentoo-kernel-bin -initramfs" > /etc/portage/package.use/gentoo-kernel-bin
echo "sys-fs/zfs dist-kernel" > /etc/portage/package.use/zfs
echo "sys-fs/zfs-kmod dist-kernel" >> /etc/portage/package.use/zfs
echo "sys-boot/grub libzfs" > /etc/portage/package.use/grub
echo "app-text/ghostscript-gpl -l10n_zh-CN" > /etc/portage/package.use/ghostscript-gpl
echo "sys-fs/zfs ~amd64" > /etc/portage/package.accept_keywords/zfs
echo "=sys-fs/zfs-9999 **" >> /etc/portage/package.accept_keywords/zfs
echo "sys-fs/zfs-kmod ~amd64" >> /etc/portage/package.accept_keywords/zfs
echo "=sys-fs/zfs-kmod-9999 **" >> /etc/portage/package.accept_keywords/zfs
echo "options zfs zfs_arc_min=268435456" > /etc/modprobe.d/zfs.conf
echo "options zfs zfs_arc_max=536870912" >> /etc/modprobe.d/zfs.conf
```
```shell
emerge -av sys-kernel/linux-firmware sys-kernel/genkernel sys-kernel/gentoo-kernel-bin zfs zfs-kmod
```
Aggiungi queste righe al /etc/genkernel.conf
```shell
# cat >> /etc/genkernel.conf
BOOTLOADER="no" # add grub2 for BIOS
INSTALL="yes"
MENUCONFIG="no"
CLEAN="yes"
KEYMAP="yes"
SAVE_CONFIG="yes"
MOUNTBOOT="no"
MRPROPER="no"
ZFS="yes"
MODULEREBUILD="yes"
```
```shell
rm -rf /etc/hostid && zgenhostid
vi /etc/genkernel.conf
genkernel initramfs --compress-initramfs --kernel-config=/usr/src/linux/.config --makeopts=-j$(nproc)
```
### 3.8 Installazione del BOOTLOADER
Aggiungi il supporto Uefi per systemd
```shell
euse -p sys-apps/systemd -E gnuefi
emerge -av sys-apps/systemd efivar
```
Copiare il kernel e initram in esp
```shell
cp /boot/vmlinuz* /efi/vmlinuz
cp /boot/initramfs-*.img /efi/initramfs
```
Creare loader per Gentoo
```shell
# cat > /efi/loader/loader.conf
default gentoo
timeout 3
editor 0
```
Aggiungere le voci corrispondenti
```shell
# cat > /efi/loader/entries/gentoo.conf
title Gentoo Linux
linux /vmlinuz
initrd /initramfs
options root=ZFS=rpool/ROOT/gentoo init=/usr/lib/systemd/systemd dozfs keymap=it
```

installare il BOOTLOADER
```shell
bootctl --path /efi install
bootctl --path /efi update
```shell

## 4. Configurazione di Sistema
### 4.1 Configurazione della localizzazione
```shell
eselect locale list | grep -i en_us
eselect locale set 253
loadkey it
echo "KEYMAP=it" > /etc/vconsole.conf
ln -sf /usr/share/zoneinfo/Europe/Rome /etc/localtime
hwclock --systohc
env-update && source /etc/profile && export PS1="(chroot) $PS1"
```
### 4.2 Configurare fstab (qui usiamo genfstab, uno strumento di archlinux, per generare automaticamente fstab).
```shell
git clone https://github.com/26hz/install-tools.git
cd install-tools
./genfstab -U -p / >> /etc/fstab
```
controllare la fat partition /boot/efi
```shell
vim /etc/fstab
UUID=<UUID efi partition> /boot/efi vfat noauto,rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8,errors=remount-ro  0 2
```
## 5. Software utili
```shell
emerge -av doas cronie eix zsh lsd gentoolkit dosfstools parted neofetch ntfs3g bpytop iwd dhcpcd

```
## 6. Abilitare i servizi zfs con OpenRC
```shell
systemctl enable zfs.target
systemctl enable zfs-import-cache
systemctl enable zfs-mount
systemctl enable zfs-import.target
```
## 7. Nuovo utente
```shell
useradd -mG users,wheel,portage,usb,input,audio,video,sys,adm,tty,disk,lp,mem,news,console,cdrom,sshd,kvm,render,lpadmin,cron,crontab -s /bin/zsh matteo
passwd matteo
echo "permit keepenv nopass :wheel" > /etc/doas.conf
```
## 8. Creare una nuova istantanea del sistema con zfs snapshot
```shell
zfs snapshot zroot/ROOT/default@install
```
## 9. Riavviare
```shell
exit
umount /mnt/gentoo/boot
umount -Rl /mnt/gentoo/{dev,proc,sys,run,}
swapoff /dev/zvol/zroot/SWAP
zfs umount -a
zpool export -f zroot
reboot
```
## 10 Abilitare lo scrubbing ZFS 
```shell
vim /etc/systemd/system/zfs-scrub@.service
```
```shell
[Unit]
Description=Run scrub for ZFS pool %I

[Service]
Type=simple
ExecStart=/sbin/zpool scrub %i
```
```shell

Schedulare scrub systemd
vim /etc/systemd/system/zfs-scrub@.timer
```
```shell
[Unit]
Description=Run weekly scrub for ZFS pool %I

[Timer]
OnCalendar=Wed *-*-* 15:00:00

[Install]
WantedBy=timers.target
```
```shell
Abilitare 
systemctl enable zfs-scrub@zfsforninja.timer
systemctl start zfs-scrub@zfsforninja.timer
```
