## Source et énoncé du CTF

<https://hackropole.fr/fr/challenges/forensics/fcsc2022-forensics-echec-op-4>


## Write-up

On cherche à récupérer l’une des adresses IP avec laquelle l'utilisateur a administré le serveur et qu'il a essayé de dissimuler.

---

En vérifiant les éléments de configuration du serveur et ses fichiers de logs susceptibles de contenir des adresses IP, on s'aperçoit assez rapidement que du ménage a été fait : 
* Elements du homedirectory de l'utilisateur *obob*
* */etc/networks/*
* */var/log/*
* */opt/*
* etc.

Cependant, en regardant le contenu du fichier */var/log/fail2ban*, on trouve un éléments intéressant. fail2ban est une application qui analyse les logs de divers services (SSH, Apache, FTP, etc.) en cherchant des correspondances entre des motifs définis dans ses filtres et les entrées des logs, pour ensuite appliquer une action définie au préalable en fonction des occurances trouvées. 

```console
┌──(root㉿kalilinux)-[~]
└─# cat /mnt/var/log/fail2ban.log
2022-03-27 21:28:33,905 fail2ban.server         [948]: INFO    --------------------------------------------------
2022-03-27 21:28:33,905 fail2ban.server         [948]: INFO    Starting Fail2ban v0.11.1
2022-03-27 21:28:33,906 fail2ban.observer       [948]: INFO    Observer start...
2022-03-27 21:28:33,914 fail2ban.database       [948]: INFO    Connected to fail2ban persistent database '/var/lib/fail2ban/fail2ban.sqlite3'
2022-03-27 21:28:33,915 fail2ban.jail           [948]: INFO    Creating new jail 'sshd'
2022-03-27 21:28:33,933 fail2ban.jail           [948]: INFO    Jail 'sshd' uses pyinotify {}
2022-03-27 21:28:33,942 fail2ban.jail           [948]: INFO    Initiated 'pyinotify' backend
2022-03-27 21:28:33,948 fail2ban.filter         [948]: INFO      maxLines: 1
2022-03-27 21:28:34,018 fail2ban.filter         [948]: INFO      maxRetry: 5
2022-03-27 21:28:34,019 fail2ban.filter         [948]: INFO      findtime: 600
2022-03-27 21:28:34,020 fail2ban.actions        [948]: INFO      banTime: 600
2022-03-27 21:28:34,020 fail2ban.filter         [948]: INFO      encoding: UTF-8
2022-03-27 21:28:34,021 fail2ban.filter         [948]: INFO    Added logfile: '/var/log/auth.log' (pos = 0, hash = 3b7f4e88c34b94b39290fbcc7f5bdab08704426d)
2022-03-27 21:28:34,031 fail2ban.jail           [948]: INFO    Jail 'sshd' started
2022-03-27 21:51:44,778 fail2ban.server         [948]: INFO    Shutdown in progress...
2022-03-27 21:51:44,778 fail2ban.observer       [948]: INFO    Observer stop ... try to end queue 5 seconds
2022-03-27 21:51:44,802 fail2ban.observer       [948]: INFO    Observer stopped, 0 events remaining.
2022-03-27 21:51:44,839 fail2ban.server         [948]: INFO    Stopping all jails
2022-03-27 21:51:44,839 fail2ban.filter         [948]: INFO    Removed logfile: '/var/log/auth.log'
2022-03-27 21:51:44,907 fail2ban.actions        [948]: NOTICE  [sshd] Flush ticket(s) with iptables-multiport
2022-03-27 21:51:46,041 fail2ban.jail           [948]: INFO    Jail 'sshd' stopped
2022-03-27 21:51:46,043 fail2ban.database       [948]: INFO    Connection to database closed.
2022-03-27 21:51:46,043 fail2ban.server         [948]: INFO    Exiting Fail2ban
```
Les logs de fail2ban sembleny indiquer une connexion via SSH loggée dans le fichier */var/log/auth.log*, mais ce fichier est absent.

On comprend que nous allons devoir essayer de récupérer les fichiers effacés par l'utilisateur. Pour cela, on peut par exemple utiliser le logiciels *PhotoREC* :

```console
┌──(root㉿kalilinux)-[~]
└─# mkdir photorec_capture

┌──(root㉿kalilinux)-[~]
└─# photorec
---
PhotoRec 7.2, Data Recovery Utility, February 2024
Christophe GRENIER <grenier@cgsecurity.org>
https://www.cgsecurity.org

  PhotoRec is free software, and
comes with ABSOLUTELY NO WARRANTY.

Select a media and choose 'Proceed' using arrow keys:
 Disk /dev/sda - 268 GB / 250 GiB (RO) - VMware Virtual S
 Disk /dev/mapper/loop0p1 - 1048 KB / 1024 KiB (RO)
 Disk /dev/mapper/loop0p2 - 951 MB / 907 MiB (RO)
 Disk /dev/mapper/loop0p3 - 9783 MB / 9330 MiB (RO)
 Disk /dev/mapper/mypartluks - 9766 MB / 9314 MiB (RO)
>Disk /dev/mapper/ubuntu--vg-ubuntu--lv - 9764 MB / 9312 MiB (RO)
 Disk /dev/dm-0 - 1048 KB / 1024 KiB (RO)
 Disk /dev/dm-1 - 951 MB / 907 MiB (RO)
 Disk /dev/dm-2 - 9783 MB / 9330 MiB (RO)
 Disk /dev/dm-3 - 9766 MB / 9314 MiB (RO)
 Disk /dev/dm-4 - 9764 MB / 9312 MiB (RO)
 Disk /dev/loop0 - 10 GB / 10 GiB (RO)
---
Disk /dev/mapper/ubuntu--vg-ubuntu--lv - 9764 MB / 9312 MiB (RO)

     Partition                  Start        End    Size in sectors
      Unknown                        0   19070975   19070976 [Whole disk]
>   P ext4                           0   19070975   19070976
---
   P ext4                           0   19070975   19070976

To recover lost files, PhotoRec needs to know the filesystem type where the
file were stored:
>[ ext2/ext3 ] ext2/ext3/ext4 filesystem
 [ Other     ] FAT/NTFS/HFS+/ReiserFS/...
---
   P ext4                           0   19070975   19070976

Please choose if all space needs to be analysed:
 [   Free    ] Scan for file from ext2/ext3 unallocated space only
>[   Whole   ] Extract files from whole partition
---
Please select a destination to save the recovered files to.
Do not choose to write the files to the same partition they were stored on.
Keys: Arrow keys to select another directory
      C when the destination is correct
      Q to quit
Directory /root/photorec_capture
>drwxr-xr-x     0     0      4096  6-Dec-2025 18:44 .
 drwx------     0     0      4096  6-Dec-2025 18:44 ..
 ---
Disk /dev/mapper/ubuntu--vg-ubuntu--lv - 9764 MB / 9312 MiB (RO)
     Partition                  Start        End    Size in sectors
   P ext4                           0   19070975   19070976

39562 files saved in /root/photorec_capture/recup_dir directory.
Recovery completed.
```

Cela nous a permis de récupérer 39562 fichiers. Il va maintenant falloir faire le tri dans toutes ces informations. On peut par exemple filtrer sur les sorties texte qui comportent le terme *fail2ban* et l'adresse IP "192.168" :
```console
┌──(root㉿kalilinux)-[~]
└─# grep -r "obob" ~/photorec_capture
```

Les résultats sont très nombreux. On peut continuer à filtrer en faisant l'hypothèse que l'adresse IP recherchée commence par "192.168." (plus probable) :
```console
┌──(root㉿kalilinux)-[~]
└─# grep -r "fail2ban" ~/photorec_capture | grep -i "192.168"
...
/root/photorec_capture/recup_dir.80/f17050648.txt:2022-03-27 04:20:08,830 fail2ban.filter         [5082]: INFO    [sshd] Found 192.168.37.1 - 2022-03-27 04:12:26
/root/photorec_capture/recup_dir.80/f17050648.txt:2022-03-27 04:20:08,835 fail2ban.filter         [5082]: INFO    [sshd] Found 192.168.37.130 - 2022-03-27 04:19:08
/root/photorec_capture/recup_dir.80/f17050648.txt:2022-03-27 04:20:13,753 fail2ban.filter         [5082]: 
...
```

```console
┌──(root㉿kalilinux)-[~]
└─# grep -r "ssh" ~/photorec_capture | grep -i "obob"
...
/root/photorec_capture/recup_dir.1/f0316064.txt:Mar 27 21:19:49 obob sshd[1308]: Accepted password for obob from 192.168.37.1 port 33028 ssh2
/root/photorec_capture/recup_dir.1/f0316064.txt:Mar 27 21:19:49 obob sshd[1308]: pam_unix(sshd:session): session opened for user obob by (uid=0)
...
```
Finalement, on peut identifier l'adresse IP : 192.168.37.1

## Flag
FCSC{192.168.37.1}