# iar-docker

Configuration docker 🐳 pour déployer l'application **iar** (IA Rameau). Ce README fait également office de fiche d'exploitation.

## Prérequis

Disposer de :

- `docker`
- `docker-compose` (ou plugin `docker compose`)

## Installation

Déployer la configuration docker dans un répertoire :

```bash
# adaptez /opt/pod/ avec l'emplacement où vous souhaitez déployer l'application
cd /opt/pod/
git clone https://github.com/abes-esr/iar-docker.git
```

Configurer l'application depuis l'exemple du [fichier `.env-dist`](./.env-dist) (ce fichier contient la liste des variables avec des explications et des exemples de valeurs) :

```bash
cd /opt/pod/iar-docker/
cp .env-dist .env
# personnaliser alors le contenu du .env
```

**Note : La clé d'API LLM (`IAR_LLM_KEY`) n'est pas renseignée par défaut. Vous devez impérativement la renseigner dans le fichier `.env` (ex: avec nano ou vim), ainsi que vérifier les versions des images et les répertoires de données.**

Démarrer l'application :

```bash
cd /opt/pod/iar-docker/
docker compose up -d
```

## Démarrage et arrêt

Pour démarrer l'application :

```bash
cd /opt/pod/iar-docker/
docker compose up -d
```

Pour arrêter l'application :

```bash
cd /opt/pod/iar-docker/
docker compose stop
```

## Supervision

Pour vérifier que l'application est démarrée, on peut consulter l'état des conteneurs :

```bash
cd /opt/pod/iar-docker/
docker compose ps
```

Pour vérifier que l'application est bien lancée, on peut consulter les logs :

```bash
cd /opt/pod/iar-docker/
docker compose logs --tail=50 -f
```

À noter que ces logs sont envoyées automatiquement au puits de log de l'Abes à l'aide du client Filebeat installé sur le nœud Docker et grâce aux labels configurés (`abes_appli=iar`).

## Sauvegardes

Pour sauvegarder l'application, il faut :

- **Sauvegarder les données persistantes** :
  - Les données de la base vectorielle Qdrant (dossier `./qdrant_data` ou `./volumes/qdrant/storage`).
  - Les fichiers de données / dumps (répertoire défini par `IAR_DATA_DIR`, par exemple `./data` ou `./volumes/csv`, `./volumes/pkl`).
- **Sauvegarder le fichier de configuration** :
  - Le fichier `/opt/pod/iar-docker/.env` qui est non versionné et qui permet de configurer l'ensemble des conteneurs (les autres fichiers étant versionnés sur le dépôt GitHub `iar-docker`).

Ces opérations sont prises en charge dans la politique de sauvegarde du SIAT (sauvegardes régulières sur sotora/socorro).

## Restauration

### Réinstallation de l'application

- Se connecter avec son compte développeur sur la machine de déploiement (via SSH) :

- Se positionner dans le répertoire des applications :

```bash
cd /opt/pod
```

- Récupérer le projet `iar-docker` et se positionner dans le répertoire :

```bash
git clone https://github.com/abes-esr/iar-docker.git
cd iar-docker
```

- Récupérer le fichier `.env` depuis le serveur de sauvegarde sotora (authentification nécessaire) :

```bash
rsync -av devel@sotora.v104.abes.fr:/backup_pool/<nom-machine>/daily.0/racine/opt/pod/iar-docker/.env /opt/pod/iar-docker/.env
```

- Si nécessaire, restaurer les données des volumes (Qdrant et données de dump/vectorisation).

- Lancer les conteneurs :

```bash
docker compose up -d
```

## Déploiement continu

Les objectifs des déploiements continus de iar sont les suivants (cf. [poldev](https://github.com/abes-esr/abes-politique-developpement/blob/main/01-Gestion%20du%20code%20source.md#utilisation-des-branches)) :

- Un `git push` sur la branche `develop` provoque un déploiement automatique sur le serveur de développement (`dev`).
- Un `git push` (ou merge) sur la branche `main` provoque un déploiement automatique sur le serveur de test (`test`).
- Un `git tag X.X.X` (associé à une release) sur la branche `main` permet un déploiement sur le serveur de production (`prod`).

iar s'appuie sur **WUD (What's Up Docker?)** pour la surveillance et la mise à jour automatique des conteneurs via les labels Docker configurés dans [docker-compose.yml](./docker-compose.yml) :

```yaml
labels:
  - "wud.watch=true"
  - "wud.watch.digest=true"
```

Le fonctionnement de WUD consiste à surveiller régulièrement la publication de nouvelles versions d'images sur Docker Hub pour `iar-batch-dump`, `iar-vectorisation` et `iar-api`. Lorsqu'une nouvelle image est détectée pour le tag ou le digest suivi, WUD la télécharge, arrête l'ancien conteneur et recrée le nouveau conteneur avec les mêmes paramètres d'environnement.

Pour le développeur, il suffit de pousser son code (ex: sur `develop`), d'attendre la complétion de la GitHub Action qui compile et publie l'image sur Docker Hub, puis WUD met à jour le service automatiquement.

## Mise à jour manuelle

Pour récupérer et appliquer manuellement la dernière version des images :

```bash
docker compose pull
docker compose up -d
```

Le `pull` téléchargera la dernière image disponible correspondant aux versions définies dans votre `.env` (ex: `IAR_API_VERSION`, `IAR_VECTORISATION_VERSION`). Sans le pull, Docker réutilise l'image déjà présente localement.

## Architecture

L'application iar est composée de 4 services orchestrés par Docker Compose :

- **iar-batch-dump** : Application Java Spring Boot chargée de l'extraction et du traitement batch des données RAMEAU.
- **iar-vectorisation** : Microservice FastAPI responsable de la vectorisation (embeddings) des données et de leur insertion dans Qdrant.
- **iar-qdrant** : Base de données vectorielle Qdrant hébergeant les collections d'embeddings pour la recherche sémantique.
- **iar-api** : API FastAPI exposant les endpoints de consultation/recherche, s'appuyant sur Qdrant et le service LLM distant.

### Dépôts sources et images Docker

Les images Docker de iar sont issues des dépôts GitHub de l'organisation [abes-esr](https://github.com/abes-esr) :

- [abes-esr/iar-api](https://github.com/abes-esr/iar-api) : Code source de l'API FastAPI
- [abes-esr/iar-vectorisation](https://github.com/abes-esr/iar-vectorisation) : Code source du service de vectorisation
- [abes-esr/iar-batch-dump](https://github.com/abes-esr/iar-batch-dump) : Code source du batch Java Spring Boot
- [abes-esr/iar-docker](https://github.com/abes-esr/iar-docker) : Configuration Docker Compose de déploiement
- Images publiées sur Docker Hub : [abesesr/iar](https://hub.docker.com/r/abesesr/iar)
