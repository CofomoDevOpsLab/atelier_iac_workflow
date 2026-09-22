# Atelier : workflow de travail Git + Pulumi

Cet atelier fait pratiquer le cycle complet d'une contribution infra-as-code :
connexion aux backends, branche Git, changement de code, `preview`/`up`
Pulumi, puis Merge Request GitLab.

Projet GitLab : `https://gitlab.com/curateur1/poc/atelier_iac/workflow.git`

## Vue d'ensemble du workflow

```
 0. pulumi login --local     ┐
    az login                 ┘  pré-étape (connexion aux backends)
 1. git clone
 2. git checkout -b <branche>
 3. éditer __main__.py (ajout d'un Resource Group)
 4. pulumi preview
 5. pulumi up                    déploie sur Azure
 6. git add / commit / push
 7. Merge Request GitLab -> revue -> merge dans main
```

## Prérequis

- [Pulumi CLI](https://www.pulumi.com/docs/install/)
- [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) (`az`)
- Python 3.9+, [Poetry](https://python-poetry.org/docs/#installation) et `git`
- Un accès au projet GitLab `atelier_iac/workflow` (clone + droit de push)
- Un abonnement Azure sur lequel tu as le droit de créer un resource group

## Étape 0 — Pré-étape : stack local Pulumi + connexion Azure

Deux connexions sont nécessaires avant de toucher au code : le **backend Pulumi**
(où est stocké l'état du stack) et **Azure** (où seront créées les ressources).

```bash
# Backend Pulumi en local (état stocké sous ~/.pulumi, pas de compte Pulumi Cloud requis)
pulumi login --local

# Connexion au compte Azure (ouvre une fenêtre de connexion)
az login

# Optionnel si plusieurs abonnements : sélectionner le bon
az account set --subscription "<nom-ou-id-de-l-abonnement>"
```

Ces deux commandes ne sont à refaire que si la session expire — inutile de les
relancer à chaque changement de code.

## Étape 1 — Cloner le projet

```bash
git clone https://gitlab.com/curateur1/poc/atelier_iac/workflow.git
cd workflow
```

## Étape 2 — Créer une branche

```bash
git checkout -b feature/ajout-resource-group
```

## Étape 3 — Initialiser le stack `atelier` (première fois seulement)

```bash
pulumi stack init atelier
# ou, si le stack existe déjà :
pulumi stack select atelier

# Installe les dépendances via Poetry (toolchain natif configuré dans Pulumi.yaml,
# Pulumi appelle Poetry et gère lui-même l'environnement virtuel — rien à activer)
pulumi install

pulumi config set azure-native:location canadacentral
```

## Étape 4 — Faire le changement : ajouter un Resource Group

Ouvre `__main__.py` et remplace le contenu par :

```python
import pulumi
from pulumi_azure_native import resources

config = pulumi.Config()
rg_name = config.get("resourceGroupName") or "rg-atelier-pulumi"

resource_group = resources.ResourceGroup(
    "atelier-rg",
    resource_group_name=rg_name,
)

pulumi.export("resourceGroupName", resource_group.name)
```

C'est le seul changement de code de l'atelier : un `ResourceGroup` Azure,
exporté en sortie de stack.

## Étape 5 — Preview

```bash
pulumi preview
```

Vérifie que le plan annonce bien **1 to create** (le resource group), et rien
d'autre.

## Étape 6 — Déployer

```bash
pulumi up
```

Confirme avec `yes`. Pulumi crée le resource group sur Azure et affiche la
sortie `resourceGroupName`.

Vérification côté Azure (optionnel) :

```bash
az group show --name rg-atelier-pulumi
```

## Étape 7 — Committer et pousser la branche

```bash
git add __main__.py
git commit -m "Ajout d'un resource group Azure dans le stack atelier"
git push -u origin feature/ajout-resource-group
```

`git push` affiche un lien direct pour créer la Merge Request — sinon,
continue à l'étape 8.

## Étape 8 — Créer la Merge Request vers GitLab

Via l'interface web :

1. Ouvrir `https://gitlab.com/curateur1/poc/atelier_iac/workflow/-/merge_requests`
2. **New merge request**
3. Source : `feature/ajout-resource-group` → Cible : `main`
4. Décrire le changement (ajout du resource group dans le stack `atelier`) puis **Create merge request**

Via `glab` CLI (si installé) :

```bash
glab mr create --source-branch feature/ajout-resource-group --target-branch main \
  --title "Ajout d'un resource group Azure dans le stack atelier" \
  --description "Ajoute un azure_native.resources.ResourceGroup dans __main__.py, testé via pulumi preview/up sur le stack atelier."
```

## Nettoyage (important)

`pulumi up` crée une vraie ressource facturée sur Azure. Une fois l'atelier
terminé, détruis-la si elle n'a pas vocation à rester :

```bash
pulumi destroy
```
