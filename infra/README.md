Okay, lass uns das einmal sauber aufziehen: Terragrunt + OpenTofu, mehrere Umgebungen in **einem** AWS Account, aber so strukturiert, dass du später jede Umgebung leicht in **einen eigenen Account migrieren** kannst.

Ich skizziere dir:

1. Empfohlene GitHub-Repo-Struktur
2. Terragrunt-Hierarchie & Account-Trennung
3. Beispiel-Configs (root `terragrunt.hcl`, Umgebungs- & Modul-Level)
4. Wie die spätere Migration in eigene AWS Accounts konkret abläuft

---

## 1. GitHub-Repo-Struktur

Mono-Repo, sauber getrennt in **live** (Instanzen je Umgebung) und **modules** (wiederverwendbare OpenTofu-Module):

```text
infra/
  modules/
    network/
      vpc/
      transit-gateway/
    security/
      iam-roles/
      kms/
    compute/
      ecs/
      eks/
      asg/
    data/
      rds/
      s3-app-buckets/
  live/
    eu-central-1/
      dev/
        vpc/
        security/
        app/
      staging/
        vpc/
        security/
        app/
      prod/
        vpc/
        security/
        app/
  terragrunt.hcl          # Root-Konfiguration
  README.md
```

Wichtige Punkte:

* **modules** sind generisch, kennen keine Umgebung und keinen Account.
* **live** beschreibt konkrete Stacks (VPC dev, RDS prod, …) pro Umgebung.
* Die Umgebung steckt im Pfad (`dev`, `staging`, `prod`), nicht im Modul selbst → Migration = nur Konfiguration/Backend/Account wechseln.

---

## 2. Terragrunt-Strategie für spätere Account-Trennung

Ziel: Am Anfang laufen alle Umgebungen in **einem** AWS Account (z. B. `111111111111`). Später bekommen sie ihre **eigenen Accounts**, ohne dass du deine Verzeichnisstruktur umwerfen musst.

Dafür legst du die Account-Zuordnung **zentral im Root-terragrunt** fest.

### Root `terragrunt.hcl` (im Repo-Root `infra/terragrunt.hcl`)

```hcl
locals {
  # Region(en)
  aws_region = "eu-central-1"

  # Umgebung aus Pfad ableiten (dev / staging / prod)
  environment = regex("[^/]+$", replace(path_relative_to_include(), "\\", "/"))

  # Mapping Umgebung -> AWS Account ID
  # Anfang: alle auf denselben Account zeigen
  account_map = {
    dev     = "111111111111"
    staging = "111111111111"
    prod    = "111111111111"
  }

  aws_account_id = local.account_map[local.environment]

  # Remote State Bucket pro Umgebung (oder pro Account)
  state_bucket_map = {
    dev     = "company-dev-tfstate"
    staging = "company-staging-tfstate"
    prod    = "company-prod-tfstate"
  }

  state_bucket = local.state_bucket_map[local.environment]
}

remote_state {
  backend = "s3"
  config = {
    bucket         = local.state_bucket
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = local.aws_region
    dynamodb_table = "terraform-locks-${local.environment}"
    encrypt        = true
  }
}

# Provider-Block für alle Stacks generieren (OpenTofu + AWS Provider)
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

provider "aws" {
  region = "${local.aws_region}"

  assume_role {
    role_arn     = "arn:aws:iam::${local.aws_account_id}:role/terraform-deploy-role"
    session_name = "terragrunt-${local.environment}"
  }
}
EOF
}
```

**Was bringt das?**

* Anfangs zeigen **alle** Umgebungen auf denselben Account.
* Sobald du z. B. `dev` in einen neuen Account verschieben willst, änderst du **nur** `account_map.dev` und ggf. den Bucket in `state_bucket_map.dev`.
* Verzeichnisstruktur bleibt identisch, nur das Zielkonto & Backend wechseln.

---

## 3. Beispiel-Hierarchie in `live/`

### Environment-Ebene (z. B. `live/eu-central-1/dev/terragrunt.hcl`)

```hcl
include "root" {
  path = find_in_parent_folders()
}

locals {
  environment = "dev"
}

inputs = {
  environment = local.environment
  aws_region  = "eu-central-1"
}
```

(Dieses File ist optional, aber hilfreich, um env-spezifische Defaults / Tags etc. zu definieren.)

### VPC-Stack (z. B. `live/eu-central-1/dev/vpc/terragrunt.hcl`)

```hcl
include "env" {
  path = find_in_parent_folders("terragrunt.hcl")
}

terraform {
  source = "../../../modules/network/vpc"
}

inputs = {
  name               = "dev-main"
  cidr_block         = "10.10.0.0/16"
  az_count           = 3
  enable_nat_gateways = true

  tags = {
    Environment = "dev"
    ManagedBy   = "terragrunt"
  }
}
```

### Modul-Beispiel (OpenTofu) `modules/network/vpc/main.tf`

```hcl
variable "name" {
  type = string
}

variable "cidr_block" {
  type = string
}

variable "az_count" {
  type    = number
  default = 2
}

variable "enable_nat_gateways" {
  type    = bool
  default = false
}

variable "tags" {
  type    = map(string)
  default = {}
}

data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = merge(var.tags, {
    Name = var.name
  })
}

# Subnets etc...
```

(Alle Module bleiben **account-agnostisch**.)

---

## 4. Migration in eigene AWS Accounts – wie läuft das ab?

Nehmen wir an, du willst in Zukunft:

* `dev` → Account `222222222222`
* `staging` → Account `333333333333`
* `prod` → bleibt erstmal in `111111111111`

### Schritt 1 – Neue Accounts & IAM Rolle

* Über AWS Organizations neue Accounts anlegen.
* In jedem neuen Account:

  * IAM Rolle `terraform-deploy-role` mit Vertrauensbeziehung für deinen CI-User oder deine AWS-Profile.
  * S3-Bucket & DynamoDB-Tabelle für Remote-State anlegen (kannst du auch mit einem kleinen Bootstrap-Stack machen).

### Schritt 2 – `account_map` & `state_bucket_map` anpassen

Im Root-`terragrunt.hcl`:

```hcl
locals {
  account_map = {
    dev     = "222222222222"
    staging = "333333333333"
    prod    = "111111111111"
  }

  state_bucket_map = {
    dev     = "company-dev-tfstate-new"
    staging = "company-staging-tfstate-new"
    prod    = "company-prod-tfstate"
  }
}
```

Damit zeigen alle `dev`-Stacks jetzt auf den neuen Account.

### Schritt 3 – State-Migration

Pro Stack (z. B. `live/eu-central-1/dev/vpc`):

```bash
cd live/eu-central-1/dev/vpc
terragrunt init -migrate-state
```

Damit wird der State vom alten Bucket in den neuen Bucket verschoben.
(Je nach Setup musst du evtl. den alten und neuen Backend-Config einmalig anpassen – aber durch Terragrunt ist das zentral.)

> **Alternative:** Wenn du die Ressourcen wirklich *neu* im neuen Account anlegen möchtest, kannst du statt State-Migration auch:
>
> * im neuen Account `terragrunt apply` ausführen
> * alten Account später `destroy`en.

### Schritt 4 – CI/CD & AWS Credentials

* In deinem GitHub Actions / CI-System pro Umgebung **eigenes AWS-Role- oder Profile-Setup**:

  * `AWS_ROLE_ARN` oder OpenID Connect Role pro Umgebung.
* Terragrunt nutzt dann dieselbe Codebasis, aber mit anderen Rollen/Accounts je nach Environment.

---

## 5. Praktische Tipps für dein Setup

* **Naming & Tagging:**

  * Alle Ressourcen mit `Environment`, `Application`, `Owner` taggen → späteres Aufräumen & Billing viel einfacher.
* **Feature-Toggles per Umgebung:**

  * Dinge wie NAT Gateways, Backups, Logging-Level etc. per `inputs` in den env-spezifischen Terragrunt-Files steuern.
* **State-Isolation:**

  * Pro Umgebung mindestens ein eigener S3-Bucket oder mindestens eigenes Prefix + Lock-Tabelle – du bist schon “Account-ready”.
* **Ordner = Stack-Grenze:**

  * Je Ordner nur ein thematisch überschaubarer Stack (z. B. `vpc`, `shared-services`, `app1`) → einfachere Plans & Rollbacks.

---

Wenn du möchtest, kann ich dir im nächsten Schritt:

* ein **konkretes Beispiel-Repo** mit vollständiger Struktur (inkl. `network`, `rds`, `eks` etc.) ausformulieren
* oder eine **GitHub Actions Pipeline** skizzieren, die automatisch `dev`, `staging` und `prod` mit Terragrunt + OpenTofu deployed.
