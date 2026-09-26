## Source et énoncé du CTF

<https://hackropole.fr/en/challenges/forensics/fcsc2025-forensics-analyse-memoire-1/>

## Write-up

On cherche à déterminer les trois informations suivantes à partir du dump mémoire fourni par l'énoncé :
* *username* de l'utilisateur de l'ordinateur
* *hostname* de l'ordinateur
* Adresse IPv4 de l'ordinateur

L'analyse du dump mémoire peut être fait via le logiciel *Volatility3*

---

Pour obtenir le *username* et le *hostname*, on utilise la commande suivante :

```console
volatility3 -f analyse-memoire.dmp windows.envars | grep -i name
```
Le plugin *windows.envars* permet d'extraire les variables d'environnement des processus en mémoire. En ne gardant que les variables d'environnement contenant "name", on retrouve facilement les variables d'environnement COMPUTERNAME à la valeur "DESKTOP-JV996VQ" et USERNAME à la valeur "userfcsc-10".

---

Pour obtenir l'adresse IPv4 de l'ordinateur, on utilise la commande suivante :

```console
volatility3 -f analyse-memoire.dmp windows.netscan
```
Le plugin *windows.netscan* permet d'afficher les connexions réseau. L'adresse IPv4 10.0.2.15 apparait à plusieurs reprise dans le champ *LocalAddr*.

---

## Flag
FCSC{userfcsc-10:DESKTOP-JV996VQ:10.0.2.15}