---
author: keymon
comments: true
date: 2012-01-15 04:20:32+00:00
layout: post
slug: new-job-assessment-mysql-online-backup-script
title: 'New job assessment: MySQL online backup script'
wordpress_id: 301
categories:
- backup
- linux/unix
- Misc
- mysql
- sysadmin
- Technical
---

These is one of the proposed solutions for the job assessment [commented in a previous post](http://keymon.wordpress.com/2012/01/15/some-posts-fro…job-assessment).


> _How could you backup all the data of a MySQL installation without taking the DB down at all. The DB is 100G and is write intensive. Provide script.
_


My solution to this problem will be _"Online backup using LVM snapshots in a replication slave"_.
It implies that a series of previous decisions have been taken:



	
  * 


**LVM snapshots**: Database data files must be on a [LVM](http://en.wikipedia.org/wiki/Logical_Volume_Manager_%28Linux%29) volume, which will allow the creation of [snapshots](http://www.linux.org/docs/ldp/howto/LVM-HOWTO/snapshots_backup.html). An alternative could be use [Btrfs snapshots](https://btrfs.wiki.kernel.org/index.php/Main_Page).


Advantages:

	
    * Almost online backup: The DB will continue working while copying the data files.

	
    * Simple to setup and without cost.

	
    * You can even start a separated mysql instance (with RW snapshots in LVM2) to perform any task.


Drawbacks:

	
    * To ensure the data file integrity the tables must be locked while creating the snapshot.
The snapshot creation is fast (~2s), but this means a lock anyway.

	
    * All datafiles must be in the same Logical Volume, to ensure an atomic snapshot.

	
    * The snapshot has an overhead that will decrease the write/read performance
([it is said](http://www.mysqlperformanceblog.com/2009/02/05/disaster-lvm-performance-in-snapshot-mode/)
that up to a 6x overhead). This is because any Logical Extend modified must be copied. After a while this overhead will be reduced because changes are usually made in the same Logical Extends.




	
  * 


**Replication slaves**: (see official documentation about[ replication backups](http://dev.mysql.com/doc/refman/5.1/en/replication-solutions-backups.html) and [backup raw data in slaves](http://dev.mysql.com/doc/refman/5.1/en/replication-solutions-backups-rawdata.html) ).
Backup operations will be executed in a slave instead of in the master.


Advantages:

	
    * Will avoid the performance impact in the Master mysql server,
since _"the slave can be paused and shut down without affecting the running operation of the master"_.

	
    * Any backup technique can be done. In fact, using this second method should be enough.

	
    * Simple to setup.

	
    * If there is already a MySQL cluster it will use existent infrastructure.


Drawbacks:

	
    * This is more an architectonic solution or backup policy than a backupscript.

	
    * If there are logical inconsistencies in the slave, they are included in the backup.





So, I will suppose that:

	
  * MySQL data files are stored in a LVM logical volume, with enough free
LEs in the volume group to create snapshots.

	
  * The backupscript will be executed in a working replication slave, not in the master.
Execution of this script on a master will result in a short service pause (tables locked)
and I/O performance impact.


This is the backup script (tested with xfs) that must be executed on a Slave replication server:

([in github](https://gist.github.com/1614287))

[sourcecode language="python"]
#!/usr/bin/python
# -*- coding: utf-8 -*-
#
# Requirements
# - Data files must be in lvm
# - Optionally in xfs (xfs_freeze)
# - User must have LOCK TABLES and RELOAD privilieges::
#
#    grant LOCK TABLES, RELOAD on *.*
#        to backupuser@localhost
#        identified by 'backupassword';
#
import MySQLdb
import sys
import os
from datetime import datetime

# DB Configuration
MYSQL_HOST = "localhost" # Where the slave is
MYSQL_PORT = 3306
MYSQL_USER = "backupuser"
MYSQL_PASSWD = "backupassword"
MYSQL_DB = "appdb"

# Datafiles location and LVM information
DATA_FILES_PATH = "/mysql/data" # do not add / at the end
DATA_FILES_LV = "/dev/datavg/datalv"
SNAPSHOT_SIZE = "10G" # tune de size as needed.

SNAPSHOT_MOUNTPOINT = "/mysql/data.snapshot" # do not add / at the end

# Backup target conf
BACKUP_DESTINATION = "/mysql/data.backup"

#----------------------------------------------------------------
# Commands
# Avoids sudo ask the password
#SUDO = "SUDO_ASKPASS=/bin/true /usr/bin/sudo -A "
SUDO = "sudo"
LVCREATE_CMD =   "%s /sbin/lvcreate" % SUDO
LVREMOVE_CMD =   "%s /sbin/lvremove" % SUDO
MOUNT_CMD =      "%s /bin/mount" % SUDO
UMOUNT_CMD =     "%s /bin/umount" % SUDO
# There is a bug in this command with the locale, we set LANG=
XFS_FREEZE_CMD = "LANG= %s /usr/sbin/xfs_freeze" % SUDO

RSYNC_CMD = "%s /usr/bin/rsync" % SUDO

#----------------------------------------------------------------
# MySQL functions
def mysql_connect():
    dbconn = MySQLdb.connect (host = MYSQL_HOST,
                              port = MYSQL_PORT,
                              user = MYSQL_USER,
                              passwd = MYSQL_PASSWD,
                              db = MYSQL_DB)
    return dbconn

def mysql_lock_tables(dbconn):
    sqlcmd = "FLUSH TABLES WITH READ LOCK"

    print "Locking tables: %s" % sqlcmd

    cursor = dbconn.cursor()
    cursor.execute(sqlcmd)
    cursor.close()

def mysql_unlock_tables(dbconn):
    sqlcmd = "UNLOCK TABLES"

    print "Unlocking tables: %s" % sqlcmd

    cursor = dbconn.cursor()
    cursor.execute(sqlcmd)
    cursor.close()

#----------------------------------------------------------------
# LVM operations
class FailedLvmOperation(Exception):
    pass

# Get the fs type with a common shell script
def get_fs_type(fs_path):
    p = os.popen('mount | grep $(df %s |grep /dev |'\
                 'cut -f 1 -d " ") | cut -f 3,5 -d " "' % fs_path)
    (fs_mountpoint, fs_type) = p.readline().strip().split(' ')
    p.close()
    return (fs_mountpoint, fs_type)

def lvm_create_snapshot():
    # XFS filesystem supports freezing. That is convenient before the snapshot
    (fs_mountpoint, fs_type) = get_fs_type(DATA_FILES_PATH)
    if fs_type == 'xfs':
        print "Freezing '%s'" % fs_mountpoint
        os.system('%s -f %s' % (XFS_FREEZE_CMD, fs_mountpoint))

    newlv_name = "%s_backup_%ilv" % \
                    (DATA_FILES_LV.split('/')[-1], os.getpid())
    cmdline = "%s --snapshot %s -L%s --name %s" % \
        (LVCREATE_CMD, DATA_FILES_LV, SNAPSHOT_SIZE, newlv_name)

    print "Creating the snapshot backup LV '%s' from '%s'" % \
            (newlv_name, DATA_FILES_LV)
    print " # %s" % cmdline

    ret = os.system(cmdline)

    # Always unfreeze!!
    if fs_type == 'xfs':
        print "Unfreezing '%s'" % fs_mountpoint
        os.system('%s -u %s' % (XFS_FREEZE_CMD, fs_mountpoint))

    if ret != 0: raise FailedLvmOperation

    # Return the path to the device
    return '/'.join(DATA_FILES_LV.split('/')[:-1]+[newlv_name])

def lvm_remove_snapshot(lv_name):
    cmdline = "%s -f %s" % \
        (LVREMOVE_CMD, lv_name)

    print "Removing the snapshot backup LV '%s'" % lv_name
    print " # %s" % cmdline

    ret = os.system(cmdline)
    if ret != 0:
        raise FailedLvmOperation

#----------------------------------------------------------------
# Mount the filesystem
class FailedMountOperation(Exception):
    pass

def mount_snapshot(lv_name):
    # XFS filesystem needs nouuid option to mount snapshots
    (fs_mountpoint, fs_type) = get_fs_type(DATA_FILES_PATH)
    if fs_type == 'xfs':
        cmdline = "%s -o nouuid %s %s" % \
                    (MOUNT_CMD, lv_name, SNAPSHOT_MOUNTPOINT)
    else:
        cmdline = "%s %s %s" % (MOUNT_CMD, lv_name, SNAPSHOT_MOUNTPOINT)

    print "Mounting the snapshot backup LV '%s' on '%s'" % \
            (lv_name, SNAPSHOT_MOUNTPOINT)
    print " # %s" % cmdline

    ret = os.system(cmdline)
    if ret != 0:
        raise FailedMountOperation

def umount_snapshot(lv_name):
    cmdline = "%s %s" % (UMOUNT_CMD, SNAPSHOT_MOUNTPOINT)

    print "Unmounting the snapshot backup LV '%s' from '%s'" % \
            (lv_name, SNAPSHOT_MOUNTPOINT)
    print " # %s" % cmdline

    ret = os.system(cmdline)
    if ret != 0:
        raise FailedMountOperation

#----------------------------------------------------------------
# Perform the backup process. For instance, an rsync
class FailedBackupOperation(Exception):
    pass

def do_backup():
    cmdline = "%s --progress -av %s/ %s" % \
                (RSYNC_CMD, DATA_FILES_PATH, BACKUP_DESTINATION)

    print "Executing the backup"
    print " # %s" % cmdline

    ret = os.system(cmdline)
    if ret != 0:
        raise FailedBackupOperation

def main():
    dbconn = mysql_connect()
    mysql_lock_tables(dbconn)

    start_time = datetime.now()
    # Critical, tables are locked!
    snapshotlv = ''
    try:
        snapshotlv = lvm_create_snapshot()
    except:
        print "Backup failed."
        raise
    finally:
        mysql_unlock_tables(dbconn)
        dbconn.close()
        print "Tables had been locked for %s" % str(datetime.now()-start_time)

    try:
        mount_snapshot(snapshotlv)
        do_backup()
        umount_snapshot(snapshotlv)
        lvm_remove_snapshot(snapshotlv)
    except:
        print "Backup failed. Snapshot LV '%s' still exists. " % snapshotlv
        raise

    print "Backup completed. Elapsed time %s" % str(datetime.now()-start_time)

if __name__ == '__main__':
    main()
[/sourcecode]
