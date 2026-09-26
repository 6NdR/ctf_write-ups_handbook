## Source et énoncé du CTF

<https://hackropole.fr/fr/challenges/misc/fcsc2022-misc-a-l-envers>

## Write-up

---

### Solution 1 : création d'un canal de communication interactif avec le processus actif du conteneur Docker local via Netcat

***Script d'inversion de la chaine de caractères proposé de manière interactive par le prompt***

```console
#!/bin/bash

LOGFILE="/<path>/fichier.log"
if [ -f "$LOGFILE" ]; then
	rm $LOGFILE
fi

while read prompt
do
	echo $prompt >> $LOGFILE
	if [[ $question == ">>> "* ]]; then
		echo ${question:4} | rev | tee -a $LOGFILE
	fi
done 
```

Détail du script :
- Un fichier de logs enregistre le prompt ainsi que ce qui sera envoyé dans le fichier FIFO au processus du conteneur : si ce fichier existe déjà il est supprimé, sinon il sera créé lors de la première écriture
- Tant que le prompt envoi du texte, ce texte sera lu sur la sortie standard et stocké dans la variable *prompt*
- Lorsque la séquence de texte commençant par >>> est repérée, la séquence de caractères qui suit est isolée, inversée, affichée sur la sortie standard et redirigée vers le fichier de logs (sans écrasement)

***Création du canal de communication interactif avec le processus actif du conteneur Docker local via Netcat***

Créer un fichier spécial pipe/FIFO (First In First Out) nommé *backpipe* permettant de créer un canal de communication bidirectionnel entre deux processus  :
```console
mkfifo /<path>/backpipe
```

Un processus peut **écrire des données dans un pipe** avec la commande suivante :
```console
echo "Hello, World!" > /<path>/backpipe
```

Un processus peut **lire des données d'un pipe** avec la commande suivante :
```console
cat < /<path>/backpipe
```

Remarques :
- Les deux processus (lecture et écriture) doivent fonctionner simultanément, sinon le processus de lecture ou d'écriture sera bloqué jusqu'à ce que l'autre partie soit active
- Lors d'une lecture depuis un fichier FIFO sans qu'un autre processus y écrive simultannéement, le processus de lecture restera en attente

***Exécution du script de façon interactive avec un processus distant via le fichier FIFO créé :***
```console
nc localhost 4000 < /<path>/backpipe | bash /<path>/script.sh > /<path>/backpipe
```
Détails de la commande :
- Lecture depuis le fichier FIFO via la connexion réseau **Netcat** créée sur le port 4000 de l'hôte local
- Le résultat de la lecture est envoyé en entrée standard du script bash local *script.sh* pour traitement
- La sortie standard du script bash local *script.sh* est revoyée en écriture à travers le fichier FIFO

---

### Solution 2 : création d'un canal de communication interactif avec le processus actif sur la machine distante via *exec*

***Script complet permettant l'ouverture d'un canal de communication bidirectionnel, l'inversion de la chaine de caractères et l'envoi de la chaine inversée***

```console
#!/bin/bash

exec 3<>/dev/tcp/localhost/4000

while read prompt 
  do
    echo $prompt
    if [[ $prompt == \>\>\>* ]]
    then
      nprompt=$(echo $prompt |rev |tr -d '>>>')
      echo $nprompt
      echo $nprompt >&3
    fi
done <&3

exec 3<&-
exec 3>&-
```
Détails du script :
- La commande suivante configure un descripteur de fichier (3) pour établir une connexion TCP bidirectionnelle (lecture/écriture) avec un service réseau local sur le port 4000 via le fichier */dev/tcp/localhost/4000* (ce descripteur est alors spécifique de ce fichier) :

```console
exec 3<>/dev/tcp/localhost/4000
```

Remarque :<br> 
La commande *exec* permet (entre autres) de manipuler les descripteurs de fichiers. Cela permet de contrôler la manière dont les fichiers ou flux (entrée/sortie/erreur) sont manipulés.

```console
exec [n]<file       # Ouvre un fichier pour lecture, associe le descripteur n
exec [n]>file       # Ouvre un fichier pour écriture, associe le descripteur n
exec [n]<>file      # Ouvre un fichier pour lecture et écriture, associe le descripteur n
exec [n]&-          # Ferme le descripteur n
exec [n]<&-         # Ferme le descripteur n en lecture
exec [n]>&-         # Ferme le descripteur n en écriture
```

Le fichier */dev/tcp/host/port* est une fonction spécifique du Bash qui permet de fournir une interface virtuelle permettant d'établir une connexion TCP vers une adresse IP (ou un nom d'hôte) et un port spécifiques.

- La boucle *while* reçoit en entrée la sortie du descripteur 3 (via l'opérateur *<&3*) associé au fichier */dev/tcp/localhost/4000*. C'est à dire que le processus de l'hôte *localhost* actif sur le port 4000 écrit en entrée de la boucle *while* via le fichier */dev/tcp/localhost/4000*.
- La boucle *while* traite alors la chaine de caractères reçu en supprimant les *>* et en inversant le mot.
- Chaque chaine traitée est renvoyée vers le processus en interconnexion via la redirection vers le fichier */dev/tcp/localhost/4000*
- Pour terminer, le descripteur de fichier 3 est fermé en lecture/écriture.