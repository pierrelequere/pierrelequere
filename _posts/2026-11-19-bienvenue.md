---
title: Bienvenue sur mon blog
tags: [divers]
---

Ce blog accueille mes notes et retours d'expérience sur l'écosystème Microsoft :
migrations vers Microsoft 365, Teams Voice, Exchange, Entra ID et, bien sûr, PowerShell.

## Écrire un article

Il suffit d'ajouter un fichier dans le dossier `_posts/`, nommé
`AAAA-MM-JJ-titre-de-l-article.md`, qui commence par :

```yaml
---
title: Mon titre
tags: [powershell, exchange]
---
```

Puis d'écrire en Markdown, code compris :

```powershell
Get-Mailbox -ResultSize Unlimited |
    Select-Object DisplayName, PrimarySmtpAddress
```

Dès que le fichier est poussé sur GitHub, l'article est en ligne.
