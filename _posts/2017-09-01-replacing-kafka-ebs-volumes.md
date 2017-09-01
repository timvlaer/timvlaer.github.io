---
layout: post
title: "Replace EBS volumes of a Kafka installation"
description: "Replace EBS volumes of a Kafka installation"
date: 2017-09-07
tags: [kafka, aws]
comments: true
share: true
---

Enlarging AWS EBS volumes is very easy and straight-forward. It is however not possible to decrease the size of the volume.

The performance of EBS volumes is strictly related to the size of the disks: bigger disks get higher I/O throughput. 
 So it might be easy to speed up kafka doing I/O intensive work (like reindexing) by temporarily increasing the disk size.
 We wanted to decrease the disks afterwards, because we don't need the extra storage for now and want to avoid the extra cost for it.
 Problem is that when you want to decrease the volume of an EBS disk, you have to replace it with a new one. Here is how you do that:

1. Create a new EBS volume and attach it to the kafka broker machine.
1. Check with `lsblk` the new disk device 
    ```bash
     $ lsblk
     NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
     xvdc    202:32   0    2T  0 disk /kafka-data2
     xvdf    202:80   0  500G  0 disk /kafka-data1
     xvdg    202:96   0  500G  0 disk 
    ```
1. Prepare it with the right filesystem `sudo mkfs -t xfs /dev/xvdg`
1. Make a mount point on the broker `sudo mkdir /kafka-data3`
1. Mount the disk on the broker `sudo mount /dev/xvdg /kafka-data3`
1. Gracefully stop kafka `sudo supervisorctl stop kafka`
1. Check if the `.kafka_cleanshutdown` file is in the data directory. It is placed by Kafka only when a shutdown succeeded. 
    If it is not there, kafka will reindex all files at restart which will take some time. (And might impact the disk's [burst balance](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html).)
1. Copy the data of the old ebs volume to the new one. `rsync -av /kafka-data1/* /kafka-data3`. 
    This will take a while.
    Optionally you can enable the `--checksum` flag, but this will have a performance impact. 
1. Explicitly copy the cleanshutdown flag as it is not copied by the rsync command. 
    `cp /kafka-data1/.kafka_cleanshutdown /kafka-data3`
1. Swap the mount points 
    ```bash
    sudo umount /kafka-data1
    sudo umount /kafka-data3
    sudo mount /dev/xvdg /kafka-data1
    ```
1. Restart Kafka `sudo supervisorctl start kafka`