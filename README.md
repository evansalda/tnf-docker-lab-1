# Lab 1 - Création d'une application multi-conteneurs avec la CLI Docker

Créez un répertoire dédié à ce lab qui sera cité tout au long de l'énoncé comme **l'espace du lab**.

## 1. Présentation de l'application

### a. L'application

Vous allez créer une application web qui contient 2 pages distinctes :

- **Vote** - permet de voter pour **CATS** et **DOGS**
- **Resultat** - affiche le résultat en pourcentage des votes

Aperçu de la page de vote :

![](./img/vote-hp.png)

Aperçu de la page de résultat :

![](./img/result-hp.png)

L'application n'accepte qu'un vote par navigateur et chaque vote est changeable.

### b. Architecture

![](./img/voting-app-archi.png)

L'architecture est composée des éléments suivants :

- **Réseau front-end** - destiné aux conteneurs accessibles depuis l'extérieur
- **Réseau back-end** - destiné aux conteneurs non accessibles depuis l'extérieur
- **vote** - conteneur qui publie la page de vote sur le port 5001 du docker host
- **result** - conteneur qui publie la page de résultat sur le port 5002 du docker host
- **redis** - conteneur qui reçoit les votes du conteneur vote et les transmet au worker
- **worker** - conteneur tampon qui reçoit les votes du conteneur redis et les inscrit dans la base de données
- **db** - base de données qui stocke les votes

## 2. Build des images

### a. Vote

Le répertoire [vote/](./src/vote/) contient l'application qui fournit la page de vote.

### Développement du Dockerfile

- Copiez ce répertoire dans l'espace du lab

- Placez-vous dans le répertoire vote et crééz un fichier nommé **Dockerfile**

Ajoutez dans ce Dockerfile les instructions requises pour effectuer les actions suivantes (ordre est à respecter) :

- L'image de base doit être la version **3.11-slim** de l'image **python**

- Ajoutez l'instruction suivante pour configurer le [répertoire courant](https://docs.docker.com/reference/dockerfile/#workdir) du Dockerfile : `WORKDIR /usr/local/app`

- Copie du répertoire **static** dans le répertoire **/usr/local/app** de l'image : `COPY static /usr/local/app/static`

- Copie du répertoire **templates** dans le répertoire **/usr/local/app** de l'image

- Copie des fichiers **app.py** et **requirements.txt** dans le répertoire **/usr/local/app** de l'image : `COPY app.py requirements.txt /usr/local/app`

- Installation des paquets listés dans le fichier requirements.txt via la commande `pip install --no-cache-dir -r requirements.txt`

- Enfin l'instruction suivante doit clotûrer le fichier pour que l'application soit lancée au démarrage : `CMD ["gunicorn", "app:app", "-b", "0.0.0.0:80", "--log-file", "-", "--access-logfile", "-", "--workers", "4", "--keep-alive", "0"]`

### Build de l'image

A l'aide du Dockerfile que vous venez de créer, utilisez la commande [docker build](https://docs.docker.com/get-started/docker-concepts/building-images/build-tag-and-publish-an-image/#building-images) pour builder la version **1.0** de l'image **vote**.

N'oubliez pas de préciser le [contexte de build](https://docs.docker.com/build/concepts/context/) dans votre commande.

### b. Result

Le répertoire [result/](./src/result/) contient l'application qui fournit la page de résultat.

### Développement du Dockerfile

- Copiez ce répertoire dans l'espace du lab

- Placez-vous dans le répertoire result et crééz un fichier nommé **Dockerfile**

Ajoutez dans ce Dockerfile les instructions requises pour effectuer les actions suivantes (ordre est à respecter) :

- L'image de base doit être la version **18-slim** de l'image **node**

- Ajoutez l'instruction suivante pour configurer le [répertoire courant](https://docs.docker.com/reference/dockerfile/#workdir) du Dockerfile : `WORKDIR /usr/local/app`

- Copie du répertoire **views** dans le répertoire **/usr/local/app** de l'image

- Copie des fichiers **package-lock.json**, **package.json** et **server.js** dans le répertoire **/usr/local/app** de l'image

- Exécution des commandes :
    - `npm install -g nodemon`
    - `npm ci && npm cache clean --force && mv /usr/local/app/node_modules /node_modules`

- La variable d'environnement **PORT** doit avoir pour valeur **80**

- Enfin les instructions suivantes doivent clotûrer le fichier pour que l'application soit lancée au démarrage :

````
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "server.js"]
````

### Build de l'image

A l'aide du Dockerfile que vous venez de créer, utilisez la commande [docker build](https://docs.docker.com/get-started/docker-concepts/building-images/build-tag-and-publish-an-image/#building-images) pour builder la version **1.0** de l'image **result**.

N'oubliez pas de préciser le [contexte de build](https://docs.docker.com/build/concepts/context/) dans votre commande.

### c. Worker

Le répertoire [worker/](./src/worker/) contient l'application qui permet de lire les votes dans le cache redis et les écrire dans la base de données PostgreSQL. Ce répertoire contient également le **[Dockerfile](./src/worker/Dockerfile)** permettant de build l'image.

Copiez ce répertoire dans l'espace du lab.

### Build de l'image

- Déplacez dans le répertoire du lab le fichier **Dockerfile** contenu dans votre répertoire **worker/**
- Placez-vous dans le répertoire worker/
- A l'aide du Dockerfile que vous venez de déplacer, utilisez la commande [docker build](https://docs.docker.com/get-started/docker-concepts/building-images/build-tag-and-publish-an-image/#building-images) pour builder la version **1.0** de l'image **worker**.

N'oubliez pas de préciser le [contexte de build](https://docs.docker.com/build/concepts/context/) dans votre commande.

### d. Redis et Db

Nous utiliserons directement les images [redis](https://hub.docker.com/_/redis) et [postgres](https://hub.docker.com/_/postgres) du Docker Hub pour créer les conteneurs correspondant.

## 3. Création des réseaux

A l'aide de la commande [docker network create](https://docs.docker.com/engine/network/#user-defined-networks) créez les 2 réseaux suivants :

- Front-end
    - **Nom** - front-end
    - **Adresse de sous-réseau** - 172.20.0.0/16
    - **Driver** - bridge

- Back-end
    - **Nom** - back-end
    - **Adresse de sous-réseau** - 172.30.0.0/16
    - **Driver** - bridge

## 4. Création du volume

Utilisez la commande [docker volume create](https://docs.docker.com/reference/cli/docker/volume/create/) pour créer un volume nommé **db-data**. Ce volume sera utilisé par la base de données pour y persister ses données.

## 5. Création des conteneurs

Afin d'observer leurs logs en direct, chaque conteneur sera lancé dans une fenêtre Powershell dédiée et en non-détaché.

Créez les conteneurs suivants en respectant l'ordre de leur listing :

- Base de données
    - **Nom** - db
    - **Image** - postgres
    - **Version de l'imgage** - 15-alpine
    - **Valeur de la variable d'environnement POSTGRES_USER** - postgres
    - **Valeur de la variable d'environnement POSTGRES_PASSWORD** - postgres
    - **Réseaux** - back-end
    - **Volume** - Utilisation du volume **db-data** pour persister les données contenues dans le répertoire **/var/lib/postgresql/data**

![](./img/db-log.png)

- Redis
    - **Nom** - redis
    - **Image** - redis
    - **Version de l'imgage** - alpine
    - **Réseaux** - back-end

![](./img/redis-log.png)

- Worker
    - **Nom** - worker-app
    - **Image** - worker
    - **Version de l'imgage** - 1.0
    - **Réseaux** - back-end

![](./img/worker-log.png)

- Vote
    - **Nom** - vote-app
    - **Image** - vote
    - **Version de l'imgage** - 1.0
    - **Réseaux** - front-end et back-end
    - **Port-binding** - Port **80 du conteneur** avec le port **5001 du Docker Host**

![](./img/vote-log.png)

- Result
    - **Nom** - result-app
    - **Image** - result
    - **Version de l'imgage** - 1.0
    - **Réseaux** - front-end et back-end
    - **Port-binding** - Port **80 du conteneur** avec le port **5002 du Docker Host**

![](./img/result-log.png)

## 6. Test de l'application

- Accédez à la page de vote via l'URL **http://localhost:5001**
- Votez pour **CATS** ou **DOGS**
- Accédez à la page de résultat via l'URL **http://localhost:5002**
- Vérifiez que votre vote a bien été pris en compte dans les résultats

Analysez les logs des conteneurs **vote-app** et **worker-app** et confirmez qu'ils correspondent bien aux actions que vous avez réalisé :

![](./img/vote-access-log.png)

![](./img/worker-process-log.png)

Félicitations, vous venez de déployer votre première application reposant sur une architecture dite de micro-services !