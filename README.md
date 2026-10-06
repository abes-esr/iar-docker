# 🐳 iar-docker : Orchestration & Fiche d'Exploitation de la Plateforme IAR

[![Docker Pulls](https://img.shields.io/docker/pulls/abesesr/iar.svg)](https://hub.docker.com/r/abesesr/iar/)

Configuration Docker Compose et **fiche d'exploitation standard** de la plateforme **IAR** (**I**ndexation **A**utomatique **R**AMEAU) conçue et maintenue par l'**ABES** (Agence Bibliographique de l'Enseignement Supérieur). Ce document est conforme à la [politique informatique de l'ABES](https://politique-informatique.abes.fr/docs/dev/documentation/#documentation-administrateur--fiche-dexploitation).

---

## 📌 Sommaire

- [1. Description brève du projet](#1-description-brève-du-projet)
- [2. Liens vers les autres dépôts du projet](#2-liens-vers-les-autres-dépôts-du-projet)
- [3. Architecture de la pile, Services & Points d'accès](#3-architecture-de-la-pile-services--points-daccès)
- [4. Explications des variables d'environnement (`.env`)](#4-explications-des-variables-denvironnement-env)
- [5. Diagramme d'architecture](#5-diagramme-darchitecture)
- [6. Procédure de déploiement & Cycle de vie de la plateforme](#6-procédure-de-déploiement--cycle-de-vie-de-la-plateforme)
- [7. Procédure de supervision](#7-procédure-de-supervision)
- [8. Procédure de restauration globale](#8-procédure-de-restauration-globale)

---

## 1. Description brève du projet

Le dépôt **`iar-docker`** constitue le cœur d'orchestration d'infrastructure et d'exploitation de la plateforme **IAR** (**I**ndexation **A**utomatique **R**AMEAU).

C'est l'**unique composant déployé sur les serveurs hôtes de l'ABES** (`donut-test` et `donut-prod` dans le répertoire d'exploitation `/opt/pod/iar-docker/`). Il orchestre l'ensemble des conteneurs interconnectés via Docker Compose : l'API d'inférence FastAPI (`iar-api`), la base vectorielle dense (`iar-qdrant`), le moteur LLM local (`iar-llm`) et le microservice de vectorisation (`iar-vectorisation`).

Ce dépôt centralise la configuration d'infrastructure, le montage des volumes persistants, l'allocation des ressources matérielles (GPU/RAM/CPU), ainsi que les procédures de déploiement, de supervision et de restauration de la plateforme.

---

## 2. Liens vers les autres dépôts du projet

La plateforme IAR est découpée en plusieurs modules complémentaires hébergés sur l'organisation GitHub [abes-esr](https://github.com/abes-esr) :

| Dépôt GitHub                                                           | Rôle & Description                                                                                                                  |
| :--------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------- |
| [**iar-docker**](https://github.com/abes-esr/iar-docker)               | **Ce dépôt d'infrastructure** : Orchestration Docker Compose et exploitation globale sur les serveurs `donut-test` et `donut-prod`. |
| [**iar-api**](https://github.com/abes-esr/iar-api)                     | Service web d'inférence FastAPI exposant les endpoints de suggestion d'indexation sujet RAMEAU.                                     |
| [**iar-vectorisation**](https://github.com/abes-esr/iar-vectorisation) | Pipeline de calcul vectoriel (embeddings Sentence-Transformers) et peuplement des collections Qdrant.                               |
| [**iar-batch-docker**](https://github.com/abes-esr/iar-batch-docker)   | Environnement Docker d'orchestration pour l'exécution des traitements par lots et interfaces de mise à jour.                        |
| [**iar-batch-dump**](https://github.com/abes-esr/iar-batch-dump)       | Extraction, transformation et génération des dumps de données d'autorités RAMEAU et notices bibliographiques Sudoc.                 |
| [**iar-script-winibw**](https://github.com/abes-esr/iar-script-winibw) | Script client (VBScript) s'intégrant au logiciel de catalogage **WinIBW** pour interroger l'API depuis le poste des catalogueurs.   |

---

## 3. Architecture de la pile, Services & Points d'accès

### Tableau des services & Points d'accès (Swagger / Dashboards)

| Service                 | Rôle & Fonction                                                       | Environnement de Test (`donut-test`)       | Environnement de Production (`donut-prod`) |
| :---------------------- | :-------------------------------------------------------------------- | :----------------------------------------- | :----------------------------------------- |
| **`iar-api`**           | API REST FastAPI de suggestion d'indexation sujet (Swagger UI)        | `http://donut-test.abes.fr:8071/docs`      | `http://donut-prod.abes.fr:8071/docs`      |
| **`iar-vectorisation`** | Pipeline de calcul d'embeddings et ingestion vectorielle (Swagger UI) | `http://donut-test.abes.fr:8100/docs`      | `http://donut-prod.abes.fr:8100/docs`      |
| **`iar-qdrant`**        | Base de données vectorielle dense (Dashboard web)                     | `http://donut-test.abes.fr:6333/dashboard` | `http://donut-prod.abes.fr:6333/dashboard` |
| **`iar-llm`**           | Moteur d'inférence LLM local Llama 3.1 (filtrage contextuel)          | _(API interne conteneur :11434)_           | _(API interne conteneur :11434)_           |

### Réseau interne Docker (Bridge)

Les conteneurs communiquent à travers un réseau bridge dédié `${IAR_DOCKER_NETWORK}` (par défaut `iar-network`) :

- **Résolution DNS automatique** : Les conteneurs s'appellent par leur nom de service (ex. `http://iar-qdrant:6333` ou `http://iar-llm:11434`).
- **Isolation sécurisée** : Les échanges internes d'inférence (FastAPI ↔ Qdrant et FastAPI ↔ Ollama) transitent de manière étanche au sein du réseau Docker bridge.

### Volumes persistants montés sur l'hôte

Tous les volumes applicatifs sont rattachés au répertoire local `./volumes` (ou chemin spécifié par `IAR_DOCKER_VOLUME_BIND`), monté dans `/app/data` :

- [`./volumes/csv/`](./volumes/csv/) : Référentiels d'autorités RAMEAU sous forme tabulaire (`rameau_ancestors_df.csv`, `rameau_parents.csv`, `rameau_subdivisionsONLY.csv`).
- [`./volumes/pkl/`](./volumes/pkl/) : Caches sérialisés Python (fichiers `.pkl`) contenant les graphes, hiérarchies et dictionnaires RAMEAU précalculés.
- [`./volumes/history/`](./volumes/history/) : Historique et traçabilité des requêtes d'indexation soumises à l'API.
- [`./volumes/responses/`](./volumes/responses/) : Archivage et sauvegarde des réponses d'inférence générées (formats JSON / Unimarc).
- [`./volumes/qdrant/storage/`](./volumes/qdrant/) : Stockage physique permanent des vecteurs d'embeddings, segments et index HNSW de Qdrant (`/qdrant/storage`).
- [`./volumes/ollama/`](./volumes/ollama/) : Poids des modèles de langage téléchargés (`/root/.ollama`), évitant tout retéléchargement au redémarrage.

---

## 4. Explications des variables d'environnement (`.env`)

La configuration est centralisée dans le fichier [`.env`](./.env), créé à partir du gabarit [`.env-dist`](./.env-dist). Seules les variables nécessaires à l'orchestration des conteneurs et au dimensionnement des ressources système sont listées ci-dessous :

| Variable                        | Description & Rôle                                                       | Exemple de valeur                            |
| :------------------------------ | :----------------------------------------------------------------------- | :------------------------------------------- |
| **`MEM_LIMIT`**                 | Limite maximale de mémoire vive autorisée par conteneur                  | `8g`                                         |
| **`CPU_LIMIT`**                 | Nombre maximal de cœurs CPU alloués par conteneur                        | `8`                                          |
| **`ENABLE_GPU`**                | Activation du support matériel GPU NVIDIA (`true` ou `false`)            | `false` ou `true`                            |
| **`NVIDIA_VISIBLE_DEVICES`**    | Identifiant(s) des cartes GPU accessibles aux conteneurs                 | `all`                                        |
| **`IAR_API_VERSION`**           | Tag Docker Hub de l'image `abesesr/iar` pour le service API              | `test-api` ou `prod-api`                     |
| **`IAR_VECTORISATION_VERSION`** | Tag Docker Hub de l'image `abesesr/iar` pour le service de vectorisation | `test-vectorisation` ou `prod-vectorisation` |
| **`IAR_QDRANT_VERSION`**        | Version de l'image officielle Qdrant                                     | `v1.19.1` ou `latest`                        |
| **`IAR_API_HTTP_PORT`**         | Port d'écoute hôte exposé pour l'API FastAPI                             | `8071`                                       |
| **`IAR_VECTORISATION_PORT`**    | Port d'écoute hôte exposé pour le service de vectorisation               | `8100`                                       |
| **`IAR_QDRANT_PORT`**           | Port d'écoute hôte exposé pour la base Qdrant                            | `6333`                                       |
| **`IAR_LLM_PORT`**              | Port d'écoute hôte exposé pour le serveur LLM                            | `11434`                                      |
| **`IAR_DOCKER_NETWORK`**        | Nom du réseau Docker externe bridge reliant les conteneurs               | `iar-network`                                |
| **`IAR_DOCKER_VOLUME_BIND`**    | Chemin racine des volumes montés sur l'hôte                              | `./volumes`                                  |

> ⚠️ **Sécurité** : Le fichier `.env` contient des configurations spécifiques à la machine et des clés sensibles. Il ne doit **jamais être commité** sur GitHub et est ignoré par le [`.gitignore`](./.gitignore).

---

## 5. Diagramme d'architecture

```text
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                CLIENTS, BATCHS & SUPERVISION                                     │
│                                                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────────────┐  ┌─────────────────────┐  │
│  │ Client WinIBW    │  │ Navigateur Web   │  │ iar-batch-dump        │  │ Supervision ABES    │  │
│  │(Catalogage Sudoc)│  │(Catalogueur/Adm.)│  │ (diplotaxis*.abes.fr) │  │(diplotaxis7-*.abes) │  │
│  └────────┬─────────┘  └────────┬─────────┘  └───────────┬───────────┘  └──────────▲──────────┘  │
└───────────┼─────────────────────┼────────────────────────┼─────────────────────────┼─────────────┘
            │ HTTP :8071          │ HTTP :8071 / :6333     │ HTTP POST :8100         │ Logs Filebeat
            │ (Indexation)        │ / :29999               │ (/init|update/upload)   │ & Métriques
            ▼                     ▼                        │ (Dumps CSV RAMEAU)      │
┌──────────────────────────────────────────────────────────┼─────────────────────────┼─────────────┐
│ SERVEUR HÔTE ABES (donut-test.abes.fr / donut-prod.abes.fr)                        │             │
│ Répertoire d'exploitation : /opt/pod/iar-docker/         │                         │             │
│                                                          │                         │             │
│  PORTS COMPOSE EXPOSÉS :  :8071 (API)   :6333 (Qdrant)   │    :11434 (LLM)   :8100 (Vect.)       │
│  ────────────────────────────────────────────────────────┼────────────────────────────────────   │
│                                                          │                                       │
│  PILE DOCKER COMPOSE iar-docker (Réseau bridge : iar-network)                                    │
│                                                          │                                       │
│  ┌───────────────────────┐             ┌─────────────────┼──────┐                                │
│  │       iar-api         │◄────gRPC────┤       iar-qdrant│      │                                │
│  │ (abesesr/iar:*-api)   │──Recherche─►│    (qdrant/qdrant)     │                                │
│  │ Port interne : 8071   │    :6333    │ Port interne : 6333    │                                │
│  └──────────▲────────────┘             └───────────▲─────┼──────┘                                │
│             │                                      │     │                                       │
│             │ Filtrage sémantique                  │     │ Téléversement CSV                     │
│             │ & Inférence LLM                      │     ▼ /init/upload /update/upload           │
│             │                                      │                                             │
│  ┌──────────┴────────────┐             ┌───────────┴────────────┐                                │
│  │        iar-llm        │             │   iar-vectorisation    │                                │
│  │    (ollama/ollama)    │             │(abesesr/iar:*-vector.) │                                │
│  │ Port interne : 11434  │             │ Port interne : 8100    │                                │
│  └───────────────────────┘             └───────────┬────────────┘                                │
│                                                    │                                             │
│  ──────────────────────────────────────────────────┼───────────────────────────────────────────  │
│  SERVICE D'INFRASTRUCTURE HÔTE (HORS COMPOSE)      │                                             │
│                                                    │                                             │
│  ┌───────────────────────────────────────────────┐ │                                             │
│  │ dozzle (logs hôte) :29999                     │ │                                             │
│  │ Surveillance directe du socket Docker hôte    │ │                                             │
│  └──────────────────────┬────────────────────────┘ │                                             │
│                         │                          │                                             │
│  ───────────────────────┼──────────────────────────┼───────────────────────────────────────────  │
│  VOLUMES PERSISTANTS HÔTE (/opt/pod/iar-docker/volumes/)                                         │
│                         │                          │                                             │
│   ├── volumes/csv/      │ ──► [/app/data/csv] Référentiels RAMEAU (ancestors, parents, subs)     │
│   ├── volumes/pkl/      │ ──► [/app/data/pkl] Graphes et dictionnaires sérialisés (.pkl)         │
│   ├── volumes/history/  │ ──► [/app/data/history] Historique des requêtes d'inférence            │
│   ├── volumes/responses/│ ──► [/app/data/responses] Archivage des réponses générées              │
│   ├── volumes/qdrant/   │ ──► [/qdrant/storage] Collections HNSW & vecteurs                      │
│   └── volumes/ollama/   │ ──► [/root/.ollama] Poids du modèle LLM Llama 3.1                      │
│                                                                                                  │
└────────────────────────────────────────────┬─────────────────────────────────────────────────────┘
                                             │
                                             │ Sauvegarde automatisée rsync (SIAT ABES)
                                             ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│ SERVEURS DE SAUVEGARDE ABES (socorro.abes.fr / sotora.abes.fr)                                   │
│ Miroir complet : /backup/donut-prod/opt/pod/iar-docker/                                          │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Procédure de déploiement & Cycle de vie de la plateforme

### Déploiement initial sur `/opt/pod/iar-docker/`

1. **Connexion au serveur hôte** en SSH avec un compte disposant des droits nécessaires.
2. **Clonage du dépôt d'infrastructure** dans le répertoire dédié :
   ```bash
   cd /opt/pod
   git clone https://github.com/abes-esr/iar-docker.git
   cd /opt/pod/iar-docker/
   ```
3. **Création du réseau Docker** s'il n'existe pas déjà :
   ```bash
   docker network create iar-network 2>/dev/null || true
   ```
4. **Initialisation de la configuration** :
   ```bash
   cp .env-dist .env
   # Éditer .env pour adapter les tags d'images et allouer les ressources matérielles
   ```
5. **Lancement de la plateforme** :
   ```bash
   docker compose up -d
   ```

### Déploiement continu & WUD (What's Up Docker?)

Conformément à la politique de développement de l'ABES, la mise à jour des conteneurs peut être automatisée grâce à l'outil **WUD** surveillant les labels Docker positionnés dans [`docker-compose.yml`](./docker-compose.yml) :

```yaml
labels:
  - "wud.watch=true"
  - "wud.watch.digest=true"
```

Lorsqu'un développeur pousse son code sur `develop` (pour `test`) ou publie une release sur `main` (pour `prod`), la GitHub Action associée compile et pousse la nouvelle image sur Docker Hub (`abesesr/iar`). WUD détecte la modification du digest, télécharge la nouvelle image et relance le conteneur automatiquement sans interruption majeure de service.

---

## 7. Procédure de supervision

### 1. Suivi de l'état des conteneurs

Contrôler que l'ensemble des services est au statut `Up` :

```bash
cd /opt/pod/iar-docker/
docker compose ps
```

### 2. Consultation des journaux d'événements (logs)

Pour suivre les logs en direct d'un ou plusieurs services :

```bash
# Tous les services
docker compose logs -f --tail=100

# Service spécifique (ex: API ou Qdrant)
docker compose logs -f --tail=100 iar-api
docker compose logs -f --tail=100 iar-qdrant
```

### 3. Consultation visuelle via Dozzle (Service d'infrastructure hôte)

Sur les serveurs `donut-test` et `donut-prod`, l'outil **Dozzle** est préinstallé au niveau système et écoute sur le port `29999` pour surveiller l'ensemble des conteneurs via le socket Docker de l'hôte :

- `http://donut-test.abes.fr:29999/` (Test)
- `http://donut-prod.abes.fr:29999/` (Production)

Il permet de filtrer, chercher et inspecter en direct les flux d'erreurs de tous les conteneurs sans avoir à ouvrir de session SSH.

### 4. Puits de logs centralisé de l'ABES

Les conteneurs sont configurés avec les labels d'export vers le puits de logs Kibana de l'ABES via Filebeat :

```yaml
labels:
  - "co.elastic.logs/enabled=true"
  - "co.elastic.logs/processors.add_fields.fields.abes_appli=iar"
  - "co.elastic.logs/processors.add_fields.fields.abes_middleware=fastapi"
```

### 5. Sondes de santé HTTP (Healthchecks)

Interroger les sondes locales ou distantes :

```bash
# Santé API (port 8071)
curl -s -f http://localhost:8071/health || echo "ERREUR API"
# Distant : http://donut-test.abes.fr:8071/health (Test) / http://donut-prod.abes.fr:8071/health (Prod)

# Santé Vectorisation (port 8100)
curl -s -f http://localhost:8100/health || echo "ERREUR Vectorisation"
# Distant : http://donut-test.abes.fr:8100/health (Test) / http://donut-prod.abes.fr:8100/health (Prod)

# Santé Qdrant (port 6333)
curl -s -f http://localhost:6333/healthz || echo "ERREUR Qdrant"
# Distant : http://donut-test.abes.fr:6333/healthz (Test) / http://donut-prod.abes.fr:6333/healthz (Prod)

# Santé Serveur LLM (port 11434)
curl -s -f http://localhost:11434/api/tags || echo "ERREUR LLM"
# Distant : http://donut-test.abes.fr:11434/api/tags (Test) / http://donut-prod.abes.fr:11434/api/tags (Prod)
```

### 6. Métriques d'infrastructure Grafana

L'utilisation CPU, GPU, RAM et I/O disque des serveurs `donut-test` et `donut-prod` est monitorée en continu sur le serveur Grafana de l'ABES :

- `http://diplotaxis7-*.abes.fr:3000`

---

## 8. Procédure de restauration globale

> 📌 **Rôle central du dépôt** : Le dépôt `iar-docker` centralise l'état et le stockage de tous les services de la plateforme. En cas d'incident majeur ou de réinstallation complète d'un serveur, la restauration est opérée depuis ce dossier.

### Étape 1 : Récupération du miroir global via `rsync`

Les serveurs de sauvegarde de l'ABES (`socorro.abes.fr` et `sotora.abes.fr`) effectuent un snapshot complet et quotidien du répertoire `/opt/pod/iar-docker/`.

```bash
# 1. Arrêter la stack si des conteneurs tournent encore
cd /opt/pod/iar-docker && docker compose down 2>/dev/null || true

# 2. Restaurer l'arborescence complète depuis le serveur de sauvegarde
rsync -avzP socorro.abes.fr:/backup/donut-prod/opt/pod/iar-docker/ /opt/pod/iar-docker/
# (ou depuis sotora.v104.abes.fr selon la rétention demandée)
```

### Étape 2 : Contrôle des éléments indispensables

Vérifier la présence du fichier d'environnement et des fichiers CSV d'autorités RAMEAU indispensables dans [`./volumes/csv/`](./volumes/csv/) :

- `rameau_ancestors_df.csv`
- `rameau_parents.csv`
- `rameau_subdivisionsONLY.csv`

_(Les caches `.pkl`, historiques et réponses archivées ne sont pas bloquants pour le fonctionnement du service)._

```bash
# 1. Vérifier la présence du fichier d'environnement
test -f /opt/pod/iar-docker/.env && echo "OK: .env présent" || echo "ERREUR: .env manquant"

# 2. Vérifier la présence des CSV d'autorités RAMEAU indispensables
ls -lh /opt/pod/iar-docker/volumes/csv/rameau_ancestors_df.csv \
       /opt/pod/iar-docker/volumes/csv/rameau_parents.csv \
       /opt/pod/iar-docker/volumes/csv/rameau_subdivisionsONLY.csv
```

### Étape 3 : Relance et validation de la plateforme

```bash
cd /opt/pod/iar-docker

# Créer le réseau Docker externe iar-network (|| true s'il existe déjà)
docker network create iar-network || true

# Démarrer la pile et vérifier l'état
docker compose up -d
docker compose ps
```

### Étape 4 (Si nécessaire) : Commandes complémentaires (Qdrant & LLM)

Si les volumes Qdrant ou Ollama nécessitent une réinitialisation manuelle ciblée :

- **Restauration d'une collection Qdrant par snapshot API** (si restauration d'un export `.snapshot` isolé) :

  ```bash
  # Exemple avec la collection concepts_allMin_only_mono :
  curl -X POST -F "snapshot=@/opt/pod/iar-docker/volumes/qdrant/snapshots/concepts_allMin_only_mono.snapshot" \
    http://localhost:6333/collections/concepts_allMin_only_mono/snapshots/upload
  ```

- **Téléchargement du modèle LLM** (si `volumes/ollama/` non restauré) :
  ```bash
  docker compose up -d iar-llm
  docker compose exec iar-llm ollama pull llama3.1
  ```
