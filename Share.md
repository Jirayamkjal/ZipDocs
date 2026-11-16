sudo -u elastalert2 bash -c 'set -a; . /etc/elastalert2/elastalert2.env; set +a; echo "user=$ES_USERNAME"; [ -n "$ES_PASSWORD" ] && echo "password: présent" || echo "password: ABSENT"'

sudo -u elastalert2 bash -c 'set -a; . /etc/elastalert2/elastalert2.env; set +a; exec /opt/elastalert2/venv/bin/python -m elastalert.elastalert --config /etc/elastalert2/config.yaml --verbose'



Voici le courriel. La pièce jointe sera 04-feuille-de-route.md, que je finis de préparer.
Objet : Incident récurrent d'ingestion Logstash vers Elasticsearch : analyse et plan de correction
Bonjour à tous,
Depuis quelques semaines, on observe des interruptions de plus en plus fréquentes de l'ingestion des logs vers Elasticsearch. Les données s'accumulent dans la file persistante de Logstash (serveur SFPPRL17728, version 8.19). Jusqu'ici, on rétablissait le service en vidant la file et en redémarrant Logstash, ce qui entraîne une perte de données.
Analyse
Les logs du 16 novembre, vers 2 h, montrent que les requêtes bulk vers es-prod.gestion:9200 expirent après 120 secondes (Manticore::SocketTimeout). Logstash marque alors la connexion comme morte et renvoie le même lot en boucle, pendant que la file grossit jusqu'à 4 Go. Logstash lui-même n'était pas surchargé. Le goulot se trouve en aval, du côté de la VIP ou du cluster Elasticsearch.
Trois facteurs aggravent la situation :
la configuration compte environ 25 sorties Elasticsearch distinctes, chacune avec son propre pool de connexions, toutes dirigées vers la même VIP;
on crée un index par client, par source et par jour, ce qui fait exploser le nombre de shards et ralentit les écritures;
aucune dead letter queue n'est configurée, donc les documents rejetés (erreurs 400) sont perdus sans alerte.
Solution proposée
Stabilisation immédiate : remplacer la suppression de la file par une procédure de récupération sûre (sauvegarde, pqcheck, pqrepair) et vérifier les timeouts, la persistance de session et les health checks de la VIP.
Court terme : activer la dead letter queue avec une alerte par courriel et superviser la taille de la file persistante.
Moyen terme : regrouper les 25 sorties en une seule sortie dynamique et réduire le nombre de shards avec ILM et des index moins granulaires.
Long terme : revoir la capacité du cluster et mettre en place des alertes Stack Monitoring.
La feuille de route jointe détaille les phases, les priorités, les dépendances et les critères de validation. Chaque décision fait l'objet d'un ADR, que je peux vous transmettre sur demande.
Je vous propose une courte rencontre cette semaine pour valider la phase 0 et attribuer les responsables.
Cordialement,
Karim Jalal
Architecte technologique
