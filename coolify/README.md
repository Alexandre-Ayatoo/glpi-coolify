# GLPI sur Coolify — gestion.alexandre-lb.fr

Ce déploiement migre l'instance GLPI 10.0.17 de `web-1` vers l'image officielle
GLPI **10.0.28**, avec MariaDB **10.11**. Le tag Docker 10.0.17 n'est pas publié.
La mise à jour du schéma de la base sera lancée manuellement après restauration.

Le fork contient les sources GLPI de la branche `12.0/bugfixes`. Ce Compose
utilise l'image publiée 10.0.28 : les modifications des sources du fork ne sont
pas intégrées à l'image. Le Compose à la racine est destiné au développement ;
utiliser exclusivement `coolify/docker-compose.yml` pour ce déploiement.

## Configurer Coolify

1. Créer une application depuis `Alexandre-Ayatoo/glpi-coolify`.
2. Sélectionner la branche contenant ces fichiers (actuellement `12.0/bugfixes`).
3. Choisir **Docker Compose**, répertoire de base `/`, fichier
   **`/coolify/docker-compose.yml`**.
4. Renseigner `GLPI_DB_PASSWORD` et `GLPI_DB_ROOT_PASSWORD` avec deux secrets
   distincts. Générer chacun avec `openssl rand -hex 32`.
5. Garder `GLPI_STORAGE_PATH=/data/glpi` et `GLPI_CRONTAB_ENABLED=0`.
6. Préparer les fichiers ci-dessous avant le premier déploiement. Garder les
   déploiements automatiques désactivés pendant la migration.
7. Ne pas attribuer le domaine de production avant les contrôles. Pour les
   essais, utiliser un domaine temporaire dont le DNS pointe vers Coolify,
   affecté au service `glpi`, port interne **80**.

Coolify gère HTTPS. Aucun port hôte ni certificat Nginx/Certbot n'est nécessaire.
MariaDB reste sur le réseau privé de l'application.

## Sauvegarder sur web-1

Exécuter comme root, dans Bash, pendant une fenêtre de maintenance. Arrêter les
tâches cron GLPI et collecteurs sur l'ancien serveur et bloquer temporairement
les accès web/API et inventaires avant le dump final. Le mode maintenance seul
ne garantit pas l'arrêt de toutes les écritures. Ne pas arrêter globalement les
services cron ou Nginx qui servent d'autres applications.

```bash
set -euo pipefail
cd /var/www/html/gestion.alexandre-lb.fr
sudo -u www-data php8.3 bin/console maintenance:enable
install -d -m 700 /root/glpi-migration
umask 077
```

Lire localement `config/config_db.php` pour identifier la base. Pour une base
locale avec accès root par socket, remplacer le nom puis lancer :

```bash
GLPI_SOURCE_DB='REMPLACER_PAR_LE_NOM_EXISTANT'
mariadb-dump --single-transaction --quick --hex-blob \
  --default-character-set=utf8mb4 "$GLPI_SOURCE_DB" \
  > /root/glpi-migration/glpi.sql
test -s /root/glpi-migration/glpi.sql
tar -czpf /root/glpi-migration/glpi-data.tar.gz \
  -C /var/www/html/gestion.alexandre-lb.fr config files marketplace plugins
cd /root/glpi-migration
sha256sum glpi.sql glpi-data.tar.gz > SHA256SUMS
```

Si la base est distante ou que root utilise une autre authentification, adapter
le dump (`--host`, `--user`, `--password` sans valeur pour demander le secret).
Ne pas continuer après une erreur. `--single-transaction` suppose des tables
InnoDB ; toute table non transactionnelle doit rester sans écritures durant le dump.

Transférer le dossier `/root/glpi-migration/` sur le serveur Coolify par SSH/SCP.
Conserver une copie séparée de ces sauvegardes. Ne jamais les ajouter à Git.

## Préparer les fichiers sur le serveur Coolify

Exécuter comme root, uniquement pour une **nouvelle destination vide**. Si
`/data/glpi` existe déjà avec des données, arrêter et vérifier son contenu.
Adapter tous les chemins si `GLPI_STORAGE_PATH` a été changé.

```bash
set -euo pipefail
cd /root/glpi-migration
sha256sum -c SHA256SUMS
test ! -e /data/glpi
install -d /data/glpi/restore /data/glpi/data
tar -xzpf glpi-data.tar.gz -C /data/glpi/restore
mv /data/glpi/restore/config /data/glpi/data/
mv /data/glpi/restore/files /data/glpi/data/
mv /data/glpi/restore/marketplace /data/glpi/
mv /data/glpi/restore/plugins /data/glpi/
rmdir /data/glpi/restore
mkdir -p /data/glpi/data/logs
chown -R 33:33 /data/glpi
chmod 750 /data/glpi/data/config
find /data/glpi/data/config -type f -exec chmod 640 {} +
nano /data/glpi/data/config/config_db.php
```

Dans le fichier restauré, modifier les quatre propriétés suivantes, en conservant
les autres réglages. Le mot de passe doit correspondre à celui saisi dans Coolify :

```php
public $dbhost = 'db';
public $dbuser = 'glpi';
public $dbpassword = 'VALEUR_DE_GLPI_DB_PASSWORD';
public $dbdefault = 'glpi';
```

Garder impérativement `glpicrypt.key` et les autres clés éventuelles. Les variables
d'environnement ne réécrivent pas le `config_db.php` restauré.

Déployer maintenant dans Coolify. L'installation, les mises à jour automatiques
et les tâches GLPI sont désactivées pendant la restauration. L'application peut
présenter une erreur de base vide tant que l'import n'est pas terminé.

## Importer et mettre à jour

Repérer les conteneurs de cette application sur le serveur Coolify :

```bash
docker ps --format 'table {{.ID}}\t{{.Names}}\t{{.Image}}'
GLPI_CID='REMPLACER_PAR_ID_GLPI'
DB_CID='REMPLACER_PAR_ID_MARIADB'
```

Contrôler la destination. Le résultat doit être **0** :

```bash
docker exec "$DB_CID" sh -c \
  'exec mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" -Nse "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema=\"glpi\""'
```

Importer une seule fois dans cette base vide. Arrêter immédiatement si une
commande échoue ; ne pas relancer un import sur une base partiellement remplie.

```bash
docker exec -i "$DB_CID" sh -c \
  'exec mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" glpi' \
  < /root/glpi-migration/glpi.sql

docker exec -u www-data "$GLPI_CID" \
  php bin/console database:update --no-interaction
docker exec -u www-data "$GLPI_CID" php bin/console database:check_schema_integrity
docker exec -u www-data "$GLPI_CID" php bin/console cache:clear
```

Vérifier la sortie de chaque commande. Toute différence de schéma doit être
expliquée avant la bascule. La mise à jour peut désactiver les extensions : ne
réactiver que les versions compatibles avec GLPI 10.0.28.

## Valider et basculer

Sur la nouvelle instance :

```bash
docker exec -u www-data "$GLPI_CID" php bin/console maintenance:disable
```

Sur le domaine temporaire, vérifier connexion, tickets, téléchargement d'anciens
documents, droits, API/inventaire et les 16 extensions : accounts, activity,
actualtime, additionalalerts, addressing, advancedplanning, connections,
escalade, moreticket, mydashboard, news, onetimesecret, pdf, satisfaction,
screenshot et tag. Ne pas démarrer les collecteurs durant les essais.

Après validation, affecter `https://gestion.alexandre-lb.fr` au service `glpi`,
port interne 80, et basculer les DNS A **et AAAA** concernés vers Coolify.
Vérifier aussi l'URL de l'application dans la configuration générale GLPI.
Passer `GLPI_CRONTAB_ENABLED=1` dans Coolify puis redéployer. Les actions
automatiques exécutées par le conteneur nécessitent le mode CLI dans GLPI.
Vérifier les notifications SMTP, les collecteurs et un inventaire réel après
la bascule. Ne pas programmer un deuxième cron pour la même instance.

Garder l'ancien serveur en maintenance et ses tâches arrêtées.

## Retour arrière et sauvegardes

Avant toute écriture sur la nouvelle instance, le retour arrière consiste à
arrêter les conteneurs depuis Coolify, rétablir les DNS puis rétablir l'accès et
les tâches sur l'ancien serveur et y lancer :

```bash
cd /var/www/html/gestion.alexandre-lb.fr
sudo -u www-data php8.3 bin/console maintenance:disable
```

Après des écritures sur la nouvelle instance, un retour DNS seul perdrait ces
changements : il faut d'abord organiser leur récupération. Ne pas importer une
base mise à jour vers 10.0.28 dans l'ancien code 10.0.17.

Sauvegarder régulièrement **le dump MariaDB et `/data/glpi` ensemble**. La base
est dans le volume nommé `db_data`, préfixé par le projet Coolify ; elle n'est
pas dans `/data/glpi`. Conserver les clés hors Git. Un changement des mots de
passe dans Coolify ne modifie pas les comptes d'une base déjà initialisée :
effectuer aussi la rotation SQL et mettre à jour `config_db.php`.

Pour une future mise à jour, sauvegarder, activer la maintenance, changer le tag
GLPI explicitement, redéployer puis lancer `database:update` manuellement.

## Vérifier le Compose sans déployer

Depuis la racine du dépôt, avec deux valeurs fictives uniquement pour le contrôle :

```bash
GLPI_DB_PASSWORD=validation GLPI_DB_ROOT_PASSWORD=validation-root \
  docker compose -f coolify/docker-compose.yml config --quiet
```

Références : [image officielle](https://github.com/glpi-project/docker-images),
[migration GLPI](https://glpi-install.readthedocs.io/en/latest/update.html),
[Docker Compose dans Coolify](https://coolify.io/docs/applications/builds/docker-compose).
