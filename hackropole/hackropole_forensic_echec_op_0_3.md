## Source et énoncé du CTF

<https://hackropole.fr/fr/challenges/forensics/fcsc2022-forensics-echec-op-1>


## Write-up

On cherche à récupérer l'identifiant unique (UUID) de la table de partition d'un disque dur.

Le fichier disque est fourni via un fichier RAW *fcsc.raw* contenu dans une archive 7Z *fcsc.7z*.

---

Un UUID (Universally Unique Identifier), également appelé GUID (Globally Unique Identifier), est un identifiant unique utilisé pour identifier de manière fiable une ressource. En particulier, dans le cadre des systèmes de fichiers, l'UUID identifie de manière unique une table de partition.

Pour déterminer l'UUID de la table de partition, il faut au préalable connaitre le type de table de partition utilisé. Pour cela, on utilise The Sleuth Kit pour afficher la structure de la table de partition de l'image disque :

```console
┌──(root㉿kalilinux)-[~]
└─# mmls fcsc.raw
GUID Partition Table (EFI)
Offset Sector: 0
Units are in 512-byte sectors

      Slot      Start        End          Length       Description
000:  Meta      0000000000   0000000000   0000000001   Safety Table
001:  -------   0000000000   0000002047   0000002048   Unallocated
002:  Meta      0000000001   0000000001   0000000001   GPT Header
003:  Meta      0000000002   0000000033   0000000032   Partition Table
004:  000       0000002048   0000004095   0000002048   
005:  001       0000004096   0001861631   0001857536   
006:  002       0001861632   0020969471   0019107840   
007:  -------   0020969472   0020971519   0000002048   Unallocated
```
On remarque qu'on a à faire à une table de partition de type GPT.

Une table de partition de type GPT peut être analysée avec *gdisk*, qui permet de facilement retrouver l'UUID (GUID) de la table de partition :

```console
┌──(root㉿kalilinux)-[~]
└─# gdisk fcsc.raw
GPT fdisk (gdisk) version 1.0.10

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.

Command (? for help): ?
b	back up GPT data to a file
c	change a partition's name
d	delete a partition
i	show detailed information on a partition
l	list known partition types
n	add a new partition
o	create a new empty GUID partition table (GPT)
p	print the partition table
q	quit without saving changes
r	recovery and transformation options (experts only)
s	sort partitions
t	change a partition's type code
v	verify disk
w	write table to disk and exit
x	extra functionality (experts only)
?	print this menu

Command (? for help): p
Disk fcsc.raw: 20971520 sectors, 10.0 GiB
Sector size (logical): 512 bytes
Disk identifier (GUID): 60DA4A85-6F6F-4043-8A38-0AB83853E6DC
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 20971486
Partitions will be aligned on 2048-sector boundaries
Total free space is 4029 sectors (2.0 MiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048            4095   1024.0 KiB  EF02  
   2            4096         1861631   907.0 MiB   8300  
   3         1861632        20969471   9.1 GiB     8300
```

## Flag
FCSC{60DA4A85-6F6F-4043-8A38-0AB83853E6DC}