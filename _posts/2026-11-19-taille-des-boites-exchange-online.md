---
title: Lister la taille des boîtes aux lettres Exchange Online
description: Le réflexe indispensable avant de planifier les lots d'une migration.
tags: [PowerShell, Exchange]
---

Pour découper une migration en lots, il faut connaître le volume de chaque boîte.
Le module **ExchangeOnlineManagement** et ses cmdlets `EXO*` font le travail rapidement.

```powershell
Connect-ExchangeOnline

Get-EXOMailbox -ResultSize Unlimited |
    Get-EXOMailboxStatistics |
    Select-Object DisplayName, ItemCount,
        @{ n = 'TailleMo'; e = {
            $octets = $_.TotalItemSize.ToString().Split('(')[1].Split(' ')[0] -replace ','
            [math]::Round([double]$octets / 1MB, 1)
        } } |
    Sort-Object TailleMo -Descending |
    Export-Csv .\tailles-bal.csv -NoTypeInformation -Encoding UTF8
```

## À retenir

- En session distante, `TotalItemSize` arrive sous forme de texte (`1.2 GB (1,288,490,188 bytes)`) :
  on extrait le nombre d'octets pour pouvoir trier correctement.

- Les cmdlets `Get-EXO*` sont plus rapides que les anciennes `Get-Mailbox` sur les gros volumes.
- Pensez aussi aux archives : ajoutez `-Archive` à `Get-EXOMailboxStatistics` pour les mesurer.
- Triez par taille pour placer les plus grosses boîtes dans des lots dédiés.
