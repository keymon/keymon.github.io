---
author: keymon
comments: true
date: 2010-11-05 14:04:00+00:00
layout: post
slug: how-to-change-the-exported-lun-number-on-a-vios-virtual-target-disk-or-vtd
title: How to change the exported LUN number on a VIOS "Virtual Target Disk" or VTD
wordpress_id: 232
categories:
- aix
- fast-tip
- script
- sysadmin
- Technical
- trick
tags:
- aix
- ioscli
- lun
- vadapter
- vhost
- vios
- virtual adapter
- virtual disk target
- vtd
---

When you create a "Virtual Target Disk" or VTD on a VIOS, there is not documented way to define or change the LUN number that it shows to the client partition. But there are situation where you might need to update it:
    



	
  1. In a dual VIOS environment, to have the same LUNs in both clients (easier to administrate)-

	
  2. In a redudant configuration, when you need to start lpars on different hardware, using SAN disks. For instance, we use this configuration for our Backup datacenter where we have al the SAN disks mirrored.



In this post I comment how to update this LUN. The idea is basicly:



	
  * Set the VTD device to Defined in the VIOS

	
  * Update the ODM database. You have to update the attribute 'LogicalUnitAddr' in ObjectClass 'CuAt'

	
  * Perform a 'cfgmgr' on the virtual host adapter (vhostX). This will enable the VTD device and reload the LUN number. Perform an cfgmgr on the VTD device does not work.


So, with commands:

    
    $ oem_setup_env
    # bash
    
    # lsmap -vadapter vhost21
    SVSA            Physloc                                      Client Partition ID
    --------------- -------------------------------------------- ------------------
    vhost21         U9117.MMA.XXXXXXX-V2-C34                     0x00000016
    
    VTD                   host01v01
    Status                Available
    LUN                   0x8200000000000000
    Backing device        hdiskpower0
    Physloc               U789D.001.BBBBBBB-P1-C3-T2-L75
    
    # ioscli mkvdev -vadapter vhost21 -dev host01v99 -vdev hdiskpower1
    cfgmgr -l vhost21
    
    # lsmap -vadapter vhost21
    SVSA            Physloc                                      Client Partition ID
    --------------- -------------------------------------------- ------------------
    vhost21         U9117.MMA.XXXXXXX-V2-C34                     0x00000016
    
    VTD                   host01v01
    Status                Available
    LUN                   0x8200000000000000
    Backing device        hdiskpower0
    Physloc               U789D.001.JJJJJJJ-P1-C3-T2-L75
    
    VTD                   host01v99
    Status                Available
    LUN                   0x8300000000000000
    Backing device        hdiskpower1
    Physloc               U789D.001.JJJJJJJ-P1-C3-T2-L77
    
    # rmdev -l host01v99
    host01v99 Defined
    
    # odmget -q "name=host01v99 and attribute=LogicalUnitAddr"  CuAt
    CuAt:
      name = "host01v99"
      attribute = "LogicalUnitAddr"
      value = "0x8300000000000000"
      type = "R"
      generic = "D"
      rep = "n"
      nls_index = 6
    
    # odmchange -o CuAt -q "name = host01v99 and attribute = LogicalUnitAddr" <<"EOF"
    CuAt:
      name = "host01v99"
      attribute = "LogicalUnitAddr"
      value = "0x8100000000000000"
      type = "R"
      generic = "D"
      rep = "n"
      nls_index = 6
    EOF
    
    # odmget -q "name=host01v99 and attribute=LogicalUnitAddr"  CuAt
    CuAt:
      name = "host01v99"
      attribute = "LogicalUnitAddr"
      value = "0x8100000000000000"
      type = "R"
      generic = "D"
      rep = "n"
      nls_index = 6
    
    # cfgmgr -l vhost21
    # lsmap -vadapter vhost21
    SVSA            Physloc                                      Client Partition ID
    --------------- -------------------------------------------- ------------------
    vhost21         U9117.MMA.XXXXXXX-V2-C34                     0x00000016
    
    VTD                   host01v01
    Status                Available
    LUN                   0x8200000000000000
    Backing device        hdiskpower0
    Physloc               U789D.001.JJJJJJJ-P1-C3-T2-L75
    
    VTD                   host01v99
    Status                Available
    LUN                   0x8100000000000000
    Backing device        hdiskpower1
    Physloc               U789D.001.JJJJJJJ-P1-C3-T2-L77
    


In the client partition, you can scan for the new disk, and it will have the LUN 0x81:

    
    root@host01:~/# cfgmgr -l vio0
    root@host01:~/# lscfg -vl hdisk5
      hdisk5           U9117.MMA.XXXXXXX-V22-C3-T1-L8100000000000000  Virtual SCSI Disk Drive
    


**Note:** Actually I changed the output of these commands to remove information of my company.

**Update**: I created an script to do this: [change_vtd_lun.sh](https://gist.github.com/raw/667682/change_vtd_lun.sh)


