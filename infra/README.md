Hier ist eine einfache, saubere und praxiserprobte Vorgehensweise, wie du die komplette Terragrunt-/OpenTofu-Ordnerstruktur **in Git initialisierst** – Schritt für Schritt, inklusive der nötigen `git`-Befehle.

---

# ✅ 1. Projekt-Verzeichnis lokal anlegen

```bash
mkdir infra
cd infra
```

---

# ✅ 2. Basis-Ordnerstruktur erstellen

```bash
mkdir -p modules \
         live/eu-central-1/dev \
         live/eu-central-1/staging \
         live/eu-central-1/prod
```

Beispiel (leer):

```
infra/
  modules/
  live/
    eu-central-1/
      dev/
      staging/
      prod/
```

---

# ✅ 3. Unterordner für Module anlegen

Beispiel: Netzwerk und Security Module:

```bash
mkdir -p modules/network/vpc
mkdir -p modules/security/iam-roles
mkdir -p modules/compute/ecs
```

---

# ✅ 4. Terragrunt-Struktur anlegen

**Root-Konfiguration**

```bash
touch terragrunt.hcl
```

**Umgebungsebene**

```bash
touch live/eu-central-1/dev/terragrunt.hcl
touch live/eu-central-1/staging/terragrunt.hcl
touch live/eu-central-1/prod/terragrunt.hcl
```

**Beispiel-Stacks**

```bash
mkdir -p live/eu-central-1/dev/vpc
touch    live/eu-central-1/dev/vpc/terragrunt.hcl

mkdir -p live/eu-central-1/dev/app
touch    live/eu-central-1/dev/app/terragrunt.hcl
```

(Entsprechendes für `staging` & `prod`.)

---

# ✅ 5. README anlegen

```bash
touch README.md
```

---

# ✅ 6. Git initialisieren

### Falls noch kein Git-Repo:

```bash
git init
```

---

# ✅ 7. `.gitignore` anlegen

Sehr wichtig für Terragrunt/Tofu (keine lokalen States oder Crash Logs einchecken):

```bash
cat <<EOF > .gitignore
# Terraform / OpenTofu
*.tfstate
*.tfstate.*
.crash
*.backup
*.log

# Terragrunt cache
.terragrunt-cache/

# IDE
.idea/
.vscode/

# OS
.DS_Store
EOF
```

---

# ✅ 8. Dateien zum Git-Index hinzufügen

```bash
git add .
```

---

# ✅ 9. Ersten Commit erstellen

```bash
git commit -m "Initial Terragrunt/OpenTofu project structure"
```

---

# ✅ 10. Remote-Repository in GitHub anlegen

### Variante A: Du hast das Repo schon in GitHub erstellt

```bash
git remote add origin git@github.com:DEINUSER/infra.git
git push -u origin main
```

### Variante B: Repo existiert noch nicht

→ GitHub → *New Repository* → Name: **infra**

Nach Anlegen:

```bash
git remote add origin git@github.com:DEINUSER/infra.git
git push -u origin main
```

---

# ✔️ Ergebnis

Dein Git-Repository enthält:

```
infra/
  terragrunt.hcl
  modules/
    network/vpc/
    security/iam-roles/
    compute/ecs/
  live/
    eu-central-1/
      dev/
        terragrunt.hcl
        vpc/terragrunt.hcl
        app/terragrunt.hcl
      staging/
        terragrunt.hcl
      prod/
        terragrunt.hcl
  README.md
  .gitignore
```

Die Struktur ist sofort einsatzbereit für Terragrunt & OpenTofu.

---

# Möchtest du, dass ich dir die **komplette Dateienbasis automatisch generiere**, inklusive:

* fertigem `terragrunt.hcl` Root-Level
* Beispiel-VPC-Modul (OpenTofu)
* Beispiel-App-Modul
* fertigen Umgebungs-Configs (dev/staging/prod)
* vorbereiteter GitHub Actions Pipeline?

Dann kann ich dir ein voll lauffähiges Beispiel-Repo erzeugen.
