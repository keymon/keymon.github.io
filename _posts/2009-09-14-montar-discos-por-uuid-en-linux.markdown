---
author: keymon
comments: true
date: 2009-09-14 10:00:30+00:00
layout: post
slug: montar-discos-por-uuid-en-linux
title: Mount disks using UUID in linux
wordpress_id: 16
categories:
- fast-tip
- linux/unix
---






To mount disks using its UUID:



	
  1. Get UUID con blkid from e2fsprogs (ok for xfs, ext3, ext2, etc):

    
    # blkid
    /dev/sdb1: UUID="2773e912-7b76-44d0-adf9-2d5f0f0719d5" TYPE="xfs"
    /dev/sdd1: UUID="2773e912-7b76-44d0-adf9-2d5f0f0719d5" TYPE="xfs"
    /dev/sda1: UUID="1c1a29b6-f182-4fdf-91cb-efc57300ea45" SEC_TYPE="ext2" TYPE="ext3"
    /dev/sdf2: UUID="05aa92dd-2e85-4c7c-95f2-67169570e013" TYPE="xfs"
    /dev/sda2: TYPE="swap" UUID="0d01df78-6bce-4ece-a61e-97e8267a5988"
    /dev/sda5: UUID="5b530599-0a58-442b-8b98-4d7bd7cdf7ec" SEC_TYPE="ext2" TYPE="ext3"
    /dev/sda6: UUID="eaf78b36-f111-4a9e-82b3-c53fac2c3961" SEC_TYPE="ext2" TYPE="ext3"
    /dev/sda7: UUID="6d57fa4b-8f6a-4d1b-9bba-1808fe7fe36d" SEC_TYPE="ext2" TYPE="ext3"
    /dev/sdc: UUID="1674e324-6bea-407d-91c2-38533ccc0059" TYPE="xfs"
    /dev/sdg: UUID="1674e324-6bea-407d-91c2-38533ccc0059" TYPE="xfs"




	
  2. Set UUID when you mount

    
    mount -t xfs -U 05aa92dd-2e85-4c7c-95f2-67169570e013 /storage




	
  3. Or add it to fstab:

    
    UUID="05aa92dd-2e85-4c7c-95f2-67169570e013" /storage        xfs     defaults 0       2









