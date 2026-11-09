sudo -u elastalert2 bash -c 'set -a; . /etc/elastalert2/elastalert2.env; set +a; echo "user=$ES_USERNAME"; [ -n "$ES_PASSWORD" ] && echo "password: présent" || echo "password: ABSENT"'

sudo -u elastalert2 bash -c 'set -a; . /etc/elastalert2/elastalert2.env; set +a; exec /opt/elastalert2/venv/bin/python -m elastalert.elastalert --config /etc/elastalert2/config.yaml --verbose'
