## Source et énoncé du CTF

<https://hackropole.fr/fr/challenges/forensics/fcsc2022-forensics-echec-op-2>


## Write-up

On cherche à déterminer la date de création (au format UTC) d'un système de fichiers chiffré de l'image disque.

---

On cherche tout d'abord à déterminer la structure de l'image disque :

```console
──(root㉿kalilinux)-[~]
└─# fdisk -l fcsc.raw 
Disk fcsc.raw: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 60DA4A85-6F6F-4043-8A38-0AB83853E6DC

Device       Start      End  Sectors  Size Type
fcsc.raw1     2048     4095     2048    1M BIOS boot
fcsc.raw2     4096  1861631  1857536  907M Linux filesystem
fcsc.raw3  1861632 20969471 19107840  9,1G Linux filesystem
```

On constate que l'image disque est composée de trois partitions. L'une des partitions est chiffrée et contient le système de fichiers dont on doit trouver la date de création.

Pour savoir quelle partition est chiffrée et quel type de chiffrement est utilisé, il nous faut extraire de l'image disque les trois partitions :

```console
┌──(root㉿kalilinux)-[~]
└─# dd if=fcsc.raw of=fcsc.raw1 bs=512 skip=2048 count=2048 status=progress 
2048+0 enregistrements lus
2048+0 enregistrements écrits
1048576 octets (1,0 MB, 1,0 MiB) copiés, 0,00895892 s, 117 MB/s

┌──(root㉿kalilinux)-[~]
└─# file fcsc.raw1   
fcsc.raw1: data

┌──(root㉿kalilinux)-[~]
└─# dd if=fcsc.raw of=fcsc.raw2 bs=512 skip=4096 count=1857536 status=progress 
918946304 octets (919 MB, 876 MiB) copiés, 8 s, 115 MB/s
1857536+0 enregistrements lus
1857536+0 enregistrements écrits
951058432 octets (951 MB, 907 MiB) copiés, 8,26317 s, 115 MB/s
                                                                                                                                                                                          
┌──(root㉿kalilinux)-[~]
└─# file fcsc.raw2                                                            
fcsc.raw2: Linux rev 1.0 ext4 filesystem data, UUID=427db55e-1263-43a4-9005-dfb083639311 (extents) (64bit) (large files) (huge files)

┌──(root㉿kalilinux)-[~]
└─# dd if=fcsc.raw of=fcsc.raw3 bs=512 skip=1861632 count=19107840 status=progress
9743454720 octets (9,7 GB, 9,1 GiB) copiés, 71 s, 137 MB/s
19107840+0 enregistrements lus
19107840+0 enregistrements écrits
9783214080 octets (9,8 GB, 9,1 GiB) copiés, 71,2294 s, 137 MB/s
                                                                                                                                                          
┌──(root㉿kalilinux)-[~]
└─# file fcsc.raw3                                                                
fcsc.raw3: LUKS encrypted file, ver 2, header size 16384, ID 5, algo sha256, salt 0xc403e369be64e5e5..., UUID: 45e2f0c4-6640-453d-8b7a-8a60bd61c63d, crc 0xf81e270588c709c0..., at 0x1000 {"keyslots":{"1":{"type":"luks2","key_size":64,"af":{"type":"luks1","stripes":4000,"hash":"sha256"},"area":{"type":"raw","offse
```
On remarque que seule la troisième partition  est chiffrée et que le format de chiffrement utilisé est LUKS.

Il nous faut maintenant monter l'image disque *fcsc.raw3* pour accéder à son contenu. On utilise pour cela un *loop device* qui est un périphérique virtuel qui permet de traiter un fichier comme un disque ou une partition physique.

On bind le fichier image disque au périphérique loop */dev/loop0* et on vérifie le mapping :
```console
┌──(root㉿kalilinux)-[~]
└─# losetup /dev/loop0 fcsc.raw
                                                                                                         
┌──(root㉿kalilinux)-[~]
└─# losetup -a          
/dev/loop0: [2049]:4205923 (/data/CTFs/fcsc.raw)
```

Si l'image disque est bien bindée sur /dev/loop0, en revanche, les trois partitions ne sont pas créées par défaut :
```console
┌──(root㉿kalilinux)-[~]
└─# lsblk                      
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0   10G  0 loop 
sda      8:0    0  250G  0 disk 
├─sda1   8:1    0  246G  0 part /
├─sda2   8:2    0    1K  0 part 
└─sda5   8:5    0    4G  0 part [SWAP]
sr0     11:0    1 1024M  0 rom 
```

Pour cela, nous devons créer manuellement les partitions :
```console
┌──(root㉿kalilinux)-[~]
└─# kpartx -av /dev/loop0      
add map loop0p1 (254:0): 0 2048 linear 7:0 2048
add map loop0p2 (254:1): 0 1857536 linear 7:0 4096
add map loop0p3 (254:2): 0 19107840 linear 7:0 1861632

┌──(root㉿kalilinux)-[~]
└─# lsblk   
NAME      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0       7:0    0   10G  0 loop 
├─loop0p1 254:0    0    1M  0 part 
├─loop0p2 254:1    0  907M  0 part 
└─loop0p3 254:2    0  9,1G  0 part 
sda         8:0    0  250G  0 disk 
├─sda1      8:1    0  246G  0 part /
├─sda2      8:2    0    1K  0 part 
└─sda5      8:5    0    4G  0 part [SWAP]
sr0        11:0    1 1024M  0 rom 
```

Nous devons maintenant déchiffrer la partition LUKS qui est la troisième partition de l'image disque et qui est mappée sur */dev/mapper/loop0p3* pointant vers */dev/dm-2* :

```console
┌──(root㉿kalilinux)-[~]
└─# ll /dev/mapper 
total 0
crw------- 1 root root 10, 236 28 nov.  12:02 control
lrwxrwxrwx 1 root root       7 28 nov.  12:02 loop0p1 -> ../dm-0
lrwxrwxrwx 1 root root       7 28 nov.  12:02 loop0p2 -> ../dm-1
lrwxrwxrwx 1 root root       7 28 nov.  12:02 loop0p3 -> ../dm-2

┌──(root㉿kalilinux)-[~]
└─# cryptsetup luksOpen /dev/dm-2 mypartluks
Saisissez la phrase secrète pour /dev/dm-2 : fcsc2022

┌──(root㉿kalilinux)-[~]
└─# lsblk         
NAME                        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
loop0                         7:0    0   10G  0 loop  
├─loop0p1                   254:0    0    1M  0 part  
├─loop0p2                   254:1    0  907M  0 part  
└─loop0p3                   254:2    0  9,1G  0 part  
  └─mypartluks              254:3    0  9,1G  0 crypt 
    └─ubuntu--vg-ubuntu--lv 254:4    0  9,1G  0 lvm   
sda                           8:0    0  250G  0 disk  
├─sda1                        8:1    0  246G  0 part  /
├─sda2                        8:2    0    1K  0 part  
└─sda5                        8:5    0    4G  0 part  [SWAP]
sr0                          11:0    1 1024M  0 rom 

┌──(root㉿kalilinux)-[~]
└─# ll /dev/mapper
total 0
crw------- 1 root root 10, 236 28 nov.  12:02 control
lrwxrwxrwx 1 root root       7 28 nov.  12:02 loop0p1 -> ../dm-0
lrwxrwxrwx 1 root root       7 28 nov.  12:02 loop0p2 -> ../dm-1
lrwxrwxrwx 1 root root       7 28 nov.  12:02 loop0p3 -> ../dm-2
lrwxrwxrwx 1 root root       7 28 nov.  12:15 mypartluks -> ../dm-3
lrwxrwxrwx 1 root root       7 28 nov.  12:15 ubuntu--vg-ubuntu--lv -> ../dm-4
```
Une fois la partition LUKS déchiffrée, nous constatons que cette partition est de type LVM2 et est accessible via le périphérique */dev/ubuntu-vg/ubuntu-lv* pointant vers */dev/dm-4* :

```console
┌──(root㉿kalilinux)-[~]
└─# pvscan
  PV /dev/mapper/mypartluks   VG ubuntu-vg   lvm2 [9,09 GiB / 0    free]
  Total: 1 [9,09 GiB] / in use: 1 [9,09 GiB] / in no VG: 0 [0   ]

┌──(root㉿kalilinux)-[~]
└─# pvdisplay
  --- Physical volume ---
  PV Name               /dev/mapper/mypartluks
  VG Name               ubuntu-vg
  PV Size               <9,10 GiB / not usable 2,00 MiB
  Allocatable           yes (but full)
  PE Size               4,00 MiB
  Total PE              2328
  Free PE               0
  Allocated PE          2328
  PV UUID               ms7xsU-mpDc-Pez2-yI3i-3nLI-49OG-OPSqcC

┌──(root㉿kalilinux)-[~]
└─# vgscan      
  Found volume group "ubuntu-vg" using metadata type lvm2

┌──(root㉿kalilinux)-[~]
└─# vgdisplay
  --- Volume group ---
  VG Name               ubuntu-vg
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  2
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               9,09 GiB
  PE Size               4,00 MiB
  Total PE              2328
  Alloc PE / Size       2328 / 9,09 GiB
  Free  PE / Size       0 / 0   
  VG UUID               W4sUn2-fdU0-9CE8-PFQ7-NDrV-muyv-6KoMd1

┌──(root㉿kalilinux)-[~]
└─# lvscan   
  ACTIVE            '/dev/ubuntu-vg/ubuntu-lv' [9,09 GiB] inherit

┌──(root㉿kalilinux)-[~]
└─# lvdisplay
  --- Logical volume ---
  LV Path                /dev/ubuntu-vg/ubuntu-lv
  LV Name                ubuntu-lv
  VG Name                ubuntu-vg
  LV UUID                W4Y1My-22pb-DbM1-o1IU-dBKO-pJ6O-FcE7sG
  LV Write Access        read/write
  LV Creation host, time ubuntu-server, 2022-03-27 05:44:49 +0200
  LV Status              available
  # open                 1
  LV Size                9,09 GiB
  Current LE             2328
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           254:4
```

En observant la sortie de la commande *lvdisplay*, on trouve finalement les informations pour compléter le flag avec la ligne *LV Creation host, time ubuntu-server, 2022-03-27 05:44:49 +0200* (la date étant donnée à UTC+2).

On peut finalement monter dans le système de fichier de notre machine le volume logique de la partition LUKS déchiffrée, en prévision de la suite du challenge : 
```console
┌──(root㉿kalilinux)-[~]
└─# mount /dev/mapper/ubuntu--vg-ubuntu--lv /mnt

┌──(root㉿kalilinux)-[~]
└─# ll /mnt       
total 1777748
lrwxrwxrwx   1 root root          7 23 févr.  2022 bin -> usr/bin
drwxr-xr-x   2 root root       4096 27 mars   2022 boot
drwxr-xr-x   5 root root       4096 23 févr.  2022 dev
drwxr-xr-x 100 root root       4096 27 mars   2022 etc
drwxr-xr-x   3 root root       4096 27 mars   2022 home
lrwxrwxrwx   1 root root          7 23 févr.  2022 lib -> usr/lib
lrwxrwxrwx   1 root root          9 23 févr.  2022 lib32 -> usr/lib32
lrwxrwxrwx   1 root root          9 23 févr.  2022 lib64 -> usr/lib64
lrwxrwxrwx   1 root root         10 23 févr.  2022 libx32 -> usr/libx32
drwx------   2 root root      16384 27 mars   2022 lost+found
drwxr-xr-x   2 root root       4096 23 févr.  2022 media
drwxr-xr-x   2 root root       4096 23 févr.  2022 mnt
drwxr-xr-x   2 root root       4096 23 févr.  2022 opt
drwxr-xr-x   2 root root       4096 15 avril  2020 proc
drwx------   4 root root       4096 27 mars   2022 root
drwxr-xr-x  11 root root       4096 23 févr.  2022 run
lrwxrwxrwx   1 root root          8 23 févr.  2022 sbin -> usr/sbin
drwxr-xr-x   6 root root       4096 23 févr.  2022 snap
drwxr-xr-x   2 root root       4096 23 févr.  2022 srv
-rw-------   1 root root 1820327936 27 mars   2022 swap.img
drwxr-xr-x   2 root root       4096 15 avril  2020 sys
drwxrwxrwt   9 root root       4096 27 mars   2022 tmp
drwxr-xr-x  14 root root       4096 23 févr.  2022 usr
drwxr-xr-x  14 root root       4096 27 mars   2022 var
```

## Flag
FCSC{2022-03-27T03:44:49Z}