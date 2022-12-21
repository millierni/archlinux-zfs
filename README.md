# Arch Linux - ZFS
## ARCH LINUX NEW INSTALL COMMANDS [ZFS (Home, OS), FAT32 (EFI)] - UEFI



References:\[ [john_ransden-arch on ZFS](https://ramsdenj.com/2016/06/23/arch-linux-on-zfs-part-2-installation.html#efi-bootloader) | [arch-systemd-boot-wiki](https://wiki.archlinux.org/index.php/Systemd-boot) \]


- Download the ISO and create a bootable USB with an Arch Linux distro
  + [EndeavourOS](https://endeavouros.com/latest-release/)
  
- Boot on the USB
- Add ZFS repo and update the software database
  ```
  curl -L https://archzfs.com/archzfs.gpg |  pacman-key -a -
  pacman-key --lsign-key $(curl -L https://git.io/JsfVS)
  curl -L https://git.io/Jsfw2 > /etc/pacman.d/mirrorlist-archzfs
  tee -a /etc/pacman.conf <<- 'EOF'

  #[archzfs-testing]
  #Include = /etc/pacman.d/mirrorlist-archzfs

  [archzfs]
  Include = /etc/pacman.d/mirrorlist-archzfs
  EOF
  pacman -Syy
  ```
- Install keysrings and create trust database and refresh keys
  ```
  pacman -S --noconfirm archlinux-keyring
  pacman-key --init
  pacman-key --populate archlinux
  pacman-key --refresh-keys
  ```
- Install and load ZFS
  ```
  curl -s https://raw.githubusercontent.com/eoli3n/archiso-zfs/master/init | bash
  ```
- Import and destroy ZFS pool (if it's contains pools)
  ```
  zpool import -d /dev/disk/by-id -f bpool
  zpool import -d /dev/disk/by-id -f rpool
  zpool destroy bpool
  zpool destroy rpool
  ```
- Set `DISK0`, `DISK1` and `DISK2` variables and erase them
  - `ls -lF /dev/disk/by-id` and set the UUID of your own hard disks
  ```
  DISK0=/dev/disk/by-id/{UUID#0} # Change UUID with the associated drive
  DISK1=/dev/disk/by-id/{UUID#1} # Change UUID with the associated drive
  DISK2=/dev/disk/by-id/{UUID#2} # Change UUID with the associated drive
  
  #pacman -S --noconfirm gptfdisk # If it's not already installed
  sgdisk --zap-all $DISK0
  sgdisk --zap-all $DISK1
  sgdisk --zap-all $DISK2
  ```
- create UEFI partition for booting:
  ```
  sgdisk -n1:1M:+512M -t1:EF00 $DISK0
  sgdisk -n1:1M:+512M -t1:EF00 $DISK1
  sgdisk -n1:1M:+512M -t1:EF00 $DISK2
  udevadm settle; sleep 1
  mkfs.fat -F 32 -n EFI ${DISK0}-part1
  mkfs.fat -F 32 -n EFI ${DISK1}-part1
  mkfs.fat -F 32 -n EFI ${DISK2}-part1
  ```
- create ZFS partitions for boot:
  ```
  sgdisk -n2:0:+20G -t2:BF00 $DISK0
  sgdisk -n2:0:+20G -t2:BF00 $DISK1
  sgdisk -n2:0:+20G -t2:BF00 $DISK2
  sleep 1
  ```
- create ZFS partitions for root:
  ```
  sgdisk -n3:0:0 -t3:BF00 $DISK0
  sgdisk -n3:0:0 -t3:BF00 $DISK1
  sgdisk -n3:0:0 -t3:BF00 $DISK2
  sleep 1
  ```
- create boot pool (bpool) with features incompatible for grub disabled. (see step [2.4](https://github.com/openzfs/zfs/wiki/Debian-Buster-Root-on-ZFS#step-2-disk-formatting)) for parameters compatible with GRUB:
  ```
  BPOOL=bpool
  zpool create -o ashift=12 -d \
    -o feature@async_destroy=enabled \
    -o feature@bookmarks=enabled \
    -o feature@embedded_data=enabled \
    -o feature@empty_bpobj=enabled \
    -o feature@enabled_txg=enabled \
    -o feature@extensible_dataset=enabled \
    -o feature@filesystem_limits=enabled \
    -o feature@hole_birth=enabled \
    -o feature@large_blocks=enabled \
    -o feature@lz4_compress=enabled \
    -o feature@spacemap_histogram=enabled \
    -o feature@userobj_accounting=enabled \
    -o feature@zpool_checkpoint=enabled \
    -o feature@spacemap_v2=enabled \
    -o feature@project_quota=enabled \
    -o feature@resilver_defer=enabled \
    -o feature@allocation_classes=enabled \
    -O acltype=posixacl -O canmount=off -O compression=lz4 -O devices=off \
    -O normalization=formD -O relatime=on -O xattr=sa \
    -O mountpoint=none -R /mnt -f $BPOOL raidz1 ${DISK0}-part2 ${DISK1}-part2 ${DISK2}-part2
  ```
- create root pool (use `ashift=13` for Enterprise SSD):
  ```
  RPOOL=rpool
  zpool create -o ashift=12 \
    -O acltype=posixacl -O canmount=off -O compression=lz4 \
    -O dnodesize=auto -O normalization=formD -O relatime=on -O xattr=sa \
    -O mountpoint=none -R /mnt \
    -f $RPOOL raidz1 ${DISK0}-part3 ${DISK1}-part3 ${DISK2}-part3
  ```
- create containers for $RPOOL and $BPOOL
  ```
  zfs create -o mountpoint=none $BPOOL/BOOT
  zfs create -o mountpoint=none  $RPOOL/ROOT
  ```
- create datasets for $RPOOL:
  ```
  rm -rf /mnt
  mkdir -p /mnt
  zfs create -o mountpoint=/     $RPOOL/ROOT/os
  zfs create -o mountpoint=none  $RPOOL/data
  zfs create -o mountpoint=/home $RPOOL/data/home
  zfs create -o mountpoint=/root $RPOOL/data/home/root
  zfs create -o mountpoint=/var/cache/pacman $RPOOL/data/paccache
  ```
- create datasets for $BPOOL:
  ```
  zfs create -o mountpoint=/boot $BPOOL/BOOT/default
  ```
- unmount all filesystems and set new mountpoints:
  ```
  zfs unmount -a # unmount all zfs filesystems
  zfs set mountpoint=/boot $BPOOL/BOOT/default
  zfs set mountpoint=/     $RPOOL/ROOT/os
  zfs set mountpoint=/home $RPOOL/data/home
  zfs set mountpoint=/root $RPOOL/data/home/root
  zfs set mountpoint=/var/cache/pacman $RPOOL/data/paccache
  ```
- Export pools:
  ```
  zpool export $BPOOL
  zpool export $RPOOL
  ```
- Re-import pools:
  ```
  rm -rf /mnt
  zpool import -d /dev/disk/by-id -R /mnt $RPOOL
  zpool import -d /dev/disk/by-id -R /mnt $BPOOL
  ```
- Disable cache files:
  ```
  zpool set cachefile=none $BPOOL
  zpool set cachefile=none $RPOOL
  ```
- make efi mount points and mount them:
  ```
  mkdir -p /mnt/{efi0,efi1,efi2}
  mount -t vfat ${DISK0}-part1 /mnt/efi0
  mount -t vfat ${DISK1}-part1 /mnt/efi1
  mount -t vfat ${DISK2}-part1 /mnt/efi2
  ```
> - edit `/etc/fstab` to include the above 4 pools from $RPOOL:
>   ```
>   echo "# generated fstab
>   # <file system>                <dir>              <type>    <options>                <dump> <pass>
>   $RPOOL/ROOT/os/root         /                  zfs       defaults,noatime          0      0
>   $RPOOL/ROOT/os/paccache     /var/cache/pacman  zfs       defaults,noatime          0      0
>   $RPOOL/data/home                 /home              zfs       defaults,noatime          0      0
>   $RPOOL/data/home/root            /root              zfs       defaults,noatime          0      0" \
>   > /etc/fstab
>   ```
> - Set the bootfs property on the root filesystem. This command will only be successful when `dnodesize` property on `$BPOOL/BOOT/default` is set to `legacy` \[ [ref](https://github.com/zfsonlinux/zfs/issues/8538) \]
>   ```
>   zfs set dnodesize=legacy $BPOOL/BOOT/default
>   zpool set bootfs=$BPOOL/BOOT/default $BPOOL
>   ```
- double check work:
  ```
  findmnt | grep /mnt # shows /mnt, /mnt/home, /mnt/root, /mnt/var/cache/pacman, /mnt/boot, /mnt/efi0, /mnt/efi1, /mnt/efi2 as mounted
  ```
- use `pacstrap` to install base set of packages into `/mnt`
  ```
  pacstrap /mnt base vi mandoc grub efibootmgr mkinitcpio
  CompatibleVer=$(pacman -Si zfs-linux | grep 'Depends On' | sed "s|.*linux=||" | awk '{ print $1 }')
  pacstrap -U /mnt https://archive.archlinux.org/packages/l/linux/linux-${CompatibleVer}-x86_64.pkg.tar.zst
  pacstrap /mnt zfs-linux zfs-utils
  pacstrap /mnt linux-firmware intel-ucode amd-ucode
  ```
- edit `/mnt/etc/mkinitcpio.conf` and change `HOOKS` line to be: `HOOKS=(base udev autodetect modconf block keyboard zfs filesystems)`
  ```
  sed -i.bak -E 's/^(HOOKS=.*)$/#\1\nHOOKS=\(base udev autodetect modconf block keyboard zfs filesystems\)/g' /mnt/etc/mkinitcpio.conf
  cat /mnt/etc/mkinitcpio.conf # verify it
  ```
- copy `/etc/fstab` to new installation:
  ```
  # cp /etc/fstab /mnt/etc/fstab.old    # keep old fstab
  genfstab -U /mnt >> /mnt/etc/fstab  # generate new fstab
  ```
> - since we mounted `/var/cache/pacman` and `/home` via ZFS (non-legacy), we need to remove these entries from the new `/mnt/etc/fstab`
>   ```
>   sed -i -E 's/^('"$RPOOL"'\/data.*)$/#\1/g; s/^('"$RPOOL"'\/ROOT\/os\/paccache.*)/#\1/g' /mnt/etc/fstab
>   ```
- save the variables for use later (optional):
  ```
  cat > /mnt/root/vars.sh << EOF
  DISK0=$DISK0
  DISK1=$DISK1
  DISK2=$DISK2
  RPOOL=$RPOOL
  BPOOL=$BPOOL
  EOF
  ```
- Reload systemd
  ```
  systemctl daemon-reload
  ```
- chroot into new installation
  ```
  arch-chroot /mnt /usr/bin/env DISK0=$DISK0 DISK1=$DISK1 DISK2=$DISK2 RPOOL=$RPOOL BPOOL=$BPOOL /bin/bash
  ```
- set timezone:
  ```
  rm -rf /etc/localtime
  ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
  hwclock --systohc --utc # generate /etc/adjtime and set HW RTC to UTC
  ```
- generate locales:
  ```
  sed -iE 's/^#en_US.UTF-8/en_US.UTF-8/g' /etc/locale.gen  # uncomment en_US.UTF-8
  locale-gen
  ```
- [set locale systemwide](https://wiki.artixlinux.org/Main/Installation#Localization):
  ```
  export LANG=en_US.UTF-8
  export LC_COLLATE=C
  cat > /etc/locale.conf << EOF
  LANG=$LANG
  LC_COLLATE=$LC_COLLATE
  EOF
  ```
- set hostname and edit `/etc/hosts` file:
  ```
  # call it whatever you want
  HOSTNAME=littlepony
  echo $HOSTNAME > /etc/hostname
  cat >> /etc/hosts << EOF
  127.0.0.1       localhost
  ::1             localhost
  127.0.1.1       $HOSTNAME
  EOF
  ```
  -Add a user
  ```
  USER=magicunicorn
  useradd -m $USER
  passwd $USER
  chown $USER:$USER /home/$USER
  ```
- Install packages:
  ```
  pacman -S ntp nano networkmanager dhcpcd iwd
  systemctl enable systemd-timesyncd
  systemctl enable ntpd
  systemctl enable dhcpcd NetworkManager
  ```
- make initramfs:
  ```
  mkinitcpio -P
  ```
- set root password:
  ```
  echo "root:abc" | chpasswd # change root password to `abc`
  grep root /etc/shadow # should show: root:$6$.....
  ```
- add 3 EFI partitions to `/etc/fstab`
  ```
  # comment everything on /etc/fstab
  sed -i -E 's/^([^#].*)$/# \1/g' /etc/fstab
  
  echo PARTUUID=$(blkid -s PARTUUID -o value ${DISK0}-part1) \
    /efi0 vfat nofail,x-systemd.device-timeout=1 0 1 >> /etc/fstab
  echo PARTUUID=$(blkid -s PARTUUID -o value ${DISK1}-part1) \
    /efi1 vfat nofail,x-systemd.device-timeout=1 0 1 >> /etc/fstab
  echo PARTUUID=$(blkid -s PARTUUID -o value ${DISK2}-part1) \
    /efi2 vfat nofail,x-systemd.device-timeout=1 0 1 >> /etc/fstab
  ```
- install grub
  ```
  pacman -S --noconfirm grub efibootmgr
  ```
- create `/etc/default/grub`
  ```
  ZPOOL_VDEV_NAME_PATH=1 grub-probe /boot # should return "zfs"
  ```
- modify `/etc/default/grub` - add `zfs=rpool`: ref \[ [grub-error-sparse-file-not-allowed-fix](https://forum.manjaro.org/t/solved-grub-btrfs-error-sparse-file-not-allowed/70031) \]
  ```
  [[ -f /etc/default/grub.original ]] && cp /etc/default/grub.original /etc/default/grub
  cp /etc/default/grub /etc/default/grub.original
  
  # comment out GRUB_TIMEOUT_STYLE=hidden
  sed -i -E 's/(^GRUB_TIMEOUT_STYLE=hidden)/#\1/g' /etc/default/grub
  
  ## add GRUB_CMDLINE_LINUX="root=ZFS=$RPOOL/ROOT/default"
  # sed -i -E 's/(^GRUB_CMDLINE_LINUX=")/\1root=ZFS='"$RPOOL"'\/ROOT\/os\/root /g' /etc/default/grub
  # add GRUB_CMDLINE_LINUX="zfs=$RPOOL" (info from mkinitcpio -H zfs)
  sed -i -E 's/^(GRUB_CMDLINE_LINUX=")(.*)$/\1zfs='"$RPOOL"' \2/g' /etc/default/grub
  
  # comment out any GRUB_SAVEDEFAULT and write GRUB_SAVEDEFAULT=false
  # this gets rid of "sparse file not supported" error when GRUB boots
  sed -i -E 's/^#?(GRUB_SAVEDEFAULT=.*)$/GRUB_SAVEDEFAULT=false\n#\1/g' /etc/default/grub
  
  # add GRUB_RECORDFAIL_TIMEOUT=5
  sed -i -E 's/(^GRUB_TIMEOUT=)[0-9]+$/\15\nGRUB_RECORDFAIL_TIMEOUT=5/g' /etc/default/grub
  
  # remove "quiet" from GRUB_CMDLINE_LINUX_DEFAULT
  sed -i -E 's/(^GRUB_CMDLINE_LINUX_DEFAULT=")(.*)(quiet)(.*)(")/\1\2\4\5/g' /etc/default/grub
  
  # remove spaces next to quotation marks on GRUB_CMDLINE_LINUX_DEFAULT
  sed -i -E 's/(^GRUB_CMDLINE_LINUX_DEFAULT=")([ \t]+)(.*)(")/\1\3\4/g' /etc/default/grub
  sed -i -E 's/(^GRUB_CMDLINE_LINUX_DEFAULT=")(.*)([ \t]+)(")/\1\2\4/g' /etc/default/grub
  ```
- run grub-mkconfig and install grub:
  ```
  ZPOOL_VDEV_NAME_PATH=1 grub-mkconfig -o /boot/grub/grub.cfg
  ZPOOL_VDEV_NAME_PATH=1 grub-install --target=x86_64-efi --efi-directory=/efi0 --bootloader-id=GRUB
  ```
- sync the two EFI partitions:
  ```
  pacman -S --noconfirm rsync
  rsync -Rai --stats --human-readable --delete --verbose --progress /efi0/./ /efi1
  rsync -Rai --stats --human-readable --delete --verbose --progress /efi0/./ /efi2
  ```
- do efibootmgr on disk2
  ```
  efibootmgr -c -g -d $DISK1 -p 1 -L "GRUB-1" -l '\EFI\GRUB\grubx64.efi'
  efibootmgr -c -g -d $DISK2 -p 1 -L "GRUB-2" -l '\EFI\GRUB\grubx64.efi'
  ```
- double check `/etc/grub/grub.cfg` according to [this](https://wiki.archlinux.org/index.php/Install_Arch_Linux_on_ZFS#Booting_your_kernel_and_initrd_from_ZFS
- start sshd and enable root login (add `PermitRootLogin yes` to `/etc/ssh/sshd_config`):
  ```
  pacman -S --noconfirm openssh
  sed -i -E 's/^(#PermitRootLogin.*)$/PermitRootLogin yes\n\1/g' /etc/ssh/sshd_config
  systemctl enable sshd
  ```
> - comment out all the `zfs` from `/etc/fstab`
>  ```
>  cp /etc/fstab /etc/fstab.original
>  sed -i -E 's/^('"$RPOOL"'.*)$/# \1/g' /etc/fstab
>  sed -i -E 's/^('"$BPOOL"'.*)$/# \1/g' /etc/fstab
>  ```
- enable `zfs-import-cache.service`
  ```
  zpool set cachefile=/etc/zfs/zpool.cache $BPOOL 
  zpool set cachefile=/etc/zfs/zpool.cache $RPOOL
  systemctl enable zfs-import-cache.service
  ```
- add aliases and prompt to `/etc/skel/.bashrc`:
  ```
  cat >> /etc/skel/.bashrc << EOF
  
  if [[ \$USER != "root" ]]; then
    PS1="\[\e[01;32m\]\u@\h\[\e[01;34m\] \w \\\\$\[\e[0m\] "
  else
    PS1="\[\e[01;31m\]\u@\h\[\e[01;34m\] \w \\\\$\[\e[0m\] "
  fi
  alias egrep='egrep --color=auto'
  alias fgrep='fgrep --color=auto'
  alias grep='grep --color=auto'
  alias l='ls -CF'
  alias la='ls -A'
  alias ll='ls -alF'
  alias ls='ls --color=auto'
  EOF
  ```
- copy this `.bash_profile` and `.bashrc` to root:
  ```
  [[ ! -f /root/.bash_profile ]] && cp /etc/skel/.bash_profile /root
  [[ ! -f /root/.bashrc ]]       && cp /etc/skel/.bashrc       /root
  ```
- save installation environment variables:
  ```
  cat > ~/vars.sh << EOF
  # installation environment
  DISK0=$DISK0
  DISK1=$DISK1
  DISK2=$DISK2
  RPOOL=$RPOOL
  BPOOL=$BPOOL
  EOF
  ```
- Exit chroot
  ```
  exit
  ```
- unmount filesystem
  ```
  umount -R /mnt
  zfs umount -a
  ```
- export them:
  ```
  zpool export $RPOOL
  zpool export $BPOOL
  ```
- remove live media and prepare to reboot
  ```
  reboot
  ```
- after reboot, login as *root*
> - make sure `zfs-import-cache.service` can find zpool cache.
>   ```
>   zpool set cachefile=/etc/zfs/zpool.cache zroot # <-- zroot here is our $ZROOT
>   ```
- edit `/etc/pacman.conf` and scroll to `#IgnorePkg` and add:
  ```
  IgnorePkg = linux54-zfs  # so this package won't be updated
  ```
- add more packages:
  ```
  pacman -S --noconfirm bash-completion pv
  ```
  
- install desktop environment
  ```
  https://wiki.manjaro.org/index.php/Install_Desktop_Environments#Overview  
  ```
