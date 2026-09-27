## Source et énoncé du CTF

<https://www.root-me.org/fr/Challenges/Forensic/Docker-layers>


## Write-up

Le challenge se présente comme une archive .tar a exploiter.

Commençont par analyser son contenu :
```bash
┌──(root㉿kalilinux)-[/data]
└─# tar xvf ch29.tar 
4942a1abcbfa1c325b1d7ed93d3cf6020f555be706672308a4a4a6b6d631d2e7.tar
b58c5e8ccaba8886661ddd3b315989f5cf7839ea06bbe36547c6f49993b0d0aa.tar
743c70a5f809c27d5c396f7ece611bc2d7c85186f9fdeb68f70986ec6e4d165f.tar
316bbb8c58be42c73eefeb8fc0fdc6abb99bf3d5686dd5145fc7bb2f32790229.tar
3309d6da2bd696689a815f55f18db3f173bc9b9a180e5616faf4927436cf199d.tar
8d364403e7bf70d7f57e807803892edf7304760352a397983ecccb3e76ca39fa.tar
b324f85f8104bfebd1ed873e90437c0235d7a43f025a047d5695fe461da717c6.json
5bcc45940862d5b93517a60629b05c844df751c9187a293d982047f01615cb30/layer.tar
5bcc45940862d5b93517a60629b05c844df751c9187a293d982047f01615cb30/VERSION
5bcc45940862d5b93517a60629b05c844df751c9187a293d982047f01615cb30/json
8f0d75885373613641edc42db2a0007684a0e5de14c6f854e365c61f292f3b4d/layer.tar
8f0d75885373613641edc42db2a0007684a0e5de14c6f854e365c61f292f3b4d/VERSION
8f0d75885373613641edc42db2a0007684a0e5de14c6f854e365c61f292f3b4d/json
db04fe239ab708e4ab56ea0e5c1047449b7ea9e04df9db5b1b95d00c6980ff3f/layer.tar
db04fe239ab708e4ab56ea0e5c1047449b7ea9e04df9db5b1b95d00c6980ff3f/VERSION
db04fe239ab708e4ab56ea0e5c1047449b7ea9e04df9db5b1b95d00c6980ff3f/json
82ba49da0bd5d767f35d4ae9507d6c4552f74e10f29777a2a27c97778962476d/layer.tar
82ba49da0bd5d767f35d4ae9507d6c4552f74e10f29777a2a27c97778962476d/VERSION
82ba49da0bd5d767f35d4ae9507d6c4552f74e10f29777a2a27c97778962476d/json
ca7f60c6e2a66972abcc3147da47397d1c2edb80bddf0db8ef94770ed28c5e16/layer.tar
ca7f60c6e2a66972abcc3147da47397d1c2edb80bddf0db8ef94770ed28c5e16/VERSION
ca7f60c6e2a66972abcc3147da47397d1c2edb80bddf0db8ef94770ed28c5e16/json
1bbd61a572ad5f5e2ac0f073465d10dc1c94a71359b0adfd2c105be4c1cb2507/layer.tar
1bbd61a572ad5f5e2ac0f073465d10dc1c94a71359b0adfd2c105be4c1cb2507/VERSION
1bbd61a572ad5f5e2ac0f073465d10dc1c94a71359b0adfd2c105be4c1cb2507/json
manifest.json
repositories
```
Cette archive s'apparente à une image Docker.

Il est possible d'importer l'image Docker à partir de l'archive .tar fournie par le challenge :
```bash
┌──(root㉿kalilinux)-[/data]
└─# docker load -i ch29.tar
Loaded image: rootme/docker_layer:latest
```

Nous pouvons ensuite visualiser l'image importée dans le magasin Docker :
```bash
┌──(root㉿kalilinux)-[/data]
└─# docker image ls        
REPOSITORY            TAG       IMAGE ID       CREATED         SIZE
rootme/docker_layer   latest    b324f85f8104   4 years ago     119MB
```

Nous pouvons également visualiser les différentes layers de l'image :
```bash
┌──(root㉿kalilinux)-[/data]
└─# docker image history rootme/docker_layer:latest           
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
b324f85f8104   4 years ago   /bin/sh -c rm /pass.txt                         0B        
<missing>      4 years ago   /bin/sh -c echo -n $(curl -s https://pastebi…   64B       
<missing>      4 years ago   /bin/sh -c #(nop) COPY file:2ca89eb39686ffcc…   64B       
<missing>      5 years ago   /bin/sh -c apt install -y curl openssl          16.2MB    
<missing>      5 years ago   /bin/sh -c apt update -y                        30.4MB    
<missing>      5 years ago   /bin/sh -c #(nop)  CMD ["bash"]                 0B        
<missing>      5 years ago   /bin/sh -c #(nop) ADD file:d2abf27fe2e8b0b5f…   72.8MB    
```

Nous pouvons visualiser les différentes layers de l'image sans troncage de la sortie :
```bash
┌──(root㉿kalilinux)-[/data]
└─# docker image history rootme/docker_layer:latest --no-trunc
IMAGE                                                                     CREATED       CREATED BY                                                                                                                                      SIZE      COMMENT
sha256:b324f85f8104bfebd1ed873e90437c0235d7a43f025a047d5695fe461da717c6   4 years ago   /bin/sh -c rm /pass.txt                                                                                                                         0B        
<missing>                                                                 4 years ago   /bin/sh -c echo -n $(curl -s https://pastebin.com/raw/P9Nkw866) | openssl enc -aes-256-cbc -iter 10 -pass pass:$(cat /pass.txt) -out flag.enc   64B       
<missing>                                                                 4 years ago   /bin/sh -c #(nop) COPY file:2ca89eb39686ffcc3d2d87bbc9293559252cff471f80c2ed5d024b214f9a6fa3 in /                                               64B       
<missing>                                                                 5 years ago   /bin/sh -c apt install -y curl openssl                                                                                                          16.2MB    
<missing>                                                                 5 years ago   /bin/sh -c apt update -y                                                                                                                        30.4MB    
<missing>                                                                 5 years ago   /bin/sh -c #(nop)  CMD ["bash"]                                                                                                                 0B        
<missing>                                                                 5 years ago   /bin/sh -c #(nop) ADD file:d2abf27fe2e8b0b5f4da68c018560c73e11c53098329246e3e6fe176698ef941 in /        
```
Détail de la fonction des différentes layers de l'image Docker :

1. ADD file:d2abf27... in / — 72.8 MB
Layer de base : c'est le rootfs complet du système (typiquement une image Debian/Ubuntu minimale extraite directement, via ADD d'une archive plutôt qu'un FROM classique). C'est le socle du filesystem.

2. CMD ["bash"] — 0B
Ne modifie aucun fichier, définit juste la commande par défaut exécutée au lancement du conteneur (métadonnée pure, donc 0 octet).

3. apt update -y — 30.4 MB
Met à jour les index de paquets (/var/lib/apt/lists/...). Gros volume car ça télécharge toute la liste des paquets disponibles dans les dépôts configurés.

4. apt install -y curl openssl — 16.2 MB
Installe les deux outils nécessaires à la suite du build : curl pour récupérer le flag chiffré sur pastebin, openssl pour le chiffrer.

5. COPY file:2ca89eb3... in / — 64B
Copie un fichier de 64 octets à la racine de l'image. C'est ici qu'est déposé pass.txt, le mot de passe qui servira de clé de chiffrement.

6. echo -n $(curl -s https://pastebin.com/raw/P9Nkw866) | openssl enc -aes-256-cbc -iter 10 -pass pass:$(cat /pass.txt) -out flag.enc — 64B
L'étape clé : le contenu du flag (récupéré en direct depuis un paste pastebin, présent au moment du build) est chiffré en AES-256-CBC en utilisant le contenu de pass.txt comme passphrase, puis écrit dans flag.enc.
l'époque —d'où l'intérêt de simplement déchiffrer ce qui est déjà dans l'image plutôt que d'essayer de reproduire cette étape.

7. rm /pass.txt — 0B
Supprime pass.txt du filesystem final. C'est cette étape qui fait tout l'intérêt du challenge : la suppression n'efface rien dans la couche précédente (mécanisme overlay/whiteout), donc pass.txt reste entièrement récupérable en extrayant le layer n°5 directement, sans passer par le conteneur final.


En desarchivant les archives contenant les layers de l'image Docker :
```bash
┌──(root㉿kalilinux)-[/data]
└─# tar xvf 316bbb8c58be42c73eefeb8fc0fdc6abb99bf3d5686dd5145fc7bb2f32790229.tar
pass.txt

┌──(root㉿kalilinux)-[/data]
└─# tar xvf 3309d6da2bd696689a815f55f18db3f173bc9b9a180e5616faf4927436cf199d.tar
flag.enc
```

Le flag du challenge est contenu dans le fichier *flag.enc* et le fichier *pass.txt* est la clé de déchiffrement (l'algorithme de chiffrement utilisé est AES-256) :
```bash
┌──(root㉿kalilinux)-[/data]
└─# cat pass.txt 
d4428185a6202a1c5806d7cf4a0bb738a05c03573316fe18ba4eb5a21a1bc8ea                                                                                                                                                                                                  
┌──(root㉿kalilinux)-[/data]
└─# cat flag.enc 
Salted__s`;?�d�q���/�!����$@�����8�=NK:�E�n%���.N��02)�d                                                                                                                                       
```

Pour déchiffrer le flag, il faut faire l'opération inverse du chiffrement qui a été fait dans les différentes layers de l'image Docker, notamment en utilisant les fichiers décrits précédemment :
```bash
┌──(root㉿kalilinux)-[/data]
└─# openssl enc -aes-256-cbc -iter 10 -d -pass pass:$(cat /data/pass.txt) -in /data/flag.enc -out flag.txt 
                                                                                                                                                                                                  
┌──(root㉿kalilinux)-[/data]
└─# cat flag.txt 
Well_D0ne_D0ckER_L@y3rs_Inspect0R
```

## Flag

Well_D0ne_D0ckER_L@y3rs_Inspect0R