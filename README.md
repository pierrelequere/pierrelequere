# Site de Pierre LE QUERE

CV + blog, en Markdown, publié automatiquement par **GitHub Pages** (Jekyll). Aucun outil à installer.

## Modifier le CV
Tout est dans **`_data/cv.yml`** : expériences, formations, certifications, compétences, liens.
Ajoute ou supprime un bloc, pousse sur GitHub, c'est en ligne.

## Écrire un article
Crée un fichier dans **`_posts/`** nommé `AAAA-MM-JJ-mon-titre.md` :

```markdown
---
title: Mon titre
tags: [powershell]
---

Le texte en **Markdown**…
```

## Changer le style
Couleurs en haut de **`assets/css/style.css`** (thème clair/sombre automatique).

## Structure
```
_config.yml        réglages (titre, URL)
_data/cv.yml       contenu du CV
_posts/            articles du blog
_layouts/          gabarits HTML (rarement à toucher)
index.html         page CV
blog/index.html    liste des articles
assets/            CSS et images
```

## Mise en ligne
Settings → Pages → Source : *Deploy from a branch* → `main` / `(root)`.
