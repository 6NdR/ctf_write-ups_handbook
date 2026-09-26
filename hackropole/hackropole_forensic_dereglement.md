## Source et énoncé du CTF

<https://hackropole.fr/fr/challenges/forensics/fcsc2021-forensics-dereglement/>

## Write-up

On cherche à accéder au contenu d'un fichier Microsoft Office corrompu :

*2021-fcsc-reglement_de_participation.doc*

---

En double-cliquant sur le fichier pour l'ouvrir avec le logiciel Microsoft Office, on peut confirmer qu'il est corrompu avec le message suivant dans la pop-up :

*Word a trouvé du contenu illisible dans 2021-fcsc-reglement_de_participation.doc*

Pour accéder au contenu du fichier sans passer par le logiciel Microsoft Office, il faut se souvenir qu'un fichier Microsoft Office est assimilable à une archive de fichiers *.xml*.

Il suffit donc de modifier l'extension du fichier Microsoft Office de *.docx* à *.zip*, puis d'extraire l'archive et de consulter le contenu du fichier *word\document.xml* pour y trouver le flag.

---

## Flag
FCSC{9bc5a6d51022ac}