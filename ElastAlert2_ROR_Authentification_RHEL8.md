# ElastAlert2 avec ReadonlyREST (ROR 1.72) sur Elasticsearch 8.16 : authentification par utilisateur

Contexte : ROR remplace la sécurité X-Pack. L'API `_security/*` n'existe donc pas (erreur « no handler »), et les clés d'API Elasticsearch (§6.1 du runbook) ne s'appliquent pas. ElastAlert2 s'authentifie avec un utilisateur défini dans `readonlyrest.yml`.

Sources :
- [Documentation ROR Elasticsearch](https://docs.readonlyrest.com/elasticsearch) : `auth_key_pbkdf2`, ordre des blocs, règles `indices` et `actions`, `force_load_from_file`
- [API ROR](https://docs.readonlyrest.com/kibana/readonlyrest-api) : le rechargement à chaud est réservé à Enterprise
- [Configuration ElastAlert2](https://elastalert2.readthedocs.io/en/latest/configuration.html) : `es_username`, `es_password`, `ES_USERNAME`, `ES_PASSWORD`, `ES_API_KEY`

---

## 1. Générer le hash PBKDF2

`auth_key_pbkdf2` utilise HmacSHA512, 10000 itérations et une clé de 512 bits. Le sel est la valeur elle-même. Le nom d'utilisateur reste en clair (`user:hash(mot de passe)`).

Le mot de passe est demandé de façon masquée : il n'apparaît ni à l'écran ni dans l'historique.

```bash
python3 -c 'import getpass,hashlib,base64; p=getpass.getpass("Mot de passe elastalert: ").encode(); print("elastalert:"+base64.b64encode(hashlib.pbkdf2_hmac("sha512",p,p,10000,64)).decode())'
```

Tu obtiens une ligne `elastalert:<base64>`. Ce n'est pas le mot de passe, mais ne la partage pas pour autant.

## 2. Bloc à ajouter dans `/etc/elasticsearch/readonlyrest.yml`

ROR applique le **premier** bloc qui correspond. Place ces deux blocs **avant** tout bloc plus large (admin, `indices: ["*"]`, etc.).

```yaml
readonlyrest:
  # ssl: ... (ne change rien)
  access_control_rules:

    - name: "elastalert - ecriture writeback"
      type: allow
      auth_key_pbkdf2: "elastalert:<COLLE_LE_HASH_ICI>"
      indices: ["elastalert_*"]
      actions: ["indices:*", "cluster:monitor/*"]

    - name: "elastalert - lecture logs"
      type: allow
      auth_key_pbkdf2: "elastalert:<COLLE_LE_HASH_ICI>"
      indices: ["logs-*"]
      actions: ["indices:data/read/*", "indices:admin/mappings/get", "indices:admin/get", "indices:admin/resolve/*", "cluster:monitor/*"]

    # ... tes blocs existants en dessous
```

Les `actions` sont nécessaires : sans elles, la règle `indices` laisse passer les requêtes qui ne visent aucun index, y compris l'administration du cluster. Ici, ElastAlert2 n'a que :
- la lecture sur `logs-*` (recherche, scroll, `_count`, `_msearch`, mappings) ;
- tous les droits sur `elastalert_*` (création des index et alias de writeback) ;
- `cluster:monitor/*` (`GET /`, `_cluster/health`).

> ⚠️ **Non vérifié :** la liste exacte des actions qu'utilise ElastAlert2. En cas de 403, le log ES donne le nom de l'action refusée : ajoute-la au bloc concerné.

## 3. Appliquer la configuration ROR

En version gratuite, il n'y a pas de rechargement à chaud. Il faut redémarrer ES :

```bash
systemctl restart elasticsearch
journalctl -u elasticsearch -n 50 --no-pager | grep -iE "readonlyrest|error"
```

> ⚠️ Si l'index `.readonlyrest` existe, ROR y lit sa configuration et ignore le fichier. Si tes changements ne sont pas pris en compte, ajoute `readonlyrest.force_load_from_file: true` dans `/etc/elasticsearch/elasticsearch.yml`, puis redémarre.

## 4. Côté ElastAlert2

Dans `/etc/elastalert2/config.yaml` :
- supprime `es_api_key` s'il y est ;
- garde `use_ssl: true`, `verify_certs: true` et `ca_certs: /etc/elastalert2/certs/ca.crt` ;
- ne mets **aucun** identifiant dans ce fichier.

Les variables d'environnement `ES_USERNAME` et `ES_PASSWORD` remplacent `es_username` et `es_password` du fichier. Attention : `ES_API_KEY` a priorité sur les autres méthodes, d'où l'importance de la retirer.

Édite l'env dans l'éditeur, sans `echo` du mot de passe :

```bash
vi /etc/elastalert2/elastalert2.env
```

```
# supprime la ligne ES_API_KEY=... si elle existe
ES_USERNAME=elastalert
ES_PASSWORD=<tape le mot de passe ici dans l'éditeur>
```

```bash
chown root:elastalert2 /etc/elastalert2/elastalert2.env
chmod 0640 /etc/elastalert2/elastalert2.env
grep -c '^ES_API_KEY' /etc/elastalert2/elastalert2.env   # doit afficher 0
```

## 5. Vérifications (aucun secret affiché)

Avec `-u elastalert` sans `:`, curl demande le mot de passe sans l'afficher.

```bash
ES=https://<hote_es>:9200; CA=/etc/elastalert2/certs/ca.crt
curl --cacert $CA -u elastalert "$ES/_cluster/health?pretty"     # 200 attendu
curl --cacert $CA -u elastalert "$ES/_cat/indices/logs-*?v"     # liste des index logs
curl --cacert $CA -u elastalert -XPUT "$ES/elastalert_test/_doc/1" -H 'Content-Type: application/json' -d '{"t":1}'   # "created"
curl --cacert $CA -u elastalert -XDELETE "$ES/elastalert_test"   # nettoyage
curl --cacert $CA -u elastalert "$ES/.kibana/_search" -o /dev/null -w '%{http_code}\n'   # 403 attendu (accès refusé)
```

Ensuite, crée les index et redémarre le service. Remplace `/chemin/venv` par le chemin du runbook :

```bash
set -a; . /etc/elastalert2/elastalert2.env; set +a
sudo -E -u elastalert2 /chemin/venv/bin/elastalert-create-index --config /etc/elastalert2/config.yaml
unset ES_USERNAME ES_PASSWORD
systemctl restart elastalert2
journalctl -u elastalert2 -n 50 --no-pager | grep -iE "error|403|401"
```

`set -a` charge les variables sans les afficher.

> ⚠️ **Non vérifié :** si `elastalert-create-index` crée aussi un template d'index. Si tu vois un 403 sur une action `template`, ajoute `indices:admin/template/*` et `indices:admin/index_template/*` au premier bloc, puis redémarre ES.

## 6. Dépannage rapide

| Symptôme | Cause probable | Action |
|---|---|---|
| 401 | Hash ou mot de passe incorrect | Régénère le hash (§1) et vérifie `ES_PASSWORD` |
| 403 sur `logs-*` ou `elastalert_*` | Action absente du bloc | Ajoute l'action citée dans le log ES |
| Changements ROR ignorés | Config lue dans l'index `.readonlyrest` | `force_load_from_file: true` (§3) |
| ElastAlert2 tente encore une clé d'API | `ES_API_KEY` encore présente | Supprime-la de l'env et de `config.yaml` |
| Bloc ignoré | Un bloc plus large le précède | Remonte les blocs elastalert en tête |
