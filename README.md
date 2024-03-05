# live_debian


#!/bin/bash

export WORK_DIR="jessie-chroot"
rm -rf ${WORK_DIR}
debootstrap --arch=i386 --variant=minbase --include=linux-image-686,ifupdown,isc-dhcp-client,openssh-server,less,nano,debootstrap,systemd-sysv,curl,wget oldoldstable ${WORK_DIR} http://ftp.ca.debian.org/debian/
echo digitalplayer > ${WORK_DIR}/etc/hostname

# Set up initramfs for booting with squashfs+aufs
cat >> "${WORK_DIR}/etc/initramfs-tools/modules" <<'EOF'
squashfs
overlayfs
EOF

cat >"${WORK_DIR}/etc/initramfs-tools/scripts/init-bottom/overlayfs" <<'EOF'
#!/bin/sh -e
case $1 in
  prereqs)
    exit 0
    ;;
esac
mkdir /tmpfs
mkdir /squashfs
mount -n -o move ${rootmnt} /squashfs
mount -t tmpfs none /tmpfs
mkdir /tmpfs/upper
mkdir /tmpfs/work
mkdir -p /mnt/boot
mount -t overlay -o redirect_dir=on,lowerdir=/squashfs,upperdir=/tmpfs/upper,workdir=/tmpfs/work none ${rootmnt}
mount -o move /dev ${rootmnt}/dev
mount -o move /sys ${rootmnt}/sys
mount -o move /proc ${rootmnt}/proc
EOF

chmod +x "${WORK_DIR}/etc/initramfs-tools/scripts/init-bottom/overlayfs"
chroot "${WORK_DIR}" ln -s /lib/systemd/systemd /etc/init
chroot "${WORK_DIR}" update-initramfs -u

chroot "${WORK_DIR}" passwd

rm -rf "${WORK_DIR}"/var/cache/apt/*


mount /dev ${WORK_DIR}/dev -o bind
mount /proc ${WORK_DIR}/proc -t proc
mount /sys ${WORK_DIR}/sys -t sysfs


chroot ${WORK_DIR} /bin/bash

umount -l ${WORK_DIR}/dev
umount -l ${WORK_DIR}/proc
umount -l ${WORK_DIR}/sys
