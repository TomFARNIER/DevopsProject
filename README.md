# DevopsProject

Stack Docker Compose pour exposer plusieurs services derrière un reverse proxy Nginx avec certificats Let's Encrypt automatiques.

Le projet déploie les composants suivants:

- Nginx Proxy pour le routage HTTP/HTTPS.
- ACME Companion pour la génération et le renouvellement des certificats.
- Dozzle pour la consultation des logs Docker.
- n8n pour l'automatisation de workflows.
- WordPress avec une base MySQL dédiée.
- Zabbix avec une base MySQL dédiée.

## Architecture

Les services publiés utilisent le domaine défini dans `.env` via `MY_DOMAIN`:

- `https://dozzle.MY_DOMAIN`
- `https://n8n.MY_DOMAIN`
- `https://wordpress.MY_DOMAIN`
- `https://zabbix.MY_DOMAIN`

Le reverse proxy détecte automatiquement les conteneurs grâce aux variables `VIRTUAL_HOST`, `LETSENCRYPT_HOST` et `VIRTUAL_PORT` définies dans `docker-compose.yml`.

## Prerequis

- Docker et Docker Compose plugin installés.
- Les ports `80` et `443` ouverts sur l'hôte.
- Les entrées DNS des sous-domaines pointent vers l'adresse IP de la machine.
- Une adresse mail valide pour Let's Encrypt.

## Variables d'environnement

Le fichier `.env` doit définir au minimum:

```env
MY_EMAIL=
MY_DOMAIN=

WP_DB_NAME=
WP_DB_USER=
WP_DB_PASSWORD=
WP_ROOT_PASSWORD=

ZBX_DB_NAME=
ZBX_DB_USER=
ZBX_DB_PASSWORD=
ZBX_ROOT_PASSWORD=
```

Description rapide:

- `MY_EMAIL`: adresse utilisée par Let's Encrypt.
- `MY_DOMAIN`: domaine racine utilisé pour publier les services.
- Variables `WP_*`: configuration de la base WordPress.
- Variables `ZBX_*`: configuration de la base Zabbix.

## Arborescence utile

- `docker-compose.yml`: définition complète de la stack.
- `data/proxy/`: certificats, fichiers ACME, vhosts et contenu web du proxy.
- `data/dozzle/users.yml`: utilisateurs Dozzle.
- `data/n8n/`: données persistantes n8n.
- `data/db/wp/`: données MySQL de WordPress.
- `data/db/zabbix/`: données MySQL de Zabbix.

## Demarrage

Lancer la stack:

```bash
docker compose up -d
```

Verifier l'etat des conteneurs:

```bash
docker compose ps
```

Afficher les logs d'un service:

```bash
docker logs nginx-proxy
docker logs nginx-proxy-acme
docker logs wordpress
docker logs zabbix-web
```

Arreter la stack:

```bash
docker compose down
```

## Services exposes

### Reverse proxy et certificats

- `proxy`: publie les ports `80` et `443`.
- `ssl-helper`: genere et renouvelle les certificats Let's Encrypt.

Les certificats et fichiers associes sont stockes dans `data/proxy/certs/` et `data/proxy/acme/`.

### Dozzle

- URL: `https://dozzle.MY_DOMAIN`
- Port interne: `8080`
- Authentification simple configuree via `data/dozzle/users.yml`

### n8n

- URL: `https://n8n.MY_DOMAIN`
- Port interne: `5678`
- Donnees persistantes: `data/n8n/`

### WordPress

- URL: `https://wordpress.MY_DOMAIN`
- Conteneur applicatif: `wordpress`
- Base de donnees: `db-wp`
- Donnees persistantes:
	- application: `data/wp/`
	- base MySQL: `data/db/wp/`

Le service WordPress attend que `db-wp` soit sain avant de demarrer.

### Zabbix

- URL: `https://zabbix.MY_DOMAIN`
- Frontend web: `zabbix-web`
- Serveur: `zabbix-server`
- Base de donnees: `db-zabbix`
- Donnees persistantes MySQL: `data/db/zabbix/`

Le frontend attend que `zabbix-server` soit sain, et le serveur attend que `db-zabbix` soit sain.

## Reseaux Docker

- `public`: services exposes via le reverse proxy.
- `net-wp-db`: isolation entre WordPress et sa base.
- `net-zabbix-db`: isolation entre Zabbix et sa base.

## Exploitation courante

Redemarrer un service:

```bash
docker compose restart wordpress
```

Mettre a jour les images puis redeployer:

```bash
docker compose pull
docker compose up -d
```

Suivre les logs en continu:

```bash
docker compose logs -f
```

## Points d'attention

- Le fichier `.env` contient des secrets: il doit rester prive.
- Les dossiers sous `data/` contiennent des donnees persistantes et ne doivent pas etre supprimes sans sauvegarde.
- L'obtention des certificats Let's Encrypt echouera si les sous-domaines ne resolvent pas vers l'hote ou si les ports `80/443` ne sont pas accessibles.
- Les certificats deja presents dans `data/proxy/certs/` peuvent provenir d'un deploiement existant.

## Depannage rapide

Si un service n'est pas accessible:

1. Verifier que le conteneur est demarre avec `docker compose ps`.
2. Verifier les logs du proxy et du service concerne.
3. Verifier la resolution DNS du sous-domaine.
4. Verifier que les ports `80` et `443` sont joignables depuis Internet.
5. Verifier que les variables du fichier `.env` sont coherentes.