## Source et énoncé du CTF

<https://hackropole.fr/fr/challenges/forensics/fcsc2022-forensics-echec-op-3>


## Write-up

On cherche à déterminer le mot de passe de l’utilisateur principal de la machine.

---

Dans l'épisode précédent, nous avions bindé l'image disque à un périphérique de type LOOP, puis nous avions déchiffré la partition LUKS et enfin nous avions monté le périphérique de type LOOP dans */mnt* :

```console
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

Nous avons maintenant accès au contenu du disque.

Pour trouver l'utilisateur principal de la machine, nous pouvons inspecter le contenu du fichier */etc/passwd* :

```console
┌──(root㉿kalilinux)-[~]
└─# cat /mnt/etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin
landscape:x:109:115::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:110:1::/var/cache/pollinate:/bin/false
usbmux:x:111:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
sshd:x:112:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
obob:x:1000:1000:obob:/home/obob:/bin/bash
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
```
Il s'agit essentiellement de comptes systèmes et de comptes de services reconnaissables par le shell placé à la valeur */bin/false* ou */usr/bin/nologin*. Les seuls comptes utilisateurs ayant pour shell */bin/bash* sont les utilisateurs *root* et *obob*. Le compte *root* étant le compte administrateur, on en déduit que le compte utilisateur recherché est *obob*. 

On peut confirmer cette information en affichant le contenu du répertoire */home* :
```console
┌──(root㉿kalilinux)-[~]
└─# ll /mnt/home
total 12
drwxr-xr-x 19 root root 4096 27 mars   2022 ..
drwxr-xr-x  3 root root 4096 27 mars   2022 .
drwxr-xr-x  6 jld  jld  4096 27 mars   2022 obob
```

Pour trouver le mot de passe, on serait tenté de brut-forcer le hash correspondant au mot de passe, accessibe dans le fichier */etc/shadow* :
```console
┌──(root㉿kalilinux)-[~]
└─# cat /mnt/etc/shadow
...
obob:$6$cvD51kQkFtMohr9Q$vE2L5CUX3jDZgVUZGOFNUFsSHGomH/EP5yYQA3dcKMm9U00mvA9pLzo7Z.Ki6exchu29jEENxtBdGUXCISNxL0:19078:0:99999:7:::
...
```
Mais l'énoncé précise que "la force ne résoud pas tout", laissant penser qu'il s'agit d'un mot de passe fort difficile à cracker. Il faut donc trouver une autre option.

En recherchant les occurences à *obob* dans tous les fichiers à partir de la racine et en filtrant sur *pass*, on obtient la ligne suivante en sortie :
```console
┌──(root㉿kalilinux)-[~]
└─# grep -r "obob" /mnt | grep -i pass
...
/mnt/root/.bash_history:passwd obob
...
```
Il s'emblerait que l'utilisateur *root* ait fait un changement de mot de passe de l'utilisateur *obob* par le passé.

En regardant plus en détail le fichier d'historique de commande de *root*, on obtient le flag en clair :
```console
┌──(root㉿kalilinux)-[~]
└─# cat /mnt/root/.bash_history
exit
passwd obob
CZSITvQm2MBT+n1nxgghCJ
exit
```
Probablement une erreur de manipulation lors du changement de mot de passe de l'utilisateur.


## Flag
FCSC{CZSITvQm2MBT+n1nxgghCJ}