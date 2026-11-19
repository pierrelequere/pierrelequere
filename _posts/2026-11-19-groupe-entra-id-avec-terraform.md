---
title: Créer un groupe Entra ID avec Terraform
description: Premier pas avec le provider azuread pour décrire ses groupes en code.
tags: [Terraform, Entra ID]
---

Créer un groupe à la main dans le portail, c'est rapide. Le recréer à l'identique dans
un autre environnement, beaucoup moins. Avec Terraform, le groupe devient du code
versionné, relu et rejoué par un pipeline.

## Le code

```hcl
terraform {
  required_providers {
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 3.0"
    }
  }
}

provider "azuread" {}

data "azuread_client_config" "current" {}

resource "azuread_group" "devops" {
  display_name     = "GRP-DevOps"
  description      = "Équipe DevOps"
  security_enabled = true
  owners           = [data.azuread_client_config.current.object_id]
}
```

## L'appliquer

```powershell
az login
terraform init
terraform plan
terraform apply
```

`terraform plan` montre ce qui va changer avant de toucher quoi que ce soit : c'est
l'étape à faire tourner automatiquement sur chaque pull request.
