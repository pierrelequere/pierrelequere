---
title: Exporter les membres de toutes les équipes Teams en CSV
description: Un script PowerShell court pour inventorier qui est membre ou propriétaire de chaque équipe.
tags: [PowerShell, Teams]
---

Avant une migration ou un audit, une question revient toujours : *qui est dans quelle équipe ?*
Le module **MicrosoftTeams** permet d'y répondre en quelques lignes.

## Prérequis

```powershell
Install-Module MicrosoftTeams -Scope CurrentUser
Connect-MicrosoftTeams
```

## Le script

```powershell
$resultat = foreach ($team in Get-Team) {
    Get-TeamUser -GroupId $team.GroupId | ForEach-Object {
        [pscustomobject]@{
            Equipe      = $team.DisplayName
            Utilisateur = $_.User
            Nom         = $_.Name
            Role        = $_.Role
        }
    }
}

$resultat | Export-Csv .\membres-teams.csv -NoTypeInformation -Encoding UTF8
```

Le fichier contient une ligne par membre et par équipe, avec son rôle (`owner`, `member`
ou `guest`). Il s'ouvre directement dans Excel pour filtrer les équipes sans propriétaire
ou repérer les invités.

> Sur un gros tenant, la boucle peut prendre plusieurs minutes : c'est un appel par équipe.
