# Tiered Storage in Linux Using DM-Cache

**Insert foreward and intro**

This is a reference link with numbers, [reference link][1]

### Preperation
First you'll want to have your disk physical volume (PV), SSD PV, and a volume group (VG) that will contain the tiered SSD and Spindle based HDD logical Volumes. Start by creating the PV for the spindle disks, in my case this is in my RAID 0 array named `/dev/md0`. Remember this, all these operations need to be done with root permissions, so either change to root, or put `sudo` in front of all following commands.

```
pvcreate /dev/md0
pvdisplay
```
With pvdisplay you'll be able to see a list of all physical volumes, including ones you may have created before even on other storage devices. Make sure you see `/dev/md0` in there. Next, we'll need the physical volume that will hold the faster SSD backed cache. Mine is on the 4th partition of the disk name `sda`. Create a partition first on your SSD and in following this guide replace `/dev/sda4` with the disk and partition you created for the SSD PV you created for this task.

```
pvcreate /dev/sda4
pvdisplay
```

Verify that your SSD PV was created, mine was `/dev/sda4`, yours will probably be different. Next we need the VG (again, Volume Group) that will contain these two PV's and will solely be used for the tiered storage system. On that note, I will be naming it `TieredVG`, substitute that name as you like, but keep that name in mind in following this guide.

```
vgcreate TieredVG /dev/md0
vgextend TieredVG /dev/sda4
vgdisplay
```

This creates a VG around `/dev/md0`, extends it to include the SSD PV `/dev/sda4`, and displays the VG's present on the system. Ensure that it includes the name you gave it, and is the size of the spindle disk PV and the SSD PV summed together.


### 0. Create OriginLV
Next we create the OriginLV. This is the LV (Logical Volume) that will hold data on the slower, but larger spindle based HDD, that will back any data that may be cached. We will refer it as `OriginLV`, use another name, but keep that name in mind going forward. I'm leaving it as only 98% of the total size of the PV because I want to leave a small space after it to perform tests with, but I see no reason why you couldn't fill it or make it smaller. The `-l` option gives you the option to specify a percentage of the available storage, while the `-L` option of `lvcreate` lets you specify size explicitly with `T`,`G`,`M` for terabyte, gigabyte, megabyte to define how big you want the LV to be.


```
lvcreate -n OriginLV -L 580G TieredVG /dev/md0
lvdisplay
```
```
# lvdisplay
  --- Logical volume ---
  LV Path                /dev/TieredVG/OriginLV
  LV Name                OriginLV
  VG Name                TieredVG
  LV UUID                RY305u-ixWR-Tj8a-jUxR-v631-qx53-VmnRH8
  LV Write Access        read/write
  LV Creation host, time jord, 2016-11-13 21:20:31 -0500
  LV Status              available
  # open                 0
  LV Size                580.00 GiB
  Current LE             148480
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     4096
  Block device           253:1
```

Now after the `lvdisplay` there should be an LV called `OriginLV` of the size specified.

### 1. Create CacheDataLV
With the backing, or `Origin`, volume in place, next there needs to be the faster, smaller volumes that caches the Origin's data. There needs to be two seperate LV's pooled together for this work. First the `CacheDataLV`, then the `CacheMetaLV`. The data volume stores the more frequently used blocks in the faster PV so reads and writes to the most used blocks are tiered in the faster location. The meta LV stores metadata on these blocks to keep track of (among other things) how frequently these blocks are used so the device mapper can make decisions on moving data between the SSD and HDD tiers. Note that by the end of this guide, there will only be the `OriginLV` left visible to the user, and I'll be showing you how to rename it to whatever you want, so I recommend copying the names of all of my LV's (logical volumes) till the end where we will rename it to something more easily understood. **Add this part to the beginning**

Now let's create the `CacheDataLV` to hold the cached data. The man-pages state that the `CacheDataLV` should be no more than 1000x bigger than the `CacheMetaLV`. To keep things simple even though it's major overkill, I'm making the data 150GB, and the meta 2GB. Make sure the `CacheDataLV` exists and is the size you specified by the end of this step.

```
lvcreate -n CacheDataLV -L 150G TieredVG /dev/sda4
lvdisplay
```
```
# lvdisplay
  --- Logical volume ---
  LV Path                /dev/TieredVG/OriginLV
  LV Name                OriginLV
  VG Name                TieredVG
  LV UUID                RY305u-ixWR-Tj8a-jUxR-v631-qx53-VmnRH8
  LV Write Access        read/write
  LV Creation host, time jord, 2016-11-13 21:20:31 -0500
  LV Status              available
  # open                 0
  LV Size                580.00 GiB
  Current LE             148480
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     4096
  Block device           253:1

  --- Logical volume ---
  LV Path                /dev/TieredVG/CacheDataLV
  LV Name                CacheDataLV
  VG Name                TieredVG
  LV UUID                0xFR2d-d9ct-daco-e29c-IZk2-aEL2-H3PCle
  LV Write Access        read/write
  LV Creation host, time jord, 2016-11-13 21:21:11 -0500
  LV Status              available
  # open                 0
  LV Size                150.00 GiB
  Current LE             38400
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:2
```

### 2. Create CacheMetaLV
Now the metadata LV to manage the cache data is needed. So we'll create it as 2GB in my case, but you can make it any size that is at least 1000th of the `CacheDataLV`. It will be in the `TieredVG` or whatever you named your volume group, and backed by the `/dev/sda4` physical volume, or whatever partition you used for your SSD cache physical volume. As before, verify the logical volume information with `lvdisplay`

```
lvcreate -n CacheMetaLV -L 2G TieredVG /dev/sda4
lvdisplay
```
```
# lvdisplay
  --- Logical volume ---
  LV Path                /dev/TieredVG/OriginLV
  LV Name                OriginLV
  VG Name                TieredVG
  LV UUID                RY305u-ixWR-Tj8a-jUxR-v631-qx53-VmnRH8
  LV Write Access        read/write
  LV Creation host, time jord, 2016-11-13 21:20:31 -0500
  LV Status              available
  # open                 0
  LV Size                580.00 GiB
  Current LE             148480
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     4096
  Block device           253:1

  --- Logical volume ---
  LV Path                /dev/TieredVG/CacheDataLV
  LV Name                CacheDataLV
  VG Name                TieredVG
  LV UUID                0xFR2d-d9ct-daco-e29c-IZk2-aEL2-H3PCle
  LV Write Access        read/write
  LV Creation host, time jord, 2016-11-13 21:21:11 -0500
  LV Status              available
  # open                 0
  LV Size                150.00 GiB
  Current LE             38400
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:2

  --- Logical volume ---
  LV Path                /dev/TieredVG/CacheMetaLV
  LV Name                CacheMetaLV
  VG Name                TieredVG
  LV UUID                FCguiU-Xs1P-OLk2-O04v-NZTB-FHhv-fejAtq
  LV Write Access        read/write
  LV Creation host, time jord, 2016-11-13 21:21:56 -0500
  LV Status              available
  # open                 0
  LV Size                2.00 GiB
  Current LE             512
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:3
```

### 3. Create CachePoolLV
With the metadata and data LV's in place, it's time to merge them into a pooled LV that obscures seperation between metadata and data, and registers the metadata LV into the cache pool. This means that when `lvdisplay` is run, all that should be visible is the `CacheDataLV` and it should only be the size of the original `CacheDataLV`. The `lvconvert` command does the actual merging, and it will warn about having to clear the merged volumes, accept unless there's data you need to move or backup first. Another command `lvs` used below shows the inner workings of LVM's device mapper that reveals the two obscured volums `cdata` and `mdata` volumes that came from the merging of the cache pool.
```
lvconvert --type cache-pool --poolmetadata TieredVG/CacheMetaLV TieredVG/CacheDataLV
lvdisplay
lvs -a TieredVG
```
```
# lvdisplay
  --- Logical volume ---
  LV Path                /dev/TieredVG/OriginLV
  LV Name                OriginLV
  VG Name                TieredVG
  LV UUID                RY305u-ixWR-Tj8a-jUxR-v631-qx53-VmnRH8
  LV Write Access        read/write
  LV Creation host, time jord, 2016-11-13 21:20:31 -0500
  LV Status              available
  # open                 0
  LV Size                580.00 GiB
  Current LE             148480
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     4096
  Block device           253:1

  --- Logical volume ---
  LV Path                /dev/TieredVG/CacheDataLV
  LV Name                CacheDataLV
  VG Name                TieredVG
  LV UUID                DelcnE-WQhM-C19l-zswf-Jjl9-zztF-mdDq2j
  LV Write Access        read/write
  LV Creation host, time jord, 2016-11-13 21:22:31 -0500
  LV Status              NOT available
  LV Size                150.00 GiB
  Current LE             38400
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
```
```
# lvs -a TieredVG
  LV                  VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  CacheDataLV         TieredVG Cwi---C--- 150.00g                                                    
  [CacheDataLV_cdata] TieredVG Cwi------- 150.00g                                                    
  [CacheDataLV_cmeta] TieredVG ewi-------   2.00g                                                    
  OriginLV            TieredVG -wi-a----- 580.00g                                                    
  [lvol0_pmspare]     TieredVG ewi-------   2.00g
```

### 4. Create CacheLV
Now we will finally merge the cache pool with the `OriginLV` forming one pool containing the faster SSD-backed cache, it's SSD-backed metadata volume, and the HDD-backed volume that holds the less used files. There are two modes in which the device mapper can write to this pooled volume, `writeback` and `writethrough`. Writethrough ensures that data written to the tiered volume gets written two both the SSD and the HDD at once. This has the benefit of securing the data more because if the SSD fails during the write, or an abrupt shut down occurs it will also have been written to the HDD. However, this is slower than writeback, because writeback will only write to the SSD cache, and queue up a write to the HDD later. Writethrough mode is the default, and when during the `lvconvert` command if no `--cachemode` option is give, it will be assumed write through is desired. If you want writeback like me include the `--cachemode writeback` as I've written it, otherwise ignore that portion of the command. When this is done, there should only be one volume left on the `TieredVG` and that should be the `OriginLV` since the cache-pool is being merged with the HDD store.We'll rename it later. Make sure that the volume is the size of the original OriginLV. The `lvs` command will reveal `cdata`, `cmeta` and a new volume `corig` which is where the old, now obscured origin volume is handled. If it's there you know that now all three logical volumes we created are now functioning together to store data between the cache and its backing store.

```
lvconvert --type cache --cachepool TieredVG/CacheDataLV --cachemode writeback TieredVG/OriginLV
lvdisplay
lvs -a TieredVG
```
```
# lvdisplay
  --- Logical volume ---
  LV Path                /dev/TieredVG/OriginLV
  LV Name                OriginLV
  VG Name                TieredVG
  LV UUID                RY305u-ixWR-Tj8a-jUxR-v631-qx53-VmnRH8
  LV Write Access        read/write
  LV Creation host, time jord, 2016-11-13 21:20:31 -0500
  LV Status              available
  # open                 0
  LV Size                580.00 GiB
  Current LE             148480
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     4096
  Block device           253:1
```
```
# lvs -a TieredVG
  LV                  VG       Attr       LSize   Pool          Origin           Data%  Meta%  Move Log Cpy%Sync Convert
  [CacheDataLV]       TieredVG Cwi---C--- 150.00g                                0.00   0.38            0.00            
  [CacheDataLV_cdata] TieredVG Cwi-ao---- 150.00g                                                                       
  [CacheDataLV_cmeta] TieredVG ewi-ao----   2.00g                                                                       
  OriginLV            TieredVG Cwi-a-C--- 580.00g [CacheDataLV] [OriginLV_corig] 0.00   0.38            0.00            
  [OriginLV_corig]    TieredVG owi-aoC--- 580.00g                                                                       
  [lvol0_pmspare]     TieredVG ewi-------   2.00g
```

### 5. Rename the tiered volume
I named the backing volume `OriginLV` that because it corresponds with the functional names the man page for lvmcache, and it's easier I think for readers to understand what each volume does. Now that every volume has been merged into one pool by that name, it might make sense to name it something that makes more sense for the current functionality of that volume. I'll be calling it `tieredLV` since it's LV that will from now on be used for data I want to be stored on the fast tier if it's used frequently, but still accessible on the slower larger tier if the data is less frequently used. This is done with the `lvrename` command, and again, verify the change with `lvdisplay`
```
lvrename /dev/TieredVG/OriginLV /dev/TieredVG/TieredLV
lvdisplay
```
```
# lvdisplay
--- Logical volume ---
LV Path                /dev/TieredVG/TieredLV
LV Name                TieredLV
VG Name                TieredVG
LV UUID                RY305u-ixWR-Tj8a-jUxR-v631-qx53-VmnRH8
LV Write Access        read/write
LV Creation host, time jord, 2016-11-13 21:20:31 -0500
LV Status              available
# open                 0
LV Size                580.00 GiB
Current LE             148480
Segments               1
Allocation             inherit
Read ahead sectors     auto
- currently set to     4096
Block device           253:1
```

### 6. Mount the TieredLV
Now that everything is in order, the last step is to mount it to the linux filesystem. As I mentioned before, I happen to think one of the best places to have a tiered filesystem is your home folder. A quick note, I've run lvm-cache before and happen to think my configuration of it is pretty solid. Also, all of my home folder items are automatically backed up to server at home, so should the RAID 0 array that hold my home data go awry, which there's a high chance of due to the nature of striped arrays, so I can always restore it.For you, I highly recommend you do one or both of these things first: 1. backup your data that you will mount to, or 2. start by putting the tiered volume on a safe directory like /home/USERNAME/.cache where no crucial data resides, then move the mount to your home, or wherever else you want it when you feel its safe.

With that out of the way, start by formatting the tiered volume. I'll be doing so with ext4, but with a caveat. If you read [Red Hat's overview on dm-cache][2] you'll see that they found that ext4 filesystems has a peculiar reaction to the tiered volume by default. Apperently on their test system a background process name `ext4lazyinit` would constantly run in the background consuming `5-11% of the overall IO`. It is apparently, `A process that backgrounds the creation of the remaining index nodes which are used to reference leaf nodes which is referencing multiple extents. Basically pointers to data on the filesystem.` This is a feature of ext4 which create more harsh (to system IO) itables and journals for the filesystem. But no need to go into more details, let's just perform that lazy ext4 formatting. **NOTE** this will be a slower process than most ext4 formats, so anticipate some time for it to complete. Mine took almost a minute, with a 150GB SSD and 580GB HDD partition.

```
mkfs -t ext4 -E lazy_itable_init=0,lazy_journal_init=0 /dev/mapper/TieredVG-TieredLV
```

With that finished, we can temporarily mount the tiered volume to `/mnt`, or wherever you'd like to temporarily mount it and then move all the data from our desired target directory, my home, to the temporarily mounted tiered volume.

```
mount /dev/TieredVG/tieredLV /mnt
rsync -av --info=progress2 --exclude 'COMMA_SEPERATED_DIRECTORIES_TO_EXCLUDE'
```

With the files now present in the tiered volume, it should be safe to mount it to the home directory, or whatever other directory you want the tier to be mounted on. First, unmount the tiered volume from the temporary location we used to transfer the destinations files to it. Then mount with the `mount` command and specify the tiered volume first as the source, and the target next, in my case `/home`.

```
umount /mnt
mount /dev/TieredVG/tieredLV /home
```


### Links
[1]: http://man7.org/linux/man-pages/man7/lvmcache.7.html
[2]: https://people.redhat.com/mskinner/rhug/q1.2016/dm-cache.pdf
