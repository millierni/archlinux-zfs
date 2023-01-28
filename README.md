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
- Import and destroy ZFS pool (if it contains pools)
  ```
  zpool import -d /dev/disk/by-id -f bpool
  zpool import -d /dev/disk/by-id -f rpool
  sleep 1
  zpool destroy bpool
  zpool destroy rpool
  ```
- Set `UUID0`, `UUID1` and `UUID2` variables
  - `ls -lF /dev/disk/by-id` and set the UUID of your own drive
  ```
  UUID0=
  ```
  ```
  UUID1=
  ```
  ```
  UUID2=
  ```
- Set `DISK0`, `DISK1` and `DISK2` variables
  ```
  DISK0=/dev/disk/by-id/$UUID0
  DISK1=/dev/disk/by-id/$UUID1
  DISK2=/dev/disk/by-id/$UUID2
  ```
- Erase `DISK0`, `DISK1` and `DISK2`
  ```
  #pacman -S --noconfirm gptfdisk # If it's not already installed
  sgdisk --zap-all $DISK0
  sgdisk --zap-all $DISK1
  sgdisk --zap-all $DISK2
  ```
- Create UEFI partition for booting
  ```
  sgdisk -n1:1M:+512M -t1:EF00 $DISK0
  sgdisk -n1:1M:+512M -t1:EF00 $DISK1
  sgdisk -n1:1M:+512M -t1:EF00 $DISK2
  udevadm settle; sleep 1
  mkfs.fat -F 32 -n EFI ${DISK0}-part1
  mkfs.fat -F 32 -n EFI ${DISK1}-part1
  mkfs.fat -F 32 -n EFI ${DISK2}-part1
  ```
- Create ZFS partitions for boot
  ```
  sgdisk -n2:0:+20G -t2:BF00 $DISK0
  sgdisk -n2:0:+20G -t2:BF00 $DISK1
  sgdisk -n2:0:+20G -t2:BF00 $DISK2
  sleep 1
  ```
- Create ZFS partitions for root
  ```
  sgdisk -n3:0:0 -t3:BF00 $DISK0
  sgdisk -n3:0:0 -t3:BF00 $DISK1
  sgdisk -n3:0:0 -t3:BF00 $DISK2
  sleep 1
  ```
- Create boot pool (bpool) with features incompatible for grub disabled  
  see step [2.4](https://github.com/openzfs/zfs/wiki/Debian-Buster-Root-on-ZFS#step-2-disk-formatting) for parameters compatible with GRUB
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
- Create root pool (use `ashift=13` for Enterprise SSD)
  ```
  RPOOL=rpool
  zpool create -o ashift=12 \
    -O acltype=posixacl -O canmount=off -O compression=lz4 \
    -O dnodesize=auto -O normalization=formD -O relatime=on -O xattr=sa \
    -O mountpoint=none -R /mnt \
    -f $RPOOL raidz1 ${DISK0}-part3 ${DISK1}-part3 ${DISK2}-part3
  ```
- Create containers for $RPOOL and $BPOOL
  ```
  zfs create -o mountpoint=none $BPOOL/BOOT
  zfs create -o mountpoint=none  $RPOOL/ROOT
  ```
- Create datasets for $RPOOL
  ```
  rm -rf /mnt
  mkdir -p /mnt
  zfs create -o mountpoint=/     $RPOOL/ROOT/archlinux
  zfs create -o mountpoint=none  $RPOOL/data
  zfs create -o mountpoint=/home $RPOOL/data/home
  zfs create -o mountpoint=/root $RPOOL/data/home/root
  zfs create -o mountpoint=/var/cache/pacman $RPOOL/data/paccache
  ```
- Create datasets for $BPOOL
  ```
  zfs create -o mountpoint=/boot $BPOOL/BOOT/default
  ```
- Unmount all filesystems and set new mountpoints
  ```
  zfs unmount -a # unmount all zfs filesystems
  zfs set mountpoint=/boot $BPOOL/BOOT/default
  zfs set mountpoint=/     $RPOOL/ROOT/archlinux
  zfs set mountpoint=/home $RPOOL/data/home
  zfs set mountpoint=/root $RPOOL/data/home/root
  zfs set mountpoint=/var/cache/pacman $RPOOL/data/paccache
  ```
- Export pools
  ```
  zpool export $BPOOL
  zpool export $RPOOL
  ```
- Re-import pools
  ```
  rm -rf /mnt
  zpool import -d /dev/disk/by-id -R /mnt $RPOOL
  zpool import -d /dev/disk/by-id -R /mnt $BPOOL
  ```
- Disable cache files
  ```
  zpool set cachefile=none $BPOOL
  zpool set cachefile=none $RPOOL
  ```
- Upgrade `bpool` and `rpool`
  ```
  zpool upgrade bpool
  zpool upgrade rpool
  ```
- Make efi mount points and mount them
  ```
  mkdir -p /mnt/{efi0,efi1,efi2}
  mount -t vfat ${DISK0}-part1 /mnt/efi0
  mount -t vfat ${DISK1}-part1 /mnt/efi1
  mount -t vfat ${DISK2}-part1 /mnt/efi2
  ```
- Double check mounting points
  ```
  findmnt | grep /mnt
  ```
- Use `pacstrap` to install base set of packages into `/mnt`
  ```
  pacstrap /mnt base vi mandoc grub efibootmgr mkinitcpio
  CompatibleVer=$(pacman -Si zfs-linux | grep 'Depends On' | sed "s|.*linux=||" | awk '{ print $1 }')
  pacstrap -U /mnt https://archive.archlinux.org/packages/l/linux/linux-${CompatibleVer}-x86_64.pkg.tar.zst
  pacstrap /mnt zfs-linux zfs-utils
  pacstrap /mnt linux-firmware intel-ucode amd-ucode
  ```
- Edit `/mnt/etc/mkinitcpio.conf` and change `HOOKS` line to be: `HOOKS=(base udev autodetect modconf block keyboard zfs filesystems)`
  ```
  sed -i.bak -E 's/^(HOOKS=.*)$/#\1\nHOOKS=\(base udev autodetect modconf block keyboard zfs filesystems\)/g' /mnt/etc/mkinitcpio.conf
  cat /mnt/etc/mkinitcpio.conf # verify it
  ```
- Copy `/etc/fstab` to new installation
  ```
  genfstab -U /mnt >> /mnt/etc/fstab  # generate new fstab
  ```
- Save the variables for use later (optional)
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
- Chroot into new installation
  ```
  arch-chroot /mnt /usr/bin/env DISK0=$DISK0 DISK1=$DISK1 DISK2=$DISK2 RPOOL=$RPOOL BPOOL=$BPOOL /bin/bash
  ```
- Set timezone
  ```
  rm -rf /etc/localtime
  ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
  hwclock --systohc --utc # generate /etc/adjtime and set HW RTC to UTC
  ```
- Generate locales
  ```
  sed -iE 's/^#en_US.UTF-8/en_US.UTF-8/g' /etc/locale.gen  # uncomment en_US.UTF-8
  locale-gen
  ```
- [Set locale systemwide](https://wiki.artixlinux.org/Main/Installation#Localization)
  ```
  export LANG=en_US.UTF-8
  export LC_COLLATE=C
  cat > /etc/locale.conf << EOF
  LANG=$LANG
  LC_COLLATE=$LC_COLLATE
  EOF
  ```
- Set hostname and edit `/etc/hosts` file  
  Change `{littlepony}` with your hostname
  ```
  HOSTNAME={littlepony}
  ```
  ```
  echo $HOSTNAME > /etc/hostname
  cat >> /etc/hosts << EOF
  127.0.0.1       localhost
  ::1             localhost
  127.0.1.1       $HOSTNAME
  EOF
  ```
- Install packages
  ```
  pacman -S --noconfirm ntp base-devel nano networkmanager dhcpcd iwd
  systemctl enable systemd-timesyncd
  systemctl enable ntpd
  systemctl enable dhcpcd NetworkManager
  ```
- Add a user  
  Change `{magicunicorn}` with your username
  ```
  USER={magicunicorn}
  ```
  ```
  useradd -m $USER
  passwd $USER
  cat >> /etc/sudoers << EOF
  
  ##
  ## $USER privilege specification
  ##
  $USER ALL=(ALL:ALL) ALL
  EOF
  ```
- Make initramfs
  ```
  mkinitcpio -P
  ```
- Set root password
  ```
  echo "root:1234" | chpasswd # you need to change root password with your password
  grep root /etc/shadow # should show: root:$6$.....
  ```
- Add 3 EFI partitions to `/etc/fstab`
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
- Create `/etc/default/grub`
  ```
  ZPOOL_VDEV_NAME_PATH=1 grub-probe /boot # should return "zfs"
  ```
- Modify `/etc/default/grub`  
  Add `zfs=rpool`: ref \[ [grub-error-sparse-file-not-allowed-fix](https://forum.manjaro.org/t/solved-grub-btrfs-error-sparse-file-not-allowed/70031) \]
  ```
  [[ -f /etc/default/grub.original ]] && cp /etc/default/grub.original /etc/default/grub
  cp /etc/default/grub /etc/default/grub.original
  
  # comment out GRUB_TIMEOUT_STYLE=hidden
  sed -i -E 's/(^GRUB_TIMEOUT_STYLE=hidden)/#\1/g' /etc/default/grub
  
  ## add GRUB_CMDLINE_LINUX="root=ZFS=$RPOOL/ROOT/default"
  # sed -i -E 's/(^GRUB_CMDLINE_LINUX=")/\1root=ZFS='"$RPOOL"'\/ROOT\/archlinux\/root /g' /etc/default/grub
  # add GRUB_CMDLINE_LINUX="zfs=$RPOOL" (info from mkinitcpio -H zfs)
  sed -i -E 's/^(GRUB_CMDLINE_LINUX=")(.*)$/\1zfs='"$RPOOL"' \2/g' /etc/default/grub
  
  # comment out any GRUB_SAVEDEFAULT and write GRUB_SAVEDEFAULT=false
  # this gets rid of "sparse file not supported" error when GRUB boots
  sed -i -E 's/^#?(GRUB_SAVEDEFAULT=.*)$/GRUB_SAVEDEFAULT=false\n#\1/g' /etc/default/grub
  
  # add GRUB_RECORDFAIL_TIMEOUT=2
  sed -i -E 's/(^GRUB_TIMEOUT=)[0-9]+$/\15\nGRUB_RECORDFAIL_TIMEOUT=2/g' /etc/default/grub
  
  # remove "quiet" from GRUB_CMDLINE_LINUX_DEFAULT
  sed -i -E 's/(^GRUB_CMDLINE_LINUX_DEFAULT=")(.*)(quiet)(.*)(")/\1\2\4\5/g' /etc/default/grub
  
  # remove spaces next to quotation marks on GRUB_CMDLINE_LINUX_DEFAULT
  sed -i -E 's/(^GRUB_CMDLINE_LINUX_DEFAULT=")([ \t]+)(.*)(")/\1\3\4/g' /etc/default/grub
  sed -i -E 's/(^GRUB_CMDLINE_LINUX_DEFAULT=")(.*)([ \t]+)(")/\1\2\4/g' /etc/default/grub
  ```
- Run grub-mkconfig and install grub
  ```
  ZPOOL_VDEV_NAME_PATH=1 grub-mkconfig -o /boot/grub/grub.cfg
  ZPOOL_VDEV_NAME_PATH=1 grub-install --target=x86_64-efi --efi-directory=/efi0 --bootloader-id=GRUB
  ```
- Sync the EFI partitions
  ```
  pacman -S --noconfirm rsync
  rsync -Rai --stats --human-readable --delete --verbose --progress /efi0/./ /efi1
  rsync -Rai --stats --human-readable --delete --verbose --progress /efi0/./ /efi2
  ```  
- Do efibootmgr on the other drives
  ```
  efibootmgr -c -g -d $DISK1 -p 1 -L "GRUB-1" -l '\EFI\GRUB\grubx64.efi'
  efibootmgr -c -g -d $DISK2 -p 1 -L "GRUB-2" -l '\EFI\GRUB\grubx64.efi'
  ```
- Double check `/etc/grub/grub.cfg` according to [this](https://wiki.archlinux.org/index.php/Install_Arch_Linux_on_ZFS#Booting_your_kernel_and_initrd_from_ZFS
- Start sshd and enable root login (add `PermitRootLogin yes` to `/etc/ssh/sshd_config`)
  ```
  pacman -S --noconfirm openssh
  sed -i -E 's/^(#PermitRootLogin.*)$/PermitRootLogin yes\n\1/g' /etc/ssh/sshd_config
  systemctl enable sshd
  ```
- Enable `zfs-import-cache.service`
  ```
  zpool set cachefile=/etc/zfs/zpool.cache $BPOOL 
  zpool set cachefile=/etc/zfs/zpool.cache $RPOOL
  systemctl enable zfs-import-cache.service
  ```
- Add aliases and prompt to `/etc/skel/.bashrc`
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
- Copy this `.bash_profile` and `.bashrc` to root
  ```
  [[ ! -f /root/.bash_profile ]] && cp /etc/skel/.bash_profile /root
  [[ ! -f /root/.bashrc ]]       && cp /etc/skel/.bashrc       /root
  ```
- Save installation environment variables
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
- Unmount filesystem
  ```
  umount -R /mnt
  zfs umount -a
  ```
- Export `bpool` and `rpool`
  ```
  zpool export $BPOOL
  zpool export $RPOOL
  ```
- Remove live media and prepare to reboot
  ```
  reboot
  ```
- If reboot into GRUB
  - Set root partition
  ```
  set root=(hd1,gpt2) # ls to search for the root partition
  ```
  - Load linux kernel
  ```
  linux /BOOT/default/@/vmlinuz-linux root=ZFS=rpool/ROOT/archlinux boot=ZFS=bpool/BOOT/default rw
  ```
  - Load the filesystem image for booting the kernel
  ```
  initrd /BOOT/default/@/initramfs-linux.img
  ```
  - Insert video module into the kernel
  ```
  insmod all_video
  ```
  - Start the OS
  ```
  boot
  ```
- If reboot into rootfs
  ```
  zpool import rpool -R /new_root
  exit
  ```
- After reboot, login as *root*
  - Import all the pools that are not listed with `zpool -list`
  ```
  zpool import -d /dev/disk/by-id -f bpool
  ```
  - Enable the automatic mounting for the ZFS pools
  ```
  systemctl enable zfs-import-cache # if it's not already enabled
  systemctl enable zfs-mount
  systemctl enable zfs-import.target
  systemctl enable zfs.target
  systemctl start zfs.target
  ```
  - Start ZFS support
  ```
  modprobe zfs
  ```
  - Generate a new GRUB configuration file
  ```
  grub-mkconfig -o /boot/grub/grub.cfg
  ```
  ```
  reboot  # reboot the OS
  ```
- Scrub the `rpool` and `bpool`
  ```
  zpool scrub rpool
  zpool scrub bpool
  ```
  Verify that there's no error
  ```
  zpool status -x -v
  ```
- Create the genesis snapshot
  ```
  zfs snapshot rpool/ROOT/archlinux@genesis
  zfs snapshot rpool/data@genesis
  zfs snapshot rpool/data/home@genesis
  zfs snapshot rpool/data/home/root@genesis
  zfs snapshot bpool/BOOT/default@genesis
  ```
  - Verify the snapshots
    ```
    zfs list -t snapshot
    ```
- Install desktop environment [List](https://wiki.archlinux.org/title/Desktop_environment)
  - Install graphics drivers
    - AMD
      ```
      pacman -S xf86-video-amdgpu mesa
      ```
    - NVIDIA
      ```
      pacman -S nvidia nvidia-utils
      ```
    - Intel
      ```
      pacman -S xf86-video-intel mesa
      ```
  - KDE Plasma desktop
  ```
  pacman -Syy
  ```
  ```
  pacman -S xorg plasma kde-applications plasma-wayland-session sddm
  ```
  ```
  systemctl enable sddm
  ```
  - Reboot the OS
  ```
  reboot
  ```
- Add ZFS and multilib repository and update the pacman database
  ```
  su
  ```
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
  ```
  ```
  cat >> /etc/pacman.conf << EOF

  [multilib]
  Include = /etc/pacman.d/mirrorlist
  EOF
  ```
  ```
  exit
  ```
  ```
  sudo pacman -Syy
  ```
- Scrub the `rpool` and `bpool`
  ```
  sudo zpool scrub rpool
  sudo zpool scrub bpool
  ```
  Verify that there's no error
  ```
  sudo zpool status -x -v
  ```
- Create the base snapshot
  ```
  sudo zfs snapshot rpool/ROOT/archlinux@base
  sudo zfs snapshot rpool/data@base
  sudo zfs snapshot rpool/data/home@base
  sudo zfs snapshot rpool/data/home/root@base
  sudo zfs snapshot bpool/BOOT/default@base
  ```
- Automate and schedule `ZFS scrub` on the pools and email any output
  - Create a gmail account, enable `2-Step verification`, create an [app passwords](https://myaccount.google.com/apppasswords)
  - Install `msmtp-mta`
    ```
    sudo pacman -S msmtp-mta
    ```
  - Create `msmtprc`
    ```
    sudo nano /etc/msmtprc
    ```
    - Add this to `msmtprc`  
      Replace the value of `user` and `from` with your gmail address  
      Replace `{gmail_app_password}` with your app password
      ```
      account default
      tls on
      auth on
      host smtp.gmail.com
      port 587
      user bot.magic.unicorn@gmail.com
      from bot.magic.unicorn@gmail.com
      password {gmail_app_password}
      ```
  - Change the permissions on `msmtprc`
    ```
    sudo chmod 600 /etc/msmtprc
    sudo chown $USER:$USER /etc/msmtprc
    ```
  - Try if it works  
    Replace the email with your email address
    ```
    echo "test" | msmtp magic@unicorn.com
    ```
  - Enable and start cronie deamon
    ```
    sudo systemctl enable cronie
    sudo systemctl start cronie
    ```
  - Edit `crontab`
    ```
    EDITOR=nano crontab -e
    ```
    - Add this
      ```
      MAILTO=magic@unicorn.com
      # zpool scrub every Monday 8 PM and send an email every Tuesday 8 AM 
      0 20 * * 1 /sbin/zpool scrub bpool
      0 20 * * 1 /sbin/zpool scrub rpool
      0 8 * * 2 /sbin/zpool status
      ```
- Create an alias to modify the `pacman command`
  - Create `.pacman-zfs-snapshot.sh`
    ```
    sudo nano /etc/.pacman-zfs-snapshot.sh
    ```
    - Add this to `.pacman-zfs-snapshot.sh`
      ```
      #!/bin/bash

      # Manage the arguments of the pacman command
      if [[ "$@" == *"-Syu"* ]] || [[ "$@" == *"-Syyu"* ]]; then
        nbUpdates="$(pacman -Qu | wc -l)"
        if (( nbUpdates > 1 )); then
          echo "There are $nbUpdates packages available to update..."
          pacman -Qu
          read -rp "Do you want to create snapshots before updating? (y/n) " answer
          if [[ "$answer" =~ ^[Yy]$ ]]; then
            # Create snapshots
            sudo zfs snapshot rpool/ROOT/archlinux@$(date +%Y-%m-%d-%H:%M:%S)
            echo "ZFS: snapshot of rpool/ROOT/archlinux   [created]"
            sudo zfs snapshot rpool/data@$(date +%Y-%m-%d-%H:%M:%S)
            echo "ZFS: snapshot of rpool/data             [created]"
            sudo zfs snapshot rpool/data/home@$(date +%Y-%m-%d-%H:%M:%S)
            echo "ZFS: snapshot of rpool/data/home        [created]"
            sudo zfs snapshot rpool/data/home/root@$(date +%Y-%m-%d-%H:%M:%S)
            echo "ZFS: snapshot of rpool/data/home/root   [created]"
            sudo zfs snapshot bpool/BOOT/default@$(date +%Y-%m-%d-%H:%M:%S)
            echo "ZFS: snapshot of bpool/BOOT/default     [created]"
          fi
          # Execute the command and ignore zfs-linux and linux
          sudo pacman $@ --ignore zfs-linux,linux
          packagesUpdated=true
        else
          echo "There is 0 package available to update..."
          sudo pacman $@ --ignore zfs-linux,linux
          packagesUpdated=true
        fi
      else
        sudo pacman $@
        packagesUpdated=false
      fi
      
      # Verify if there is an update available for zfs-linux and search for the compatible linux kernel

      # Check if packages have been updated and if there is an update available for zfs-linux
      if [[ "$packagesUpdated" = true ]] && pacman -Qu | grep -q 'zfs-linux'; then
        # Get the compatible version of the linux kernel from the zfs-linux package information
        linuxCompatibleVersion=$(pacman -Si zfs-linux | grep -oP "(?<= linux=)[^\s]+")
        currentLinuxVersion=$(uname -r)
        # Compare the current linux kernel version with the compatible version of the zfs-linux package
        if [[ "$currentLinuxVersion" != "$linuxCompatibleVersion" ]]; then
          updateLinuxKernel=true
        fi
      fi


      # Prompt the user for updating linux and zfs-linux to their last compatible version
      if [[ "$updateLinuxKernel" = true ]]; then
        read -rp "Do you want to update linux and zfs-linux? (y/n) " answer
        if [[ $answer =~ ^[Yy]$ ]]; then
          sudo pacman -S "linux=$linuxCompatibleVersion" "zfs-linux=$zfsLinux_availableVersion"
          echo "It is recommended to reboot the system"
        elif [[ $answer =~ ^[Nn]$ ]]; then
          exit 0
        else
          echo "Please answer Y (yes) or N (no)."
          exit 1
        fi
      fi

      ```
  - Change the permissions on `.pacman-zfs-snapshot.sh`
    ```
    sudo chmod +x /etc/.pacman-zfs-snapshot.sh
    ```
  - Add alias to `/etc/bash.bashrc`
    ```
    echo "alias sudo='sudo '" | sudo tee -a /etc/bash.bashrc
    echo "alias pacman='/etc/.pacman-zfs-snapshot.sh'" | sudo tee -a /etc/bash.bashrc
    ```
- Change `pulseaudio` by `pipewire` (optional)
  ```
  sudo pacman -S --needed pipewire
  sudo pacman -S --needed pipewire-media-session
  sudo pacman -S --needed pipewire-alsa
  sudo pacman -S --needed pipewire-jack
  sudo pacman -S --needed pipewire-zeroconf
  sleep 1
  sudo pacman -R pulseaudio-equalizer-ladspa
  sudo pacman -R pulseaudio-alsa
  sudo pacman -R gnome-bluetooth blueberry
  sudo pacman -R pulseaudio-bluetooth
  sudo pacman -R pulseaudio
  sleep 1
  sudo pacman -S --needed pipewire-pulse
  #sudo pacman -S --needed blueberry #If you want to install bluetooth
  ```
  ```
  sudo reboot
  ```
- Verify that the `Server Name` is running `PulseAudio (on PipeWire)`
  ```
  pactl info
  ```
- Scrub the `rpool` and `bpool`
  ```
  sudo zpool scrub rpool
  sudo zpool scrub bpool
  ```
  Verify that there's no error
  ```
  sudo zpool status -x -v
  ```
- Create snapshots
  ```
  sudo zfs snapshot rpool/ROOT/archlinux@`date +%Y-%m-%d-%H:%M:%S`
  sudo zfs snapshot rpool/data@`date +%Y-%m-%d-%H:%M:%S`
  sudo zfs snapshot rpool/data/home@`date +%Y-%m-%d-%H:%M:%S`
  sudo zfs snapshot rpool/data/home/root@`date +%Y-%m-%d-%H:%M:%S`
  sudo zfs snapshot bpool/BOOT/default@`date +%Y-%m-%d-%H:%M:%S`
  ```
- [Install packages](https://github.com/millierni/archlinux-packages)
