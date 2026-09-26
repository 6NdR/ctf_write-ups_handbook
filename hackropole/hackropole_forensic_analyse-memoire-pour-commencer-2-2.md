## Source et énoncé du CTF

<https://hackropole.fr/en/challenges/forensics/fcsc2025-forensics-analyse-memoire-2/>

## Write-up

Un document était en cours d'édition au moment du dump mémoire. On cherche à déterminer les deux informations suivantes à partir du dump mémoire fourni par l'énoncé :
* Le nom du logiciel ayant servi à éditer le document
* Le nom du document en cours d'édition

La commande suivante utilise le module *windows.pslist* qui permet d'afficher les processus actifs dans le dump mémoire :

```console
volatility3 -f analyse-memoire.dmp windows.pslist
```
En regardant la sortie de la commande et en cherchant un nom de processus pouvant correpondre à un logiciel d'édition de texte, on remarque le processus *soffice.exe*. Après quelques recherches, on constate que cet exécutable est utilisé comme lanceur principal de LibreOffice/OpenOffice. Il peut charger différents composants : Writer, Calc, Impress, Draw, Base, Math.

Les formats de fichiers correspondants à *soffice.exe* sont les suivants :

| Type de fichier | Extensions | Description |
|-----------------|------------|-------------|
| Texte (Writer) | .odt, .doc, .docx, .txt, .rtf | Documents texte |
| Tableur (Calc) | .ods, .xls, .xlsx, .csv | Feuilles de calcul |
| Présentation (Impress) |	.odp, .ppt, .pptx | Diapositives |
| Dessin (Draw)	| .odg, .svg, .vsd, .pdf | Dessins vectoriels |
| Base de données (Base) | .odb	| Bases de données |
| Formules (Math) | .odf, .smf | Équations/formules mathématiques |
| Modèles | .ott, .ots, .otp, .otg, etc. | Modèles de document |

---

Le plugin *windows.handles* permet de lister les fichiers ouverts au moment de la réalisation du dump mémoire. Le problème est que ma commande suivante affiche trop de sorties :

```console
volatility3 -f analyse-memoire.dmp windows.handles
```
Il nous faut donc trier la sortie selon le type "file" (pour ne garder que les fichiers) et selon l'extension du fichier recherché (le nom du fichier apparaissant dans la sortie du plugin *windows.handles* et le nom du fichier contient l'extension du fichier).

La commande suivante donnera le fichier recherché :

```console
volatility3 -f analyse-memoire.dmp windows.handles | grep -i file | grep -i odt
```

---

## Flag

FCSC{soffice.exe:[SECRET-SF][TLP-RED]Plan FCSC 2026.odt}