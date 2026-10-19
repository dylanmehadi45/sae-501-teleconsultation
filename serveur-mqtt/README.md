# Serveur MQTT — Mosquitto

Broker Mosquitto sur une VM Ubuntu. C'est le point central : l'ESP32 y publie, le serveur web s'y abonne.

## Ports
| Port | Service | Rôle |
|------|---------|------|
| 1883 | MQTT | non chiffré, tests uniquement |
| 8883 | MQTTS | MQTT chiffré TLS (celui qu'on utilise) |
| 9001 | MQTT WebSocket | pour un client WebSocket |

Topic des mesures : `teleconsultation/patient/mesures`

## Vérifs utiles
```bash
sudo ss -tulpn | grep mosquitto      # ports écoutés
sudo systemctl status mosquitto       # état du service
sudo journalctl -u mosquitto -f       # logs en direct
```

## Certificats
Les fichiers TLS (`ca.crt`, `server.crt`, `server.key`) vivent dans `/etc/mosquitto/certs/` sur la VM et **ne sont pas versionnés ici**. Seul le `ca.crt` est à distribuer aux clients (ESP32 + VM web) ; la clé privée du serveur ne se partage jamais.
