# DevopsProject

Stack Docker Compose pour exposer plusieurs services derrière un reverse proxy Nginx avec certificats Let's Encrypt automatiques.

Cette stack déploie :

- **nginx-proxy** pour router les requêtes HTTP/HTTPS vers les bons conteneurs.
- **acme-companion** pour générer et renouveler automatiquement les certificats Let's Encrypt.
- **Dozzle** pour consulter les logs Docker depuis une interface web.
- **n8n** pour créer des workflows d’automatisation.
- **WordPress** avec une base MySQL dédiée.
- **Zabbix** avec son serveur, son interface web et une base MySQL dédiée.

Le fonctionnement global est le suivant :

1. chaque service expose un `VIRTUAL_HOST` ;
2. `nginx-proxy` détecte automatiquement les conteneurs via le socket Docker ;
3. `acme-companion` génère le certificat SSL pour chaque sous-domaine ;
4. les services deviennent accessibles en HTTPS ;
5. Zabbix peut envoyer une alerte vers **n8n** via un **webhook** ;
6. n8n traite la donnée, la reformate, puis peut publier un article ou une notification dans **WordPress**.

---

## 1. Architecture

### Services publiés

Les services sont publiés via des sous-domaines distincts définis dans le fichier `.env` :

- `https://dozzle.votre-domaine.tld`
- `https://n8n.votre-domaine.tld`
- `https://wordpress.votre-domaine.tld`
- `https://zabbix.votre-domaine.tld`

Dans le `docker-compose.yml`, chaque service exposé au reverse proxy utilise au minimum :

- `VIRTUAL_HOST`
- `LETSENCRYPT_HOST`
- `LETSENCRYPT_EMAIL`
- `VIRTUAL_PORT`

C’est ce mécanisme qui permet à `nginx-proxy` et `acme-companion` de découvrir automatiquement les conteneurs à publier. Voir la définition des services `dozzle`, `n8n`, `wordpress` et `zabbix-web` dans le compose. 

### Réseaux Docker

La stack définit 3 réseaux :

- `public` : réseau partagé par les services publiés derrière le reverse proxy.
- `net-wp-db` : réseau isolé entre WordPress et sa base MySQL.
- `net-zabbix-db` : réseau isolé entre Zabbix et sa base MySQL. 

---

## 2. Prérequis

Avant de lancer la stack, prévoir :

- un serveur Linux avec Docker et le plugin Docker Compose installés ;
- les ports `80` et `443` ouverts sur le firewall et redirigés vers la machine ;
- un nom de domaine avec les enregistrements DNS pointant vers l’IP publique du serveur ;
- une adresse mail valide pour Let's Encrypt ;
- suffisamment d’espace disque pour les volumes persistants ;
- un accès root ou sudo pour créer l’arborescence et gérer les permissions.

### Vérifications utiles

```bash
docker --version
docker compose version
sudo ss -tulpn | grep -E ':80|:443'
```

---

## 3. Arborescence conseillée

```text
.
├── docker-compose.yml
├── .env
└── data/
    ├── proxy/
    │   ├── acme/
    │   ├── certs/
    │   ├── html/
    │   └── vhost/
    │       ├── dozzle.votre-domaine.tld
    │       ├── n8n.votre-domaine.tld
    │       └── zabbix.votre-domaine.tld
    ├── dozzle/
    │   └── users.yml
    ├── n8n/
    ├── wp/
    └── db/
        ├── wp/
        └── zabbix/
```

---

## 4. Exemple complet du fichier `.env`

Le README d’origine mentionne seulement les variables minimales. La stack actuelle utilise en réalité les variables suivantes pour le proxy, Dozzle, WordPress et Zabbix. 

Exemple :

```env
# Email utilisé par Let's Encrypt
MY_EMAIL=admin@.exemple.com

# Sous-domaines publiés
DOZZLE_DOMAIN=dozzle.exemple.com
N8N_DOMAIN=n8n.exemple.com
WORDPRESS_DOMAIN=wordpress.exemple.com
ZABBIX_DOMAIN=zabbix.exemple.com

# --- WORDPRESS ---
WP_DB_HOST=db-wp
WP_DB_PORT=3306
WP_DB_NAME=wordpress
WP_DB_USER=wpuser
WP_DB_PASSWORD=change_me_wp_password
WP_ROOT_PASSWORD=change_me_wp_root_password

# --- ZABBIX ---
ZBX_SERVER_HOST=zabbix-server
ZBX_DB_HOST=db-zabbix
ZBX_DB_NAME=zabbix
ZBX_DB_USER=zabbix
ZBX_DB_PASSWORD=change_me_zbx_password
ZBX_ROOT_PASSWORD=change_me_zbx_root_password

# --- RECOMMANDÉ POUR N8N (production) ---
# Ces variables ne sont pas toutes présentes dans le compose actuel,
# mais elles sont fortement recommandées si tu veux fiabiliser n8n.
N8N_HOST=n8n.exemple.com
N8N_PROTOCOL=https
N8N_PORT=5678
WEBHOOK_URL=https://n8n.exemple.com/
GENERIC_TIMEZONE=Europe/Paris
TZ=Europe/Paris
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=change_me_n8n_password
```

### Explication rapide

- `MY_EMAIL` : adresse utilisée pour l’émission des certificats Let's Encrypt.
- `DOZZLE_DOMAIN`, `N8N_DOMAIN`, `WORDPRESS_DOMAIN`, `ZABBIX_DOMAIN` : sous-domaines exposés par le reverse proxy.
- `WP_DB_*` : accès à la base MySQL de WordPress.
- `ZBX_*` : accès à la base MySQL et lien entre le frontend Zabbix et le serveur Zabbix.
- `WEBHOOK_URL` : important pour n8n quand il génère des URLs de webhook publiques.
- `GENERIC_TIMEZONE` / `TZ` : utile pour l’horodatage cohérent côté n8n et dans les logs.

> Ne jamais versionner ce fichier dans Git avec des mots de passe réels.

---

## 5. Configuration des vhosts Nginx

Le compose monte le dossier `./data/proxy/vhost` dans `/etc/nginx/vhost.d` du reverse proxy. Chaque fichier de ce dossier permet d’ajouter des règles Nginx spécifiques à un sous-domaine.

### Exemple fourni pour Dozzle

Le fichier `dozzle.exemple.com` contient une restriction d’accès par IP, puis une redirection en cas de refus :

```nginx
allow 8.8.8;
allow 1.1.1;
allow 192.168.X.X/XX;
deny all;

error_page 403 =302 https://www.youtube.com/watch?v=xvFZjo5PgG0;
```

Même logique pour `n8n.exemple.com` et `zabbix.exemple.com`. Ces fichiers existent déjà dans les fichiers fournis. 

### Installation / déploiement des vhosts

Créer les fichiers :

```bash
nano data/proxy/vhost/dozzle.exemple.com
nano data/proxy/vhost/n8n.exemple.com
nano data/proxy/vhost/zabbix.exemple.com
```

Exemple type :

```nginx
allow 8.8.8;
allow 1.1.1;
allow 192.168.X.X/XX;
deny all;

error_page 403 =302 https://www.youtube.com/watch?v=xvFZjo5PgG0;
```

Puis redéployer :

```bash
docker compose up -d
```

### À savoir

- ces règles s’appliquent au vhost ciblé uniquement ;
- `allow/deny` sont pratiques pour réserver Dozzle, n8n ou Zabbix à certaines IP publiques ou privées ;
- WordPress est souvent laissé ouvert publiquement, donc aucun fichier `vhost` spécifique n’est obligatoire pour lui ;
- si tu modifies un fichier `vhost`, redémarrer le proxy peut aider à recharger la configuration :

```bash
docker compose restart proxy ssl-helper
```

---

## 6. Fichier d’authentification Dozzle

Le service Dozzle est configuré avec :

- `DOZZLE_AUTH_PROVIDER=simple`
- un volume `./data/dozzle/users.yml:/data/users.yml:ro` 

Exemple :

```yaml
users:
  admin:
    email: admin@exemple.com
    name: admin
    password: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    filter:
    roles: all
```

### Générer un mot de passe hashé

Exemple avec `htpasswd` :

```bash
docker run --rm httpd:2.4-alpine htpasswd -nbB admin 'mot_de_passe_tres_fort'
```

Récupère uniquement la partie hashée et colle-la dans `data/dozzle/users.yml`.

---

## 7. Installation de la stack complète

### 7.1. Fichier `docker-compose.yml`

Le compose fourni contient déjà :

- le reverse proxy `nginxproxy/nginx-proxy` ;
- `nginxproxy/acme-companion` ;
- `amir20/dozzle:latest` ;
- `n8nio/n8n:latest` ;
- `wordpress:latest` ;
- deux conteneurs `mysql:8.0` pour WordPress et Zabbix ;
- `zabbix/zabbix-web-nginx-mysql:ubuntu-6.4-latest` ;
- `zabbix/zabbix-server-mysql:ubuntu-6.4-latest`. 

### 7.2. Lancement initial

```bash
docker compose pull
docker compose up -d
```

### 7.3. Vérification

```bash
docker compose ps
docker compose logs -f proxy
docker compose logs -f ssl-helper
```

### 7.4. Vérifier les certificats Let's Encrypt

Au premier démarrage :

- les DNS doivent déjà pointer vers le serveur ;
- les ports 80 et 443 doivent être accessibles ;
- les sous-domaines doivent être correctement définis dans `.env`.

Les certificats et artefacts ACME sont stockés dans :

- `data/proxy/certs/`
- `data/proxy/acme/`
- `data/proxy/html/`
- `data/proxy/vhost/` 

---

## 8. Installation détaillée par composant

## 8.1. Reverse proxy + SSL

### Rôle

- publication des services en HTTP/HTTPS ;
- terminaison TLS ;
- renouvellement auto des certificats.

### Services concernés

- `proxy`
- `ssl-helper` 

### Points importants

- `proxy` expose `80:80` et `443:443` ;
- `ssl-helper` dépend de `proxy` ;
- le socket Docker est monté dans les 2 conteneurs ;
- les certificats sont persistés dans `data/proxy/certs`.

---

## 8.2. Installation de Dozzle

### Rôle

Dozzle permet de consulter rapidement les logs des conteneurs Docker depuis une interface web et de les up / down.

### Variables utilisées

```yaml
environment:
  - VIRTUAL_HOST=${DOZZLE_DOMAIN}
  - LETSENCRYPT_HOST=${DOZZLE_DOMAIN}
  - LETSENCRYPT_EMAIL=${MY_EMAIL}
  - VIRTUAL_PORT=8080
  - DOZZLE_ENABLE_ACTIONS=true
  - DOZZLE_AUTH_PROVIDER=simple
```

### Volume utilisé

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro
  - ./data/dozzle/users.yml:/data/users.yml:ro
```

C’est bien la configuration présente dans le compose. 

### Étapes

1. créer `data/dozzle/users.yml` ;
2. y définir au moins un utilisateur ;
3. créer le vhost `data/proxy/vhost/dozzle.exemple.com` si tu veux restreindre l’accès ;
4. lancer la stack ;
5. ouvrir `https://dozzle.exemple.com`.

---

## 8.3. Installation de n8n

### Rôle

n8n sert à recevoir des webhooks, transformer les données, appeler des APIs et chaîner des actions automatiques.

### Configuration actuelle du compose

```yaml
n8n:
  image: n8nio/n8n:latest
  container_name: n8n
  environment:
    - VIRTUAL_HOST=${N8N_DOMAIN}
    - LETSENCRYPT_HOST=${N8N_DOMAIN}
    - LETSENCRYPT_EMAIL=${MY_EMAIL}
    - VIRTUAL_PORT=5678
  volumes:
    - ./data/n8n:/home/node/.n8n
```

Cette définition existe déjà dans le compose. 

### Recommandations de production

Même si elles ne sont pas encore dans le compose fourni, il est conseillé d’ajouter ensuite :

```yaml
    - N8N_HOST=${N8N_HOST}
    - N8N_PROTOCOL=${N8N_PROTOCOL}
    - N8N_PORT=${N8N_PORT}
    - WEBHOOK_URL=${WEBHOOK_URL}
    - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
    - TZ=${TZ}
    - N8N_BASIC_AUTH_ACTIVE=${N8N_BASIC_AUTH_ACTIVE}
    - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
    - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
```

### Étapes

1. créer `data/n8n/` ;
2. démarrer la stack ;
3. ouvrir `https://n8n.exemple.com` ;
4. terminer l’installation initiale n8n ;
5. configurer les credentials nécessaires, notamment pour WordPress.

---

## 8.4. Installation de WordPress

### Rôle

WordPress reçoit ici les publications créées manuellement ou automatiquement via n8n.

### Services concernés

- `wordpress`
- `db-wp` 

### Configuration applicative

```yaml
wordpress:
  image: wordpress:latest
  environment:
    - VIRTUAL_HOST=${WORDPRESS_DOMAIN}
    - LETSENCRYPT_HOST=${WORDPRESS_DOMAIN}
    - LETSENCRYPT_EMAIL=${MY_EMAIL}
    - WORDPRESS_DB_HOST=${WP_DB_HOST}:${WP_DB_PORT}
    - WORDPRESS_DB_USER=${WP_DB_USER}
    - WORDPRESS_DB_PASSWORD=${WP_DB_PASSWORD}
    - WORDPRESS_DB_NAME=${WP_DB_NAME}
```

### Base MySQL

```yaml
db-wp:
  image: mysql:8.0
  environment:
    - MYSQL_DATABASE=${WP_DB_NAME}
    - MYSQL_USER=${WP_DB_USER}
    - MYSQL_PASSWORD=${WP_DB_PASSWORD}
    - MYSQL_ROOT_PASSWORD=${WP_ROOT_PASSWORD}
```

Le conteneur WordPress attend que `db-wp` soit sain avant de démarrer. 

### Étapes

1. démarrer la stack ;
2. ouvrir `https://wordpress.exemple.com` ;
3. terminer l’assistant d’installation WordPress ;
4. créer un compte administrateur ;
5. installer si besoin l’API REST ou vérifier qu’elle est accessible ;
6. pour n8n, créer idéalement un utilisateur dédié à la publication automatique.

---

## 8.5. Installation de Zabbix

### Rôle

Zabbix collecte les métriques, surveille les hôtes et génère les alertes.

### Services concernés

- `zabbix-web`
- `zabbix-server`
- `db-zabbix` 

### Frontend web

```yaml
zabbix-web:
  image: zabbix/zabbix-web-nginx-mysql:ubuntu-6.4-latest
  environment:
    - VIRTUAL_HOST=${ZABBIX_DOMAIN}
    - LETSENCRYPT_HOST=${ZABBIX_DOMAIN}
    - LETSENCRYPT_EMAIL=${MY_EMAIL}
    - VIRTUAL_PORT=8080
    - ZBX_SERVER_HOST=${ZBX_SERVER_HOST}
    - DB_SERVER_HOST=${ZBX_DB_HOST}
    - MYSQL_DATABASE=${ZBX_DB_NAME}
    - MYSQL_USER=${ZBX_DB_USER}
    - MYSQL_PASSWORD=${ZBX_DB_PASSWORD}
    - MYSQL_ROOT_PASSWORD=${ZBX_ROOT_PASSWORD}
```

### Serveur Zabbix

```yaml
zabbix-server:
  image: zabbix/zabbix-server-mysql:ubuntu-6.4-latest
  environment:
    - DB_SERVER_HOST=${ZBX_DB_HOST}
    - MYSQL_DATABASE=${ZBX_DB_NAME}
    - MYSQL_USER=${ZBX_DB_USER}
    - MYSQL_PASSWORD=${ZBX_DB_PASSWORD}
    - MYSQL_ROOT_PASSWORD=${ZBX_ROOT_PASSWORD}
```

### Base MySQL

```yaml
db-zabbix:
  image: mysql:8.0
```

La base dispose d’un healthcheck, le serveur Zabbix attend la base, puis le frontend attend le serveur. 

### Étapes

1. démarrer la stack ;
2. ouvrir `https://zabbix.exemple.com` ;
3. terminer l’installation du frontend ;
4. créer les hôtes, templates, triggers et actions ;
5. restreindre l’accès au frontend via le vhost si nécessaire.

---

## 9. Chaîne d’automatisation : Zabbix → n8n → WordPress

L’objectif est :

- Zabbix détecte une alerte ;
- Zabbix envoie les données à un **webhook n8n** ;
- n8n reçoit les données, les transforme ;
- n8n publie ensuite un article WordPress, une actu, un brouillon ou une notification.

---

## 9.1. Création du webhook dans n8n

Dans n8n :

1. créer un nouveau workflow ;
2. ajouter un nœud **Webhook** ;
3. choisir par exemple :
   - méthode `POST`
   - path `zabbix-alert`
4. sauvegarder le workflow ;
5. récupérer l’URL de test puis l’URL de production.

Exemple d’URL finale attendue :

```text
https://n8n.exemple.com/webhook/zabbix-alert
```

> Pour que l’URL soit correcte derrière le reverse proxy, `WEBHOOK_URL` est fortement recommandé côté n8n.

### Exemple de payload JSON attendu

Tu peux faire envoyer à Zabbix quelque chose de ce type :

```json
{
  "host": "srv-web-01",
  "trigger": "CPU trop élevé",
  "severity": "High",
  "status": "PROBLEM",
  "event_id": "123456",
  "item": "CPU utilization",
  "value": "95%",
  "date": "2026-03-19 10:15:00",
  "message": "Le CPU du serveur srv-web-01 dépasse le seuil critique"
}
```

---

## 9.2. Création du média type / webhook dans Zabbix

Dans Zabbix, la façon la plus propre consiste à :

1. créer un **Media type** de type **Webhook** ou utiliser une action appelant une URL ;
2. définir l’URL du webhook n8n ;
3. envoyer le contenu de l’alerte en JSON ;
4. créer une **Action** qui utilise ce média type sur les triggers voulus.

### Exemple de destination

```text
https://n8n.exemple.com/webhook/zabbix-alert
```

### Exemple de corps JSON côté Zabbix

```json
{
  "host": "{HOST.NAME}",
  "trigger": "{TRIGGER.NAME}",
  "severity": "{TRIGGER.SEVERITY}",
  "status": "{EVENT.STATUS}",
  "event_id": "{EVENT.ID}",
  "item": "{ITEM.NAME}",
  "value": "{ITEM.LASTVALUE}",
  "date": "{EVENT.DATE} {EVENT.TIME}",
  "message": "{TRIGGER.DESCRIPTION}"
}
```

Selon la version et la méthode choisie dans Zabbix, certains macros peuvent varier. Le principe reste le même : envoyer un JSON propre vers le webhook n8n.

---

## 9.3. Workflow n8n pour publier sur WordPress

### Logique du workflow

1. **Webhook** : reçoit l’alerte Zabbix.
2. **Set** ou **Code** : reformate le contenu.
3. **IF** : filtre les alertes selon la sévérité ou le statut.
4. **WordPress** : crée un article, une page ou un brouillon.
5. éventuellement **Telegram / Email / Slack / Discord** en plus.

### Exemple simple de mapping dans n8n

Titre WordPress :

```text
[Alerte Zabbix] {{$json.trigger}} sur {{$json.host}}
```

Contenu WordPress :

```html
<h2>Alerte de supervision</h2>
<ul>
  <li><strong>Hôte :</strong> {{$json.host}}</li>
  <li><strong>Trigger :</strong> {{$json.trigger}}</li>
  <li><strong>Sévérité :</strong> {{$json.severity}}</li>
  <li><strong>Statut :</strong> {{$json.status}}</li>
  <li><strong>Valeur :</strong> {{$json.value}}</li>
  <li><strong>Date :</strong> {{$json.date}}</li>
</ul>
<p>{{$json.message}}</p>
```

Statut conseillé :

- `draft` si tu veux relire avant publication ;
- `publish` si tu veux publier automatiquement.

---

## 9.4. Connexion n8n à WordPress

Dans n8n, créer un credential WordPress avec :

- l’URL du site, par exemple `https://wordpress.exemple.com` ;
- un utilisateur WordPress dédié ;
- un mot de passe applicatif WordPress si possible.

### Bonne pratique

Créer un utilisateur spécifique du type :

- `n8n-bot`

Avec des droits limités à la publication si ton workflow n’a pas besoin d’administration complète.

---

## 9.5. Exemple de workflow minimal

```text
Webhook (POST /zabbix-alert)
  -> IF severity == High OR Disaster
    -> WordPress Create Post (status=draft)
```

### Variante plus complète

```text
Webhook
  -> Set (normalisation des champs)
  -> IF status == PROBLEM
      -> WordPress Create Post
      -> Dozzle / logs / autre notif si besoin
  -> IF status == RESOLVED
      -> WordPress Create Post ou Update Post
```

---

## 10. Commandes d’exploitation courante

### Démarrer

```bash
docker compose up -d
```

### Arrêter

```bash
docker compose down
```

### Voir l’état des conteneurs

```bash
docker compose ps
```

### Suivre tous les logs

```bash
docker compose logs -f
```

### Logs ciblés

```bash
docker logs nginx-proxy
docker logs nginx-proxy-acme
docker logs dozzle
docker logs n8n
docker logs wordpress
docker logs zabbix-web
docker logs zabbix-server
```

### Redémarrer un service

```bash
docker compose restart wordpress
docker compose restart n8n
```

### Mettre à jour les images

```bash
docker compose pull
docker compose up -d
```

---

## 11. Sauvegardes recommandées

Sauvegarder régulièrement :

- `.env`
- `docker-compose.yml`
- `data/n8n/`
- `data/wp/`
- `data/db/wp/`
- `data/db/zabbix/`
- `data/dozzle/users.yml`
- `data/proxy/` si tu veux conserver certificats et configuration.

---

## 12. Points d’attention

- le fichier `.env` contient des secrets ;
- les volumes Docker contiennent les données persistantes ;
- les restrictions de vhost peuvent bloquer ton accès à n8n, Dozzle ou Zabbix si ton IP change ;
- sans `WEBHOOK_URL`, certains webhooks n8n peuvent être générés avec une URL incorrecte derrière le reverse proxy ;
- WordPress doit être complètement initialisé avant de pouvoir recevoir des publications via n8n ;
- il est préférable de publier en `draft` au début pour valider le workflow Zabbix → n8n → WordPress.

---

## 13. Dépannage rapide

### Un service n’est pas accessible

1. vérifier `docker compose ps`
2. vérifier les logs du proxy et du service concerné ;
3. vérifier le DNS ;
4. vérifier les ports 80/443 ;
5. vérifier les variables du `.env`.

### Le certificat n’est pas généré

- le sous-domaine ne pointe pas vers le serveur ;
- le port 80 n’est pas joignable ;
- `LETSENCRYPT_HOST` ou `MY_EMAIL` est mal renseigné.

### n8n reçoit mal le webhook

- URL publique incorrecte ;
- `WEBHOOK_URL` absent ;
- webhook non activé ;
- vhost n8n bloqué par IP.

### WordPress ne publie pas depuis n8n

- mauvais identifiants ;
- API REST inaccessible ;
- utilisateur WordPress insuffisamment autorisé ;
- URL du site incorrecte dans le credential.

### Zabbix n’envoie rien

- action non déclenchée ;
- mauvais média type ;
- JSON invalide ;
- accès HTTP sortant bloqué.

---

## 14. Fichiers fournis dans ce projet

- `docker-compose.yml` : définition complète de la stack. 
- `README.md` : documentation du projet. 
- `users.yml` : utilisateurs Dozzle avec mot de passe hashé. 
- fichiers vhost pour `dozzle.exemple.com`, `n8n.exemple.com` et `zabbix.exemple.com` : filtrage IP côté Nginx. 

---

## 15. Résumé

Cette stack permet d’avoir une plateforme auto-hébergée cohérente avec :

- reverse proxy SSL automatique ;
- centralisation des logs avec Dozzle ;
- automatisation avec n8n ;
- publication web avec WordPress ;
- supervision avec Zabbix ;
- possibilité de chaîner **Zabbix → Webhook n8n → Workflow n8n → Publication WordPress**.

Pour une mise en production propre, les améliorations les plus utiles à ajouter ensuite sont :

- variables n8n complètes (`WEBHOOK_URL`, auth, timezone) ;
- sauvegardes automatiques ;
- limitation d’accès par IP ou auth forte sur n8n, Dozzle et Zabbix ;
- utilisateur WordPress dédié à l’automatisation ;
- publication WordPress d’abord en brouillon avant passage en automatique.
