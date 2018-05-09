# Scripts Review

## Commands
Loop through a list of machiens in the list_hosts file and test for SSH access

```bash
for host in `cat list_hosts`;do ssh -q -o "BatchMode=yes" kevin.markwardt@$host "echo 2>&1" && echo "$host SSH OK"  || echo "$host SSH NOT OK"; done;
```

View Disk Schedulers

```bash
ls /sys/block/ | grep -v loop | grep -v ram | while read i; do echo "Drive $i Scheduler:"; cat /sys/block/$i/queue/scheduler; done;
```

Directory Summary for disk space used

```bash
for i in `ls -1`; do [ $i != 'proc' ] && du -mh /$i/* | sort -nr | head -n 5; done;
```

PIGZ compress and uncompress all files in a directory

```bash
time pigz -p 8 --recursive /directory

time pigz --decompress -p 8 --recursive /directory
```

Flush DNS Cache

```bash
nscd -i hosts;service nscd restart
```

Memory
- Clear Cache
- Turn off and on Swap Space to clear swap usage.  Make sure there is enough Free Physical RAM to cover the Swap used.

```bash
sync; echo 3 > /proc/sys/vm/drop_caches  #Clear Cache

swapoff -a && swapon -a  #Swap off and On
```


### Logical Volumes

**Display Volume Groups and Logical Volumes**

```bash
vgs # Summary
vgdisplay 

lvs # Summary
lvdisplay # Full list
```

**Creating Physical Volume and Volume Group**

```bash
fdisk -l (Find the disk to use to create the Physical volume from)
pvcreate <physical_disk>
vgcreate <vg_name> <physical_disk>

ex.
pvcreate /dev/sdb
vgcreate appvg /dev/sdb
```

**Adding disk to Existing Volume Group**

```bash
fdisk -l  #(Find the disk to use to create the Physical volume from)
pvcreate <physical_disk>
vgextend <vg_name> <physical_disk>

ex.
pvcreate /dev/sdc
vgextend appvg /dev/sdc
```

**Creating Logical Volume**

```bash
lvcreate -L <sizeG> -n <lv_name>  6
mkfs.ext4 /dev/<vg_name>/<lv_name>

ex.
lvcreate -L 50G -n lv_name appvg  #(Specific Size)
lvcreate --extents 100%FREE -n lv_name appvg  #(Percentage)
```

**Extending Logical Volume to use free space in Volume Group**

```bash
lvextend -l +100%FREE /dev/<vg_name>/<lv_name> #(Extend the Logical Volume to 100% of what is available in the Volume Group)

lvextend -L+1G /dev/<vg_name>/<lv_name> #(Extend the logical volume by adding the specified about of space.  In this example 1GB)

lvextend -L12G /dev/<vg_name>/<lv_name> #(Extend the logical volume to the specified size given.  In this example 12G)

ex.

lvextend -l +100%FREE /dev/appvg/lv_name

lvextend -L+1G /dev/appvg/lv_name

lvextend -L12G /dev/appvg/lv_name

```

Make or Growing the Partition

```bash

mkfs.xfs /dev/<vg_name>/<lv_name>  #Create Parition XFS
mkfs.ext4 /dev/<vg_name>/<lv_name>  #Create Parition EXT4

xfs_growfs /dev/<vg_name>/<lv_name> #Grow Partition XFS
resize2fs /dev/<vg_name>/<lv_name>  #Grow Parition EXT4
```


**Reducing Partition and Logical Volume Size (REQUIRES OUTAGE)**

```bash

Requires unmounting / Outage while the operation occurs

umount /dev/<vg_name>/<lv_name>

e2fsck -f /dev/<vg_name>/<lv_name>  #(Check the file system)
resize2fs /dev/<vg_name>/<lv_name> 10G  #(Resize the EXT4 files system to be smaller
xfs_growfs /dev/<vg_name>/<lv_name>
lvreduce --size -1G /dev/<vg_name>/<lv_name>  #(Reduce the size BY a specified amount)
lvreduce --size 10G /dev/<vg_name>/<lv_name>  #(Reduce the size TO a specified amount)

mount /dev/<vg_name>/<lv_name>

ex. 

umount /dev/appvg/lv_name
e2fsck -f /dev/appvg/lv_name
resize2fs /dev/appvg/lv_name 10G

lvreduce --size -1G /dev/appvg/lv_name
or
lvreduce --size 10G /dev/appvg/lv_name
```



**Removing a disk from a volume group**

```bash
pvs -o+pv_used  #(View Current Usage)

pvmove <physical_disk>  #(Move the data off of the disk specified to other disks.  The other disks need to have enough free space to accommodate)
or
pvmove <source_physical_disk> <destination_physical_disk>  #(You can specify where to move the data)

pvs -o+pv_used (View Current Usage, ensure the usage is 0 for the disk to remove)
vgreduce <vg_name> <physical_disk>  #(Removes the disk from the volume group

ex

pvs -o+pv_used  #(View Current Usage)

pvmove /dev/sdb  #(Move the data off of the disk specified to other disks.  The other disks need to have enough free space to accommodate)
or
pvmove /dev/sdb /dev/sdc  #(You can specify where to move the data)

pvs -o+pv_used  #(View Current Usage, ensure the usage is 0 for the disk to remove)
vgreduce appvg /dev/sdb  #(Removes the disk from the volume group)
```