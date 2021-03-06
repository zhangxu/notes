#Using LVM to Workaround the File Size Limitation in FAT32 File System to Create Big Encrypted Container

##Creation
1. Create container files:
    `truncate -s 2048M home.0.vol && truncate -s 2048M home.1.vol`
2. Link the container files to loopback devices:
    `losetup /dev/loop0 home.0.vol && losetup /dev/loop1 home.1.vol`
3. Create LVM partition on each of the loopback devices:
    
    ```
    fdisk /dev/loop0 #select type 8e or Linux LVM
    fdisk /dev/loop1 #select type 8e or Linux LVM
    ```
4. Or use sfdisk: [See this](http://download.vikis.lt/doc/util-linux-ng-2.17.2/sfdisk.examples)
5. Create physical volumes:
    `pvcreate /dev/loop0 && pvcreate /dev/loop1`
6. (Optional) To automate steps 1 to 5
    
    ```
    for i in {3..7}; do sudo losetup /dev/loop$i home.$i.vol && echo ",,8e,," | sudo sfdisk /dev/loop$i && sudo pvcreate /dev/loop$i; done
    ```
7. Create volume group:
    `vgcreate vg_home /dev/loop0 /dev/loop1`
8. Create logical volume:
    `lvcreate -l +100%FREE vg_home -n home`
9. Create encryption key file
    
    ```
    mkdir -m 700 /opt/luks-keys
    dd if=/dev/random of=/opt/luks-keys/home bs=1 count=256
    ```
10. Encrypt the logical volume using the keyfile
    
    ```
    cryptsetup --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 10000 luksFormat /dev/vg_home/home /opt/luks-keys/home
    cryptsetup -d /opt/luks-keys/home open --type luks /dev/vg_home/home home
    mkfs.ext4 /dev/mapper/home
    ```
11. And mount:
    `mount -t ext4 /dev/mapper/home /mnt/home`

##Add more space to the encrypted logical volume
1. create new loopback file: `truncate -s 2048M home.2.vol`
2. link the container to loopback device: `losetup /dev/loop2 home.2.vol`
3. Create LVM partition on the device: `fdisk /dev/loop2`
4. create new physical volume: `pvcreate /dev/loop2`
5. Add the PV to VG: `vgextend vg_home /dev/loop2`
6. Bring the mounted encrypted volume offline: `umount /dev/mapper/home`
7. Close the opened encrypted container: `cryptsetup close home`
8. Extend the logical volume: `lvextend -l +100%FREE /dev/vg_home/home`
9. Open and mount the encrypted container again: `cryptsetup -d /opt/luks-keys/home open --type luks /dev/vg_home/home home`
10. (Make sure the opened container isn't mounted automatically)
11. Resize the opened container: `cryptsetup --verbose resize home`
12. Resize the file system inside the container: `e2fsck -f /dev/mapper/home && resize2fs /dev/mapper/home`
13. Mount again: `mount -t ext4 /dev/mapper/home /mnt/home`

##Move the loopback files around

1. umount and close the container: `cryptsetup close home && umount /mnt/home`
2. Deactivate the volume group: `vgchange -a n home`
3. Detach the loop devices: `losetup -d /dev/loop0 /dev/loop1 /dev/loop2`
4. Move the container files to USB disk

##Sequence to close the container

1. `umount /mnt/home`
2. `cryptsetup close home`
2. `vgchange -a n vg_home`
3. `a=($(for i in {0..7}; do echo "/dev/loop$i"; done))`
4. `losetup -d ${a[*]}`
5. `vgscan`
6. script
    ```
    #!/bin/bash

    mountpoint -q /mnt/home && sudo umount /mnt/home
    
    sudo cryptsetup status home > /dev/null 2>&1 && sudo cryptsetup close home
    
    sudo vgs vg_home > /dev/null 2>&1 && sudo vgchange -a n vg_home
    
    files=(`ls -v home.*.vol`)
    
    for file in ${files[@]}; do
        sudo losetup -d $(losetup -j `pwd`/$file|awk -F: '{print $1}')
    done
    
    sudo vgscan && sudo pvscan
    ```

## Sequence to open the container

1. `for i in {0..7}; do losetup /dev/loop$i home.$i.vol; done`
2. `vgscan`
2. `vgchange -a y vg_home`
3. `cryptsetup -d /opt/luks-keys/home open --type luks /dev/vg_home/home home`
4. `mount -t ext4 /dev/mapper/home /mnt/home`
5. script
    ```
    #!/bin/bash
    files=(`ls -v home.*.vol`)
    
    for file in ${files[@]}; do
        sudo losetup `losetup -f` $file
    done
    
    sleep 1
    sudo pvscan && sudo vgscan && luks-open /dev/vg_home/home home
    ```
