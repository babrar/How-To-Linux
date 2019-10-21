# Useful How To's for Linux

## What's this?
This is a list of solutions and tips I found useful when debugging problems on Linux (mostly Debian). I'll keep it updated as I run into more problems.

## Contents

### Setting swap partition manually

A Linux swap partition/file is a bit like an extended RAM that can be used for caching files when RAM is low on space. The swap space can be allocated on the root partition or can be on a separate partition from the root's.

Till Ubuntu 16 (iirc), swap space was assigned on a different partition from root's by default. In the newer Ubuntu versions (i.e. 19.10) however, the swap space is assigned to a 'swapfile' on the root partition instead. This led to a problem for me ... I had already reserved 4 GB of swap space on a separated partition during installation, but Ubuntu chose to create a 2 GB swap space of its own under `/swapfile`.

Following the steps below, I was able to make Ubuntu use my reserved swap partition as a swap storage instead of the default swapfile.

First, using the **fdisk** utility I listed all my partitions. This could also be done using **GParted** which provides a nice and interactive GUI for partitioning (highly recommend to try it out). From the listing, it's clear that `/dev/nvme0n1p7` should be my target swap space.
```sh
# list all partitions
$ sudo fdisk -l  

fdiskDevice             Start       End   Sectors   Size Type
/dev/nvme0n1p1      2048    534527    532480   260M EFI System
/dev/nvme0n1p2    534528    567295     32768    16M Microsoft reserved
/dev/nvme0n1p3    567296   1589247   1021952   499M Windows recovery environment
/dev/nvme0n1p4   1589248   1794047    204800   100M EFI System
/dev/nvme0n1p5   1794048 203157542 201363495    96G Microsoft basic data
/dev/nvme0n1p6 203159552 204466175   1306624   638M Windows recovery environment
/dev/nvme0n1p7 204466176 212467711   8001536   3.8G Linux swap
/dev/nvme0n1p8 212467712 498069503 285601792 136.2G Linux filesystem
/dev/nvme0n1p9 498069504 500117503   2048000  1000M Windows recovery environment

```

However, listing the current swap info. shows that my current swap is actually file called swapfile placed in the root directory.
```sh
# list current swap info
$ sudo swapon --show

NAME           TYPE      SIZE USED PRIO
/swapfile 	   file 	 2.0G   0B   -2
```

I proceeded to re-format the partition as a swap partition and activated it as a swap storage for Linux using the **swapon** command. The **swapon** command can take in the swap partition's UUID or the mount location as it's argument. In this case, I decided to provide it with the UUID (which I found out using the **blkid** command).
```sh
# format partition to be a swap partition
$ sudo mkswap /dev/nvme0n1p7

# list uuid of the swap partition
$ sudo blkid /dev/nvme0n1p7 

/dev/nvme0n1p7: UUID="6100fec9-3553-435d-8d8d-6101ef69d34c" TYPE="swap" ...

# Reconfigure swap to be /dev/nvme0n1p7. UUID is obtained from prev. step 
$ sudo swapon -U <UUID>
```

Finally, to automate the swap partition mounting, I had to add the following lines to my `/etc/fstab` file. More info on **fstab** is available [here](https://help.ubuntu.com/community/Fstab).
```
UUID=<UUID>    none    swap    sw    0    0
```
After a restart (can't be too careful), running **swapon** yielded the result below. Seems like the current swap space is set as my reserved swap partition. A success !!

```sh
# show swap info
$ sudo swapon --show

NAME           TYPE      SIZE USED PRIO
/dev/nvme0n1p7 partition 3.8G   0B   -2  # This is my target swap location !!
```

I went ahead and removed the previous swapfile since it was not being used anymore.
```sh
# free up 2.0 GB of unused swap space
$ sudo rm -f /swapfile 
```

