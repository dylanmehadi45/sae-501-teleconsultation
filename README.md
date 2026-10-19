# SAÉ 501 — Maquette de téléconsultation sécurisée

Projet de BUT Réseaux & Télécoms (parcours Cybersécurité, IUT d'Auxerre). Objet : une petite maquette de téléconsultation où un capteur mesure une donnée côté « patient », l'envoie **chiffrée** sur le réseau, et où un professionnel de santé la voit s'afficher **en temps réel** dans son navigateur.

C'est un projet en cours : la chaîne MQTT/TLS est validée, il reste à finaliser le code capteur (modèle à confirmer) et le certificat. L'état d'avancement est détaillé plus bas.

## Le principe

Une donnée part d'un capteur et traverse toute la chaîne jusqu'à un navigateur, en restant chiffrée sur le réseau :

```
Capteur I²C ──I²C──> ESP32 ──MQTT/TLS (8883)──> Mosquitto (broker)
                                                      │
                                                      ▼
                                        Serveur web Flask + Socket.IO
                                                      │ HTTP (5000)
                                                      ▼
                                              Navigateur (temps réel)
```

Les rôles sont volontairement séparés, comme dans un vrai système : l'ESP32 est l'équipement « patient », une VM héberge le broker MQTT (Mosquitto) et une autre VM le site web du « soignant ». Les échanges passent par un point d'accès Wi-Fi dédié (routeur DD-WRT, WPA2).

## Le capteur

Un capteur **I²C** (broches VIN / GND / SDA / SCL), câblé sur l'ESP32 :

| Capteur | ESP32 |
|---------|-------|
| VIN | 3V3 |
| GND | GND |
| SDA | GPIO 21 |
| SCL | GPIO 22 |

> Le modèle exact reste à confirmer sur le marquage de la carte (candidats probables : BME280, BMP280, SHT30/31, MCP9808…). Le code capteur sera finalisé une fois le modèle identifié, avec la bonne bibliothèque Arduino.

## La stack technique

- **Côté capteur :** ESP32 (Wi-Fi 2,4 GHz), client MQTT en TLS
- **Transport :** MQTT chiffré TLS sur le port **8883**, topic `teleconsultation/patient/mesures`
- **Broker :** Mosquitto sur une VM Ubuntu (1883 test / 8883 MQTTS / 9001 WebSocket)
- **Serveur web :** Python 3, Flask, Flask-SocketIO, Paho-MQTT, Eventlet — pousse les mesures au navigateur en direct via Socket.IO
- **Réseau :** point d'accès DD-WRT dédié (`SAE-501-Teleconsultation`, WPA2 AES/CCMP), VM en accès par pont pour la démo

Format des messages échangés :

```json
{ "temperature": 22.5, "humidite": 45.0, "pression": 1013.2 }
```

## La sécurité

Tout le transport est chiffré en TLS (MQTTS). Le certificat CA sert aux clients (ESP32 et VM web) à vérifier l'identité du broker.

Deux règles que je respecte dans ce dépôt :
- **aucun certificat ni clé privée** (`*.key`, `*.crt`) n'est versionné — ils sont dans le `.gitignore` ;
- **aucun secret** (mot de passe Wi-Fi, identifiants) n'est publié ; ils sont remplacés par des placeholders à renseigner en local.

Point technique en cours : le certificat a été généré pour une IP qui ne correspond plus à celle du broker, donc les tests tournent temporairement en mode non-strict. La correction prévue est de régénérer le certificat avec le bon SAN/IP et de réactiver la vérification complète.

## Où en est le projet

**Fait :** réseau Wi-Fi dédié, broker Mosquitto en TLS installé et testé localement (publication + souscription OK), serveur web préparé.

**Reste à faire :** identifier le capteur et finaliser son code, basculer les VM en pont sur le DD-WRT, intégrer le CA côté web et ESP32, lancer la chaîne complète, puis régénérer le certificat et retirer le mode non-strict.

## Organisation du dépôt

```
esp32/          → code Arduino de l'ESP32 (capteur + MQTT/TLS)
serveur-mqtt/   → configuration du broker Mosquitto
serveur-web/    → application Flask + Socket.IO (app.py, templates/)
ova/            → machines virtuelles (voir le README du dossier)
docs/           → résumé complet du projet
```

---
Projet académique — BUT R&T, IUT d'Auxerre.
