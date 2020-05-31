## Ubuntu full disk encryption (FDE)

[https://help.ubuntu.com/community/Full_Disk_Encryption_Howto_2019](https://help.ubuntu.com/community/Full_Disk_Encryption_Howto_2019) - new ubuntu

2020-05-31 - GRUB does not support LUKS2. Use LUKS1(`--type luks1`) on GRUB patition.

### Copy current linux

```
rsync -avxHAX /mnt/linux /target
```

### Chroot

```
(
    cd /target
    mount -t proc /proc ./proc
    mount -o bind /sys  ./sys
    mount -o bind /dev  ./dev
    mount --make-rslave ./sys
    mount --make-rslave ./dev
    cd ..
)
```

### Health checks in new chroot linux

- /etc/fstab

```
cat /etc/fstab
/dev/mapper/ubuntu--vg-root               /       ext4    errors=remount-ro 0       1
/dev/mapper/LUKS_BOOT                     /boot   ext4    defaults          0       2
UUID=home-uuid                            /home   ext4    defaults          0       2
```

```
grep 'GRUB_ENABLE_CRYPTODISK=y' /etc/default/grub
GRUB_ENABLE_CRYPTODISK=y
```

```
## Keys for disks if needed
grep -e KEYFILE_PATTERN /etc/cryptsetup-initramfs/conf-hook| grep -v '^#'
KEYFILE_PATTERN=/etc/luks/*.keyfile
```

```
grep UMASK=0077 /etc/initramfs-tools/initramfs.conf
UMASK=0077
```

```
## If keys used
cat /etc/crypttab
# <target name> <source device>         <key file>      <options>
LUKS_BOOT UUID=uuid-dev-boot-uncrypt /etc/luks/boot_os.keyfile luks,discard
LUKS_ROOT UUID=uuid-dev-root-uncrypt /etc/luks/boot_os.keyfile luks,discard
```

```
menuentry 'Ubuntu' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple' {
        recordfail
        load_video
        gfxmode $linux_gfx_mode
        insmod gzio
        if [ x$grub_platform = xxen ]; then insmod xzio; insmod lzopio; fi
        insmod part_msdos
        insmod cryptodisk
        insmod luks
        insmod gcry_rijndael
        insmod gcry_rijndael
        insmod gcry_sha256
        insmod ext2
        cryptomount -u uuid_dev_boot_uncrypt_without_dashes
        set root='cryptouuid/uuid_dev_boot_uncrypt_without_dashes'
        if [ x$feature_platform_search_hint = xy ]; then
          search --no-floppy --fs-uuid --set=root --hint='cryptouuid/uuid_dev_boot_uncrypt_without_dashes'  uuid-dev-mapper-LUKS_BOOT
        else
          search --no-floppy --fs-uuid --set=root uuid-dev-mapper-LUKS_BOOT
        fi
        linux   /vmlinuz-4.15.0-101-generic root=/dev/mapper/ubuntu--vg-root ro net.ifnames=0 biosdevname=0 ipv6.disable=1 debug nosplash elevator=noop consoleblank=0 ipv6.disable=1
        initrd  /initrd.img-4.15.0-101-generic
}
```


### Troubleshooting

udevd errors - create new initramfs

GRUB not found self filesystem

```
error: no such device ...
error: unknown filesystem
```

