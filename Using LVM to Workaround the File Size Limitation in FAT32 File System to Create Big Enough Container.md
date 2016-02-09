#Using LVM to Workaround the File Size Limitation in FAT32 File System to Create Big Enough Container

#Create container files
    ```for i in {0..1}; do truncate -s 2048M home.$i.vol; done```
#Link the container files to loopback devices
    ```
    losetup /dev/loop0 home.0.vol
    losetup /dev/loop1 home.1.vol
    ```
(and so on)...
Create LVM partition on each of the loopback devices.
fdisk /dev/loop0
fdisk /dev/loop1
(and so on)...
Or use sfdisk: http://download.vikis.lt/doc/util-linux-ng-2.17.2/sfdisk.examples
Create physical volumes
pvcreate /dev/loop0 /dev/loop1 #(and so on)..
(Optional) To automate steps 1 to 4
for i in {3..7}; do sudo losetup /dev/loop$i home.$i.vol && echo ",,8e,," | sudo sfdisk /dev/loop$i && sudo pvcreate /dev/loop$i; done
Create volume group
vgcreate vg_home /dev/loop0 /dev/loop1
Create logical volume
lvcreate -l +100%FREE vg_home -n home
Create encryption key file
mkdir -m 700 /opt/luks-keys
dd if=/dev/random of=/opt/luks-keys/home bs=1 count=256
Encrypt the logical volume using the keyfile
cryptsetup --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 10000 luksFormat /dev/vg_home/home /opt/luks-keys/home
cryptsetup -d /opt/luks-keys/home open --type luks /dev/vg_home/home home
mkfs.ext4 /dev/mapper/home
And mount: mount -t ext4 /dev/mapper/home /mnt/home
Add more space to the encrypted logical volume
create new loopback file: truncate -s 2048M home.2.vol
link the container to loopback device: losetup /dev/loop2 home.2.vol
Create LVM partition on the device: fdisk /dev/loop2
create new physical volume: pvcreate /dev/loop2
Add the PV to VG: vgextend vg_home /dev/loop2
Bring the mounted encrypted volume offline: umount /dev/mapper/home
Close the opened encrypted container: cryptsetup close home
Extend the logical volume: `lvextend -l +100%FREE /dev/vg_home/home`
Open and mount the encrypted container again: cryptsetup -d /opt/luks-keys/home open --type luks /dev/vg_home/home home
(Make sure the opened container isn't mounted automatically)
Resize the opened container: cryptsetup --verbose resize home
Resize the file system inside the container: e2fsck -f /dev/mapper/home && resize2fs /dev/mapper/home
Mount again: mount -t ext4 /dev/mapper/home /mnt/home
Move the loopback files around
umount and close the container:
Deactivate the volume group: vg_change -a n home
Detach the loop devices: losetup -d /dev/loop0 /dev/loop1 /dev/loop2
Move the container files to USB disk
