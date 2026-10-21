# GLPI sur Coolify — gestion.alexandre-lb.fr

Ce Compose utilise l'image officielle **GLPI 10.0.28** avec la **base distante
existante** : `bdd.alexandre-lb.fr:3306`, base `glpidb`, utilisateur `glpiuser`.
Il ne crée aucun serveur MariaDB et ne demande aucun mot de passe root SQL.
Le `mysql --version` fourni initialement décrit le client de web-1, pas la
version réelle du serveur distant.

Le fork contient les sources de la branche `12.0/bugfixes`. Ce déploiement
utilise l'image publiée 10.0.28 : les modifications des sources du fork ne sont
pas intégrées à l'image. Le Compose à la racine sert au développement.

## Réglages Coolify

- Dépôt : `Alexandre-Ayatoo/glpi-coolify`, branche `12.0/bugfixes`.
- Build : **Docker Compose**, répertoire `/`, fichier `/coolify/docker-compose.yml`.
- Service HTTP : `glpi`, port interne **80**. Coolify gère HTTPS.
- Variables : reprendre `coolify/.env.example` et saisir le mot de passe SQL
  existant dans `GLPI_DB_PASSWORD`, uniquement dans Coolify.
- Rendre les variables disponibles pendant **Build time et Runtime** : Coolify
  lit aussi le Compose avec son fichier `build-time.env`.
- Garder `GLPI_CRONTAB_ENABLED=0` jusqu'à la fin de la migration et les
  déploiements automatiques désactivés pendant l'opération.
- Supprimer l'ancienne variable `GLPI_DB_ROOT_PASSWORD` si elle avait été ajoutée.

L'installation et la mise à jour automatiques sont désactivées. Les variables
SQL servent notamment aux contrôles réseau de l'image ; elles ne réécrivent
pas le fichier restauré `config_db.php`. Conserver les mêmes identifiants dans
les deux endroits. Ne pas mettre le mot de passe ni les sauvegardes dans Git.

## Préparer la migration

Le passage **10.0.17 → 10.0.28** nécessite une mise à jour manuelle de la base.
Si la cible est `glpidb`, cette opération modifie la base de production distante.
Avant toute mise à jour, arrêter l'ancien GLPI, ses accès web/API/inventaire et
ses tâches cron/collecteurs, et disposer d'une sauvegarde SQL et des fichiers.
Le mode maintenance seul n'arrête pas toutes les écritures externes.
Ne pas arrêter globalement les services partagés avec d'autres sites.

Pour tester en parallèle, faire préparer une **copie séparée de la base** et
modifier à la fois les variables Coolify et le `config_db.php` de la copie.
Ne pas faire tourner les deux versions sur la même base. Ne pas lancer de
collecteurs, notifications ni tâches automatiques pendant les essais.

## Sauvegarder sur web-1

Comme root dans Bash, après arrêt des écritures :

```bash
set -euo pipefail
cd /var/www/html/gestion.alexandre-lb.fr
sudo -u www-data php8.3 bin/console maintenance:enable
install -d -m 700 /root/glpi-migration
umask 077
mariadb --host=bdd.alexandre-lb.fr --port=3306 --user=glpiuser --password \
  glpidb -e 'SELECT VERSION();'
mariadb-dump --host=bdd.alexandre-lb.fr --port=3306 \
  --user=glpiuser --password --single-transaction --quick --hex-blob \
  --default-character-set=utf8mb4 glpidb > /root/glpi-migration/glpi.sql
test -s /root/glpi-migration/glpi.sql
tar -czpf /root/glpi-migration/glpi-data.tar.gz \
  -C /var/www/html/gestion.alexandre-lb.fr config files marketplace plugins
cd /root/glpi-migration
sha256sum glpi.sql glpi-data.tar.gz > SHA256SUMS
```

Le client demande le mot de passe sans le placer dans la commande. Vérifier
que le dump réussit ; en cas de droits insuffisants, le faire réaliser par
l'administrateur de la base. `--single-transaction` suppose des tables InnoDB ;
les tables non transactionnelles doivent rester sans écritures pendant le dump.
Conserver la sauvegarde hors du serveur et hors Git.

## Préparer les fichiers sur l'hôte Coolify

Si Coolify est également sur `web-1`, les sauvegardes y sont déjà ; sinon,
transférer `/root/glpi-migration/` par SSH/SCP. Pour une nouvelle destination vide :

```bash
set -euo pipefail
cd /root/glpi-migration
sha256sum -c SHA256SUMS
test ! -e /data/glpi/restore
test ! -e /data/glpi/data
test ! -e /data/glpi/marketplace
test ! -e /data/glpi/plugins
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
```

Si une des destinations existe déjà, inspecter son contenu au lieu de l'écraser
ou de réexécuter ce bloc. Les chemins sont explicites dans le Compose ; pour
les changer, modifier les champs `source` et adapter les commandes ci-dessus.
Conserver `config/config_db.php` et **glpicrypt.key** restaurés. Pour la base
existante, ne pas remplacer l'hôte par `db` : il reste `bdd.alexandre-lb.fr`.
Vérifier que les quatre propriétés SQL correspondent aux variables Coolify.

## Déployer et vérifier la connexion

Déployer sans ouvrir l'accès public à l'application. Le conteneur doit pouvoir
résoudre le nom de la base et joindre son port 3306. Autoriser la source réseau
réelle du serveur Coolify sur le pare-feu et dans les droits SQL de `glpiuser`.
`localhost` dans un conteneur désigne le conteneur lui-même.

### Vérifier le stockage et le routage Coolify

Après modification du Compose dans Git, recharger sa définition dans Coolify
avant de redéployer. Dans Persistent Storage, vérifier les chemins ci-dessous
et retirer les anciennes entrées incorrectes vers `/data`, `/marketplace` ou
`/plugins` si elles ont été conservées. Ne pas supprimer les données sur disque.

| Source sur web-1 | Destination dans le conteneur |
| --- | --- |
| `/data/glpi/data` | `/var/glpi` |
| `/data/glpi/marketplace` | `/var/www/glpi/marketplace` |
| `/data/glpi/plugins` | `/var/www/glpi/plugins` |

Le Compose emploie la syntaxe longue des montages avec des chemins fixes pour
éviter une mauvaise interprétation des séparateurs par Coolify. Les dossiers
doivent être préparés avant de déployer (`create_host_path: false`).

Dans Domains, associer explicitement le domaine au **service glpi, port 80**.
Le nom de l'application Coolify ne configure pas à lui seul le routage.
Si le conteneur répond en HTTP mais n'a aucun label `traefik.*`, vérifier cette
association et que la génération des labels du proxy n'a pas été désactivée.
Si GLPI redirige vers `install/install.php`, vérifier que `config_db.php` et les
données restaurées sont bien visibles dans `/var/glpi`. Ne pas lancer une
nouvelle installation sur la base existante.

Repérer le conteneur GLPI puis vérifier la connexion sans afficher le mot de passe :

```bash
docker ps --format 'table {{.ID}}\t{{.Names}}\t{{.Image}}'
GLPI_CID='REMPLACER_PAR_ID_GLPI'
docker exec "$GLPI_CID" php -r '
mysqli_report(MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT);
$db = new mysqli(getenv("GLPI_DB_HOST"), getenv("GLPI_DB_USER"),
    getenv("GLPI_DB_PASSWORD"), getenv("GLPI_DB_NAME"), (int)getenv("GLPI_DB_PORT"));
echo "Connexion SQL OK", PHP_EOL;
'
```

Il n'y a **aucun dump à importer dans la base existante**. Le dump est la
sauvegarde de retour arrière. Pour un test sur une copie, l'administrateur SQL
restaure le dump dans une nouvelle base vide, jamais sur `glpidb` en activité.

## Mettre à jour et basculer

Seulement après sauvegarde vérifiée et arrêt de l'ancien GLPI :

```bash
docker exec -u www-data "$GLPI_CID" \
  php bin/console database:update --no-interaction
docker exec -u www-data "$GLPI_CID" php bin/console database:check_schema_integrity
docker exec -u www-data "$GLPI_CID" php bin/console cache:clear
```

Arrêter si une commande échoue. Vérifier toute différence de schéma avant la
bascule. Les extensions peuvent être désactivées par la mise à jour ; vérifier
leur compatibilité avec GLPI 10.0.28 avant de les réactiver.

Après validation, désactiver la maintenance sur la nouvelle instance :

```bash
docker exec -u www-data "$GLPI_CID" php bin/console maintenance:disable
```

Puis affecter `https://gestion.alexandre-lb.fr` au service `glpi`, port 80, et
basculer les DNS A/AAAA si nécessaire. Si l'ancien Nginx sert toujours le domaine,
retirer son routage GLPI avant d'ouvrir le nouveau service ; la maintenance est
stockée dans la base partagée et ne protège pas séparément l'ancienne instance.
Sur un même serveur, vérifier aussi que Nginx et le proxy Coolify ne se disputent
pas les ports 80/443.

Vérifier connexion, tickets, anciens documents, droits, API/inventaire, LDAP,
SMTP et les 16 extensions présentes dans `marketplace`. Reprendre l'URL de
production dans la configuration générale GLPI. Passer ensuite
`GLPI_CRONTAB_ENABLED=1` dans Coolify et redéployer ; vérifier les actions
automatiques en mode CLI et les collecteurs. Ne pas ajouter un deuxième cron.
L'ancien GLPI doit rester inaccessible et ses tâches arrêtées.

## Retour arrière et sauvegardes régulières

Avant `database:update`, on peut arrêter la nouvelle application et rétablir
l'ancien accès. Après `database:update`, arrêter toutes les écritures et faire
restaurer la sauvegarde SQL 10.0.17 par l'administrateur sur une destination
propre, avec les fichiers cohérents, avant de relancer l'ancien code. Un simple
retour DNS ne restaure pas le schéma. Les écritures intervenues après la
sauvegarde nécessitent un plan de récupération avant tout retour arrière.

Sauvegarder ensemble **la base distante et `/data/glpi`**. Coolify ne sauvegarde
pas automatiquement la base distante avec les fichiers. Une rotation SQL du mot
de passe doit aussi être reportée dans Coolify et dans `config_db.php`.

## Vérification locale du Compose

```bash
GLPI_DB_PASSWORD=validation \
  docker compose -f coolify/docker-compose.yml config --quiet
```

Références : [image officielle](https://github.com/glpi-project/docker-images),
[migration GLPI](https://glpi-install.readthedocs.io/en/latest/update.html),
[Docker Compose dans Coolify](https://coolify.io/docs/applications/builds/docker-compose).
