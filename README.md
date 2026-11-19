# Site de Pierre LE QUERE

Site personnel + blog, écrit en Markdown et publié automatiquement par **GitHub Pages** (Jekyll).
Rien à installer : on modifie un fichier, on pousse, c'est en ligne.

## Écrire un article
Crée un fichier dans **`_posts/`** nommé `AAAA-MM-JJ-mon-titre.md` :

```markdown
---
title: Mon titre
description: Une phrase de résumé (affichée dans la liste).
tags: [PowerShell, Teams]
---

Le texte en **Markdown**, avec du code :

    ```powershell
    Get-Mailbox
    ```
```

Le premier tag choisit l'icône de l'article (voir `_data/tags.yml`). Les tags créent automatiquement les pages de sujets ; le temps de lecture,
les liens précédent/suivant et le flux RSS sont générés tout seuls.

## Modifier le contenu
| Quoi | Fichier |
|---|---|
| Accroche, expertises, expériences, certifications, compétences, liens | `_data/cv.yml` |
| Page « À propos » | `a-propos/index.md` |
| Icône de chaque sujet du blog | `_data/tags.yml` + `assets/icons/` |
| Couleurs et typographie | `assets/css/style.css` (en haut du fichier) |
| Titre du site, URL | `_config.yml` |

## Structure
```
_config.yml          réglages
_data/cv.yml         contenu du site
_posts/              articles du blog
_layouts/ _includes/ gabarits (rarement à toucher)
index.html           accueil
cv/                  parcours
blog/                liste des articles + sujets
a-propos/            page À propos
404.html             page d'erreur
assets/              CSS et badges
```

## Mise en ligne
Settings → Pages → Source : *Deploy from a branch* → `main` / `(root)`.
