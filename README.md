# btrfs-debian-ubuntu

Btrfs on Debian or its derivatives such as Ubuntu.

## System Installation

Select Btrfs when installing your OS.

## Check

### findmnt

Print the filesystem type:
```
findmnt -no FSTYPE /
```
Example output:
```
btrfs
```
Print the source of that mount, i.e., the block device, subvolume, network share, or other backing object:
```
findmnt -no SOURCE /
```
Example output:
```
/dev/nvme0n1p4[/@]
```

### btrfs

List Btrfs subvolumes:
```
sudo btrfs subvolume list /
```
Example output:
```
ID 256 gen 928 top level 5 path @
ID 257 gen 928 top level 5 path @home
ID 258 gen 685 top level 5 path @swap
```
Print swap file offset:
```
sudo btrfs inspect-internal map-swapfile -r /swap/swapfile
```

### blkid

Print all block devices' metadata:
```
blkid
```
Print a block device's metadata, `block_device_name` e.g. `/dev/nvme0n1p4`:
```
blkid <block_device_name>
```
Example output:
```
/dev/nvme0n1p4: LABEL="kubuntu_2604" UUID="d5488b10-d932-462d-8cde-3ec4d8ac9102" UUID_SUB="2afdff56-43b0-4ce0-ab31-0e430d5e3537" BLOCK_SIZE="4096" TYPE="btrfs" PARTLABEL="kubuntu_2604" PARTUUID="2aa42c4a-a679-4f41-94a7-26dafffa3602"
```
Print a block device's specific metadata, `metadata_name` e.g `LABEL`, `UUID`, `TYPE`:
```
blkid -s <metadata_name> -o value <block_device_name>
```

### free

Print memory and swap information:
```
free -h
```
Example output:
```
               total        used        free      shared  buff/cache   available
Mem:            15Gi       2.1Gi        10Gi       302Mi       3.3Gi        13Gi
Swap:           27Gi          0B        27Gi
```

### swapon

Print swaps:
```
swapon --show
```
Example output:
```
NAME           TYPE      SIZE USED PRIO
/dev/zram0     partition 7.7G   0B  100
/swap/swapfile file       20G   0B   -1
```

## Swap

Remove existing swap file:
```
sudo swapoff /swap/swapfile
sudo rm /swap/swapfile
```
Optionally print memory information:
```
free -h
```
Create and turn on new swap file:
```
sudo btrfs filesystem mkswapfile --size 20G /swap/swapfile
sudo swapon /swap/swapfile
```
Replace `20G` with your desired swap size. If you want to use hibernate (aka suspend to disk). The swap size must be greater than your RAM size (the `total` size of `Mem` as printed in `free -h`). Refer to [Hibernate aka Suspend to Disk](#hibernate-aka-suspend-to-disk).

Check:
```
swapon --show
```

## Hibernate aka Suspend to Disk

### Introduction

Suspend to disk, also known as hibernate, is the S4 sleeping state as defined by ACPI, which saves the machine's state into swap space and completely powers off the machine. When the machine is powered on, the state is restored. Until then, there is zero power consumption.

In contrast, suspend to RAM, also known as suspend, is the S3 sleeping state as defined by ACPI, which works by cutting off power to most parts of the machine aside from the RAM, which is required to restore the machine's state.

For more information, refer to [Arch Wiki](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate).

### Enable

To enable hibernate, first create a swap file whose size is greater than your RAM size (the `total` size of `Mem` as printed in `free -h`). Refer to [Swap](#swap).

And then, find the block device of your current filesystem:
```
findmnt -no SOURCE /
```
Example output:
```
/dev/nvme0n1p4[/@]
```
Find the UUID of it, with `/dev/nvme0n1p4` below replaced with the output of `findmnt -no SOURCE /` without `[/@]` or similar suffix. The output will be denoted as `<uuid>` afterwards.
```
blkid -s UUID -o value /dev/nvme0n1p4
```
Example output:
```
d5488b10-d932-462d-8cde-3ec4d8ac9102
```
Find the offset of the swap file. The output will be denoted as `<offset>` afterwards.
```
sudo btrfs inspect-internal map-swapfile -r /swap/swapfile
```
Edit `/etc/default/grub`. Change the line
```
GRUB_CMDLINE_LINUX_DEFAULT='quiet splash'
```
to
```
GRUB_CMDLINE_LINUX_DEFAULT='quiet splash resume=UUID=<uuid> resume_offset=<offset>'
```
Update and reboot:
```
sudo update-grub
sudo update-initramfs -u
sudo reboot
```
After reboot, print the parameters passed to the kernel at the time it is started. The output should contain the `resume=UUID=<uuid> resume_offset=<offset>` you set earlier.
```
cat /proc/cmdline
```
### Hibernate

To hibernate your system:
```
sudo systemctl hibernate
```
In desktop environment such as KDE Plasma, the hibernate option may be shown besides sleep, shutdown, etc. after reboot.

## ZRAM

Configure ZRAM with `ram / 2` replaced with your desired ZRAM size relative to RAM size (`ram`) and reboot.
```
sudo apt update
sudo apt install systemd-zram-generator -y
echo '[zram0]
zram-size = ram / 2
compression-algorithm = zstd
swap-priority = 100' | sudo tee /etc/systemd/zram-generator.conf >/dev/null
sudo reboot
```
Check:
```
swapon --show
```

## Create and Mount Subvolumes

Suppose the subvolume you want to create is in `$DIR` (absolute path, e.g., `/data/project`, directory should not already exist).
```
sudo mkdir -p "$(dirname "$DIR")"
sudo btrfs subvolume create "$DIR"
```
And then, find the block device of your current filesystem:
```
findmnt -no SOURCE /
```
Example output:
```
/dev/nvme0n1p4[/@]
```
Test:
```
sudo mount -o subvol=/@"$DIR" /dev/nvme0n1p4 "$DIR"
```
Check:
```
findmnt "$DIR"
```
Example output:
```
TARGET SOURCE                  FSTYPE OPTIONS
/data  /dev/nvme0n1p4[/@/data] btrfs  rw,relatime,ssd,discard=async,space_cache=v2,autodefrag,subvolid=306,subvol=/@/data
```
Find the UUID of the block device, with `/dev/nvme0n1p4` below replaced with the output of `findmnt -no SOURCE /` without `[/@]` or similar suffix, and store in `$UUID`.
```
UUID="$(blkid -s UUID -o value /dev/nvme0n1p4)"
echo $UUID
```
Example output:
```
d5488b10-d932-462d-8cde-3ec4d8ac9102
```
Make mounting permanent by adding to `/etc/fstab`:
```
echo "UUID=${UUID} ${DIR} btrfs subvol=/@${DIR},defaults,noatime,autodefrag 0 0" | sudo tee -a /etc/fstab >/dev/null
```
or with zstd compression:
```
echo "UUID=${UUID} ${DIR} btrfs subvol=/@${DIR},defaults,noatime,autodefrag,compress=zstd 0 0" | sudo tee -a /etc/fstab >/dev/null
```
Reboot:
```
sudo reboot
```
After reboot, add back your `$DIR` variable and check:
```
findmnt "$DIR"
```

## Snapper

### Install and Enable

```
sudo apt update
sudo apt install snapper -y
sudo snapper -c root create-config /
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer
```
Find the block device of your current filesystem:
```
findmnt -no SOURCE /
```
Example output:
```
/dev/nvme0n1p4[/@]
```
Test:
```
sudo mount -o subvol=/@/.snapshots /dev/nvme0n1p4 /.snapshots
```
Check:
```
findmnt /.snapshots
```
Example output:
```
TARGET      SOURCE                        FSTYPE OPTIONS
/.snapshots /dev/nvme0n1p4[/@/.snapshots] btrfs  rw,relatime,ssd,discard=async,space_cache=v2,autodefrag,subvolid=275,subvol=/@/.snapshots
```
Find the UUID of the block device, with `/dev/nvme0n1p4` below replaced with the output of `findmnt -no SOURCE /` without `[/@]` or similar suffix, and store in `$UUID`.
```
UUID="$(blkid -s UUID -o value /dev/nvme0n1p4)"
echo $UUID
```
Example output:
```
d5488b10-d932-462d-8cde-3ec4d8ac9102
```
Make mounting permanent by adding to `/etc/fstab`:
```
echo "UUID=${UUID} /.snapshots btrfs subvol=/@/.snapshots,defaults,noatime,autodefrag 0 0" | sudo tee -a /etc/fstab >/dev/null
```

### Create Config

```
sudo snapper -c <name> create-config <subvolume_mount_path>
```
Note that `snapper -c root create-config /` is already run in the previous session.

List configs:
```
sudo snapper list-configs
```

### Inspect

List snapshots:
```
sudo snapper -c <name> list
```
Compare snapshots: `<fist_#>` and `<second_#>` are the values in the `#` column in `sudo snapper -c <name> list` for the two snapshots.
```
sudo snapper diff <fist_#>..<second_#>
```

### Edit Config

```
sudo vim /etc/snapper/configs/<name>
```
Enable automatic snapshots creation:
```
TIMELINE_CREATE="yes"
```
Enable automatic snapshots cleanup:
```
TIMELINE_CLEANUP="yes"
```
Control retention:
```
TIMELINE_LIMIT_HOURLY="10"
TIMELINE_LIMIT_DAILY="10"
TIMELINE_LIMIT_WEEKLY="0"
TIMELINE_LIMIT_MONTHLY="10"
TIMELINE_LIMIT_YEARLY="10"
```
Number-based cleanup:
```
NUMBER_LIMIT="50"
```

### Manually Create Snapshots

Type `single`:
```
sudo snapper -c <name> create --description <description>
```
Type `pre`: The number outputted is the `<pre_#>` you later use when creating `post` snapshot. You can also find it in the `#` column in `sudo snapper -c <name> list`.
```
sudo snapper -c <name> create --type pre --description <description> --print-number
```
Type `post`: must be linked to a `pre` snapshot. In `sudo snapper -c <name> list`, the `<pre_#>` will be in the `Pre #` column in the `post` snapshot.
```
sudo snapper -c <name> create --type post --pre-number <pre_#> --description <description>
```

### Manually Delete Snapshots

`<#>` is the value in the `#` column in `sudo snapper -c <name> list` for the snapshot.
```
sudo snapper -c <name> delete <#>
```
You can pass multiple numbers to delete multiple snapshots, e.g.,
```
sudo snapper -c root delete 35 36
```

## grub-btrfs

grub-btrfs improves the grub bootloader by adding a btrfs snapshots sub-menu, allowing the user to boot into snapshots.

We will assume your snapshots is in a subvolume that is mounted in `/etc/fstab`, which is done in [Snapper](#snapper).

If you are not on Kali Linux, add Kali Linux kali-last-snapshot brancb APT repo. If you are already are Kali Linux, skip this block.
```
sudo apt update
sudo apt install gnupg wget -y
wget -qO - https://archive.kali.org/archive-key.asc | sudo gpg --dearmor -o /etc/apt/keyrings/kali.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kali.gpg] http://http.kali.org/kali kali-last-snapshot main contrib non-free non-free-firmware' | sudo tee /etc/apt/sources.list.d/kali.list >/dev/null
```
Install:
```
sudo apt update
sudo apt install grub-btrfs -y
sudo systemctl enable --now grub-btrfs.path
```
