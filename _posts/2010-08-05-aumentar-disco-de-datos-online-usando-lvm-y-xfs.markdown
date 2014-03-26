---
author: keymon
comments: true
date: 2010-08-05 11:16:10+00:00
layout: post
slug: aumentar-disco-de-datos-online-usando-lvm-y-xfs
title: Aumentar disco de datos online usando LVM y xfs
wordpress_id: 144
categories:
- fast-tip
- linux/unix
- sysadmin
- trick
tags:
- disk
- linux
- lvm
- resize
- scsi
- sysadmin
- tip
- trick
- xfs
---



Un amigo mío me comenta que necesitaba ampliar o incrementar el disco de  un hosting  en internet, y me preguntaba cual sería la mejor forma. Obviamente hay que minimizar el tiempo de caída.

Allí, lo único que le hicieron fué ampliar el disco iSCSI asignado en  50GB. El disco está  particionado en 2 particiones, una de boot y otra de datos, y quiere  ampliar la de datos (ficheros en fs en ext3)  con la mínima disrupción. No usa LVM.

La empresa de hosting le propone reiniciar en modo "administración" (con  una imagen en red) y borrar y crear de nuevo la partición, para luego  redimensionar el fs con resize2fs... Pero no se lo recomiendo porque:



	
  * Es una perdida de servicio muy grande.

	
  * Cada vez que amplie tendrá que reiniciar.

	
  * no me fio de resize2fs, ya me falló con anterioridad.


Yo le propongo que se pase a LVM + xfs. E incluso puede hacerlo sin  reiniciar el servidor, en caliente, y con una parada de servicio mínima (<1min).  En este post comento el procedimiento con comandos simples y disponibles en practiamente todas las distribuciones.

El proceso seria:

	
  1. Hacer backup. Siempre.

	
  2. Reescanear buses y discos scsi para detectar nuevo tamaño de  disco.

	
  3. Reparticionar para crear una nueva partición con el nuevo  espacio. Es mejor extendida, para poder ampliar en futuras ocasiones.

	
  4. Configurar una LV con LVM en el nuevo espacio  (pvcreate, vgcreate, lvcreate).

	
  5. Montar y clonar los datos con rsync.

	
  6. Parar el servicio, resincronizar los últimos cambios con rsync,  intercambiar el punto de montaje y arrancar el servicio.


En el paso 2 nos encontramos un problema. Linux, al ser el mismo de boot  y estar montado no va a recargar la tabla al salir del fdisk. Pero por  lo visto el comando partprobe, que viene con parted, es capaz  de crear las nuevas particiones aún usando ese disco :).

Así que simplemente los pasos son:



	
  1. Decirle al hosting que incremente el disco.

	
  2. Reescanear las fibras con este sencillo script:

    
    cat > reescanea-scsi <<"EOF"
    #!/bin/bash
    
    for fn in /sys/class/scsi_host/*
    do
            host=$(basename $fn)
            echo "Scanning $host ... "
            if [ -d $fn ]; then
                    echo "- - -" > /sys/class/scsi_host/$host/scan
            else
                    echo "ERROR, device not found : '$fn'"
            fi
    done
    
    for disk in /sys/class/scsi_device/*/device/rescan; do
            echo "Rescanning device $disk ..."
            echo 1 > "$disk"
    done
    
    exit 0
    EOF
    chmod +x reescanea-scsi
    ./reescanea-scsi
    


La salida será algo así. Vemos que nos cambia el tamaño:

    
    # ./reescanea-scsi
    Scanning host0 ...
    Scanning host1 ...
    Scanning host2 ...
    Rescanning device /sys/class/scsi_device/0:0:0:0/device/rescan ...
    Rescanning device /sys/class/scsi_device/0:0:2:0/device/rescan ...
    Rescanning device /sys/class/scsi_device/0:0:3:0/device/rescan ...
    Rescanning device /sys/class/scsi_device/1:0:0:0/device/rescan ...
    # dmesg|grep sda 
    sd 0:0:0:0: [sda] 20971520 512-byte hardware sectors: (10.7GB/10.0GiB)
    sd 0:0:0:0: [sda] Test WP failed, assume Write Enabled
    sd 0:0:0:0: [sda] Cache data unavailable
    sd 0:0:0:0: [sda] Assuming drive cache: write through
    sd 0:0:0:0: [sda] 20971520 512-byte hardware sectors: (10.7GB/10.0GiB)
    sd 0:0:0:0: [sda] Test WP failed, assume Write Enabled
    sd 0:0:0:0: [sda] Cache data unavailable
    sd 0:0:0:0: [sda] Assuming drive cache: write through
     sda: sda1 sda2 sda3
    sd 0:0:0:0: [sda] Attached SCSI disk
    Adding 1052248k swap on /dev/sda2.  Priority:1 extents:1 across:1052248k
    EXT3 FS on sda1, internal journal
    sd 0:0:0:0: [sda] 23068672 512-byte hardware sectors: (11.8GB/11.0GiB)
    sd 0:0:0:0: [sda] Write Protect is off
    sd 0:0:0:0: [sda] Mode Sense: 03 00 00 00
    sd 0:0:0:0: [sda] Cache data unavailable
    sd 0:0:0:0: [sda] Assuming drive cache: write through
    sda: detected capacity change from 10737418240 to 11811160064
    




	
  3. Creamos la partición extendida con fdisk (o otro  similar):

    
    # fdisk /dev/sda
    
    The number of cylinders for this disk is set to 1435.
    There is nothing wrong with that, but this is larger than 1024,
    and could in certain setups cause problems with:
    1) software that runs at boot time (e.g., old versions of LILO)
    2) booting and partitioning software from other OSs
       (e.g., DOS FDISK, OS/2 FDISK)
    
    Command (m for help): p
    
    Disk /dev/sda: 11.8 GB, 11811160064 bytes
    255 heads, 63 sectors/track, 1435 cylinders
    Units = cylinders of 16065 * 512 = 8225280 bytes
    Disk identifier: 0x000a4c74
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/sda1   *           1          13      104391   83  Linux
    /dev/sda2              14         144     1052257+  82  Linux swap / Solaris
    /dev/sda3             145        1305     9325732+  8e  Linux LVM
    
    Command (m for help): n
    Command action
       e   extended
       p   primary partition (1-4)
    e
    Selected partition 4
    First cylinder (1306-1435, default 1306):
    Using default value 1306
    Last cylinder, +cylinders or +size{K,M,G} (1306-1435, default 1435):
    Using default value 1435
    
    Command (m for help): p
    
    Disk /dev/sda: 11.8 GB, 11811160064 bytes
    255 heads, 63 sectors/track, 1435 cylinders
    Units = cylinders of 16065 * 512 = 8225280 bytes
    Disk identifier: 0x000a4c74
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/sda1   *           1          13      104391   83  Linux
    /dev/sda2              14         144     1052257+  82  Linux swap / Solaris
    /dev/sda3             145        1305     9325732+  8e  Linux LVM
    /dev/sda4            1306        1435     1044225    5  Extended
    
    Command (m for help): n
    First cylinder (1306-1435, default 1306):
    Using default value 1306
    Last cylinder, +cylinders or +size{K,M,G} (1306-1435, default 1435):
    Using default value 1435
    
    Command (m for help): p
    
    Disk /dev/sda: 11.8 GB, 11811160064 bytes
    255 heads, 63 sectors/track, 1435 cylinders
    Units = cylinders of 16065 * 512 = 8225280 bytes
    Disk identifier: 0x000a4c74
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/sda1   *           1          13      104391   83  Linux
    /dev/sda2              14         144     1052257+  82  Linux swap / Solaris
    /dev/sda3             145        1305     9325732+  8e  Linux LVM
    /dev/sda4            1306        1435     1044225    5  Extended
    /dev/sda5            1306        1435     1044193+  83  Linux
    
    Command (m for help): w
    The partition table has been altered!
    
    Calling ioctl() to re-read partition table.
    
    WARNING: Re-reading the partition table failed with error 16: Device or resource busy.
    The kernel still uses the old table.
    The new table will be used at the next reboot.
    Syncing disks.
    
    dcsrvmonits1:/home/invitado # ls /dev/sda5
    ls: cannot access /dev/sda5: No such file or directory
    


Observamos cómo falla la ioctl de recarga de particiones y no  detecta la nueva partición... Probamos con partprobe:

    
    dcsrvmonits1:/home/invitado # partprobe /dev/sda
    dcsrvmonits1:/home/invitado # ls /dev/sda5
    /dev/sda5
    


Así funciona. Cosa curiosa, no sale ningún mensaje en dmesg.

	
  4. Configuramos LVM... mirate el manual para saber más:

    
    # pvcreate /dev/sda5
    File descriptor 5 left open
      No physical volume label read from /dev/sda5
      Physical volume "/dev/sda5" successfully created
    # pvdisplay
    File descriptor 5 left open
      "/dev/sda5" is a new physical volume of "1019.72 MB"
      --- NEW Physical volume ---
      PV Name               /dev/sda5
      VG Name
      PV Size               1019.72 MB
      Allocatable           NO
      PE Size (KByte)       0
      Total PE              0
      Free PE               0
      Allocated PE          0
      PV UUID               LpvfCq-gzrR-tjC5-N3E2-dA6x-Hmoi-UlPzBK
    
    # vgcreate datavg /dev/sda5
    File descriptor 5 left open
      Volume group "datavg" successfully created
    # vgdisplay datavg
    File descriptor 5 left open
      --- Volume group ---
      VG Name               datavg
      System ID
      Format                lvm2
      Metadata Areas        1
      Metadata Sequence No  1
      VG Access             read/write
      VG Status             resizable
      MAX LV                0
      Cur LV                0
      Open LV               0
      Max PV                0
      Cur PV                1
      Act PV                1
      VG Size               1016.00 MB
      PE Size               4.00 MB
      Total PE              254
      Alloc PE / Size       0 / 0
      Free  PE / Size       254 / 1016.00 MB
      VG UUID               1wC2Vb-omIq-zpDJ-pnUg-oU2f-HaXP-sp29XD
    
    # lvcreate -n reposlv datavg -L 1016.00M
    File descriptor 5 left open
      Logical volume "reposlv" created
    # lvdisplay
      --- Logical volume ---
      LV Name                /dev/datavg/reposlv
      VG Name                datavg
      LV UUID                bIrslV-vSlB-elpP-no2v-B1yt-FO2G-CMjq9l
      LV Write Access        read/write
      LV Status              available
      # open                 0
      LV Size                1016.00 MB
      Current LE             254
      Segments               1
      Allocation             inherit
      Read ahead sectors     auto
      - currently set to     256
      Block device           253:7
    






	
  1. Listo, formateamos, montamos y  sincronizamos:

    
    # mkfs.xfs /dev/datavg/reposlv
    meta-data=/dev/datavg/reposlv    isize=256    agcount=4, agsize=65024 blks
             =                       sectsz=512   attr=2
    data     =                       bsize=4096   blocks=260096, imaxpct=25
             =                       sunit=0      swidth=0 blks
    naming   =version 2              bsize=4096   ascii-ci=0
    log      =internal log           bsize=4096   blocks=1200, version=2
             =                       sectsz=512   sunit=0 blks, lazy-count=0
    realtime =none                   extsz=4096   blocks=0, rtextents=0
    # mkdir /mnt/repos.new
    # mount /dev/datavg/reposlv /mnt/repos.new
    


Clonamos:

    
    # rsync -av --delete /mnt/repos/ /mnt/repos.new
    




	
  2. Paramos un segundo el servicio, volvemos a sincronizar,  damos el cambiazo y arrancamos el servicio. Se puede hacer en un script  de una tacada:

    
    apachectl stop
    rsync -av --delete /mnt/repos/ /mnt/repos.new
    umount /mnt/repos
    umount /mnt/repos.new
    mount /dev/datavg/reposlv /mnt/repos
    apachectl start
    # Actualiza el fstab
    sed -i 's|/dev/sda2|/dev/datavg/reposlv|' /etc/fstab
    




	
  3. Por último, despues de comprobar que todo está ok,  agregamos el viejo espacio al VG y aumentamos así el LV. En caliente :):

    
    pvcreate /dev/sda2
    vgextend datavg /dev/sda2
    lvextend -l FREE /dev/datavg/reposlv
    xfs_growfs /dev/datavg/reposlv
    





Simple ¿no?


