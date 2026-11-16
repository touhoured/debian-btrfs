#!/usr/bin/env bash
set -euo pipefail

# Usage: sudo bash scripts/build-debian-ecs-btrfs.sh SUITE {true|false}
if (( $# != 2 )); then
  echo "Usage: $0 {bookworm|trixie|forky|testing|sid} {true|false}" >&2
  exit 2
fi
SUITE=$1
CLOUD_INIT=$2
case "$SUITE" in
  bookworm) version=12 ;;
  trixie) version=13 ;;
  forky|testing|sid) version='' ;;
  *) echo "Unsupported Debian release: $SUITE" >&2; exit 1 ;;
esac
case "$CLOUD_INIT" in
  true|false) ;;
  *) echo "Invalid cloud-init switch: $CLOUD_INIT" >&2; exit 1 ;;
esac

if (( EUID != 0 )); then
  echo "Run this script as root (mount, loop devices and chroot are required)." >&2
  exit 1
fi

work=$(mktemp -d)
root="$work/root"
disk="$work/disk.raw"
loop=''
cleanup() {
  if mountpoint -q "$root"; then umount --recursive "$root" || return; fi
  if [[ -n "$loop" ]]; then losetup --detach "$loop" || return; fi
  rm -rf "$work"
}
trap cleanup EXIT
mkdir -p "$root" output

# BIOS/MBR, 1 MiB embedding gap; root is the only/last partition.
truncate -s 20G "$disk"
parted -s "$disk" mklabel msdos mkpart primary btrfs 1MiB 100% set 1 boot on
loop=$(losetup --find --show --partscan "$disk")
part="${loop}p1"
for ((i=0; i<50; i++)); do
  [[ -b "$part" ]] && break
  sleep 0.1
done
test -b "$part"
mkfs.btrfs -f "$part"
mount -o compress=zstd "$part" "$root"

debootstrap --arch=amd64 --variant=minbase --include=ca-certificates \
  "$SUITE" "$root" https://deb.debian.org/debian
# debootstrap copies the builder's hostname; let the guest initialize its own.
rm -f "$root/etc/hostname"
echo "deb https://deb.debian.org/debian $SUITE main" > "$root/etc/apt/sources.list"
case "$SUITE" in
  bookworm|trixie)
    cat >> "$root/etc/apt/sources.list" <<EOF
deb https://deb.debian.org/debian $SUITE-updates main
deb https://security.debian.org/debian-security $SUITE-security main
EOF
    ;;
  # Unreleased forky, testing and sid use main rather than stable pockets.
esac
uuid=$(blkid -s UUID -o value "$part")
printf 'UUID=%s / btrfs defaults,compress=zstd 0 0\n' "$uuid" > "$root/etc/fstab"

mount --rbind /dev "$root/dev"
mount --make-rslave "$root/dev"
mount -t proc proc "$root/proc"
mount -t sysfs sysfs "$root/sys"
mount -t tmpfs tmpfs "$root/run"
# Prevent package installation from starting services in the builder.
printf '#!/bin/sh\nexit 101\n' > "$root/usr/sbin/policy-rc.d"
chmod 755 "$root/usr/sbin/policy-rc.d"
cp /etc/resolv.conf "$root/etc/resolv.conf"

chroot "$root" /usr/bin/env TMPDIR=/tmp CLOUD_INIT="$CLOUD_INIT" \
  DEBIAN_FRONTEND=noninteractive /bin/bash -euo pipefail <<'CHROOT'
packages=(linux-image-cloud-amd64 systemd-sysv initramfs-tools grub-pc-bin grub2-common
          btrfs-progs openssh-server iproute2 systemd-resolved)
if [[ "$CLOUD_INIT" == true ]]; then
  # Recommends are off: growpart and Debian's cloud user's sudo are needed.
  packages+=(cloud-init cloud-guest-utils sudo)
fi
apt-get update
apt-get install -y --no-install-recommends "${packages[@]}"

# Garden Linux cloud/ali use networkd for DHCP and resolved for DNS.
# https://github.com/gardenlinux/gardenlinux/blob/main/features/cloud/file.include/etc/systemd/network/99-default.network
mkdir -p /etc/systemd/network
cat > /etc/systemd/network/99-default.network <<'EOF'
[Match]
Name=en* eth*

[Network]
DHCP=yes

[DHCPv4]
UseDomains=true
UseMTU=true
EOF
systemctl enable systemd-networkd
if [[ "$CLOUD_INIT" == true ]]; then
  # Keep Debian's default user, SSH modules and resize defaults.
  # As in Garden Linux ali, networkd owns networking; cloud-init consumes metadata.
  # minbase already has C.UTF-8. Debian 12 otherwise requests an ungenerated locale.
  cat > /etc/cloud/cloud.cfg.d/99-ecs.cfg <<'EOF'
datasource_list: [AliYun]
locale: C.UTF-8
network: {config: disabled}
EOF
  python3 - <<'PY'
import yaml
cfg = yaml.safe_load(open('/etc/cloud/cloud.cfg'))
assert cfg['system_info']['default_user']['name'] == 'debian'
assert {'growpart', 'resizefs', 'ssh', 'users-groups'} <= set(cfg['cloud_init_modules'])
PY
  command -v growpart
  cloud-init clean --logs
else
  # Recent openssh-server supplies this; older Debian needs a first-boot unit.
  if [[ ! -f /usr/lib/systemd/system/sshd-keygen.service ]]; then
    cat > /etc/systemd/system/ssh-host-keys.service <<'EOF'
[Unit]
Description=Generate SSH host keys
Before=ssh.service

[Service]
Type=oneshot
ExecStart=/usr/bin/ssh-keygen -A
RemainAfterExit=yes

[Install]
WantedBy=ssh.service
EOF
    systemctl enable ssh-host-keys.service
  fi
  test ! -e /etc/cloud
  test ! -e /var/lib/cloud
  ! dpkg-query -s cloud-init 2>/dev/null | grep -qx 'Status: install ok installed'
fi

mkdir -p /etc/default/grub.d
cat > /etc/default/grub.d/ecs.cfg <<'EOF'
GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200n8"
EOF

# The kernel package generated the initramfs using Debian's MODULES=most default.
# Check that default rather than duplicating its module selection.
for initrd in /boot/initrd.img-*; do
  lsinitramfs "$initrd" > /tmp/initrd-files
  kernel=${initrd#/boot/initrd.img-}
  for module in virtio_pci virtio_blk virtio_scsi virtio_net nvme btrfs; do
    grep -Eq "/${module}\.ko(\.(xz|gz|zst))?$" /tmp/initrd-files ||
      grep -Eq "/${module}\.ko$" "/lib/modules/$kernel/modules.builtin"
  done
done
mkdir -p /run/sshd
/usr/sbin/sshd -t
# Retain debootstrap's locked root; never set a default password.
awk -F: '$1 == "root" { if ($2 !~ /^[!*]/) exit 1; found=1 }
         END { if (!found) exit 1 }' /etc/shadow
apt-get clean
rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
rm -f /etc/ssh/ssh_host_* /var/lib/systemd/random-seed
rm -f /var/lib/dbus/machine-id
echo uninitialized > /etc/machine-id
# resolved's maintainer script installed the resolver symlink; retain it.
test -L /etc/resolv.conf
rm -f /etc/.resolv.conf.systemd-resolved.bak /usr/sbin/policy-rc.d
find /var/log -type f -exec truncate -s 0 {} +
CHROOT

grep -qx 'ID=debian' "$root/etc/os-release"
if [[ -n "$version" ]]; then
  grep -qx "VERSION_ID=\"$version\"" "$root/etc/os-release"
fi
if [[ "$SUITE" == forky ]]; then
  # Prerelease Debian 14 may omit VERSION_ID; the codename must still match.
  grep -qx 'VERSION_CODENAME=forky' "$root/etc/os-release"
fi
cat "$root/etc/os-release"
chroot "$root" grub-install --target=i386-pc --recheck "$loop"
chroot "$root" update-grub
chroot "$root" grub-script-check /boot/grub/grub.cfg
grep -F "root=UUID=$uuid" "$root/boot/grub/grub.cfg"
rm -f "$root/boot/grub/device.map"
sync
# Reclaim deleted bootstrap/package data before converting the sparse images.
fstrim "$root"
umount --recursive "$root"
btrfs check --readonly "$part"
losetup --detach "$loop"
loop=''

image="output/debian-$SUITE-amd64-cloud-init-$CLOUD_INIT"
# VHD is 'vpc' in QEMU. Avoid CHS capacity rounding.
qemu-img convert -f raw -O vpc -o subformat=dynamic,force_size=on "$disk" "$image.vhd"
qemu-img info -f vpc "$image.vhd"
# VHD has no qemu-img check implementation; compare virtual disk contents.
qemu-img compare -f raw -F vpc "$disk" "$image.vhd"
qemu-img convert -f raw -O qcow2 "$disk" "$image.qcow2"
qemu-img check -f qcow2 "$image.qcow2"
qemu-img compare -f raw -F qcow2 "$disk" "$image.qcow2"
(cd output && sha256sum ./*.vhd ./*.qcow2 > SHA256SUMS)
