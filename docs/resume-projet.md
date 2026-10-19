# Résumé du projet SAE501 — Maquette de téléconsultation

## Objectif du projet

Le projet consiste à réaliser une maquette de **téléconsultation sécurisée**. Le but est de récupérer des mesures provenant d’un capteur connecté à un ESP32, de les transmettre de manière chiffrée vers un serveur MQTT, puis de les afficher en temps réel sur un site web.

Le projet répond au sujet de téléconsultation de la SAE501 : récupération de données médicales ou environnementales, transmission réseau, affichage pour un utilisateur distant et sécurisation des échanges.

## Architecture prévue

```text
┌──────────────────────┐
│ Capteur I²C          │
│ Température          │
│ SDA / SCL / VIN/GND  │
└──────────┬───────────┘
           │ I²C
           ▼
┌──────────────────────┐
│ ESP32                │
│ Wi-Fi 2,4 GHz        │
│ Client MQTT/TLS      │
└──────────┬───────────┘
           │ MQTT chiffré TLS — port 8883
           ▼
┌──────────────────────────────┐
│ VM 1 — Serveur Mosquitto     │
│ Broker MQTT                  │
│ 1883 / 8883 / 9001           │
└──────────┬───────────────────┘
           │ MQTT TLS
           ▼
┌──────────────────────────────┐
│ VM 2 — Serveur Web           │
│ Flask + Socket.IO            │
│ Abonné MQTT / affichage      │
└──────────┬───────────────────┘
           │ HTTP — port 5000
           ▼
┌──────────────────────┐
│ Navigateur web       │
│ Mesures en direct    │
└──────────────────────┘
```

Les rôles sont volontairement séparés : l’ESP32 représente l’équipement patient, la première VM héberge le broker Mosquitto et la seconde VM héberge l’application web destinée au professionnel de santé.

## Réseau Wi-Fi

Un point d’accès DD-WRT dédié au projet a été configuré.

| Élément | Configuration |
|---|---|
| SSID | `SAE-501-Teleconsultation` |
| Bande | 2,4 GHz |
| Sécurité | WPA2-PSK |
| Chiffrement | AES / CCMP-128 |
| Renouvellement de clé | 3 600 secondes |
| Rôle | Connecter ESP32, VM Mosquitto et VM web au même LAN |

Le Wi-Fi 2,4 GHz est nécessaire car l’ESP32 fonctionne sur cette bande. Pour vérifier la fréquence exacte dans DD-WRT : `Wireless > Basic Settings > Wireless Channel`. Le canal 1 correspond à 2,412 GHz.

## Serveur MQTT TLS

Le broker Mosquitto est installé sur une VM Ubuntu. Les ports ont été vérifiés avec :

```bash
sudo ss -tulpn | grep mosquitto
```

| Port | Service | Rôle |
|---:|---|---|
| 1883 | MQTT | Connexion non chiffrée, uniquement utile pour tests |
| 8883 | MQTTS | MQTT chiffré avec TLS |
| 9001 | MQTT WebSocket | MQTT possible depuis un client WebSocket |

Le topic utilisé pour les mesures est :

```text
teleconsultation/patient/mesures
```

### Tests déjà validés

Le fonctionnement MQTT/TLS a été validé localement sur la VM Mosquitto.

Écoute du topic :

```bash
mosquitto_sub -h localhost -p 8883   --cafile /etc/mosquitto/certs/ca.crt   --insecure   -t teleconsultation/patient/mesures -v
```

Publication de test :

```bash
mosquitto_pub -h localhost -p 8883   --cafile /etc/mosquitto/certs/ca.crt   --insecure   -t teleconsultation/patient/mesures   -m '{"temperature":22.5,"humidite":45.0}'
```

Le message a bien été reçu :

```text
teleconsultation/patient/mesures {"temperature":22.5,"humidite":45.0}
```

Cela confirme que la publication, la souscription et le chiffrement TLS sont fonctionnels localement.

### Commandes utiles Mosquitto

```bash
# Redémarrer le broker et vérifier son état
sudo systemctl restart mosquitto
sudo systemctl status mosquitto
sudo systemctl enable mosquitto

# Vérifier les ports écoutés
sudo ss -tulpn | grep mosquitto

# Suivre les logs en temps réel
sudo journalctl -u mosquitto -f

# Écouter tous les topics, utile pour le débogage
mosquitto_sub -h localhost -p 8883   --cafile /etc/mosquitto/certs/ca.crt   --insecure   -t '#' -v
```

## Certificats et sécurité

Les fichiers TLS importants sur la VM Mosquitto sont normalement :

```text
/etc/mosquitto/certs/ca.crt
/etc/mosquitto/certs/server.crt
/etc/mosquitto/certs/server.key
```

| Fichier | Rôle | Diffusion |
|---|---|---|
| `ca.crt` | Autorité de certification | À copier vers ESP32 et VM web |
| `server.crt` | Certificat présenté par Mosquitto | Reste sur la VM broker |
| `server.key` | Clé privée du serveur | Ne jamais partager |

Le certificat CA doit être intégré dans l’ESP32 et copié sur la VM web, car ces deux éléments sont des clients MQTT qui doivent vérifier le serveur Mosquitto.

### Point de sécurité restant

Le certificat a été généré pour l’IP `192.168.1.148`, tandis que l’IP actuelle annoncée du broker est `192.168.1.145`.

Cette incohérence empêche potentiellement la validation stricte de l’identité du serveur. Les tests utilisent donc temporairement :

```bash
--insecure
```

et dans le programme ESP32 :

```cpp
secureClient.setInsecure();
```

Le transport reste chiffré, mais la vérification de l’identité du broker est désactivée. La solution finale prévue est de régénérer le certificat avec l’IP actuelle ou un nom DNS dans le champ SAN, puis de supprimer ces options temporaires.

## Serveur web

Le serveur web est prévu sur une VM distincte. Il est basé sur :

- Python 3
- Flask
- Flask-SocketIO
- Paho MQTT
- Eventlet
- HTML, CSS et JavaScript

Fonctionnement :

1. `app.py` se connecte au broker Mosquitto avec MQTT/TLS.
2. Il s’abonne au topic `teleconsultation/patient/mesures`.
3. Il reçoit les messages JSON publiés par l’ESP32.
4. Il transmet les valeurs au navigateur via Socket.IO.
5. `index.html` affiche les valeurs en direct sans rechargement de page.

### Fichiers créés

Les fichiers suivants ont été préparés :

```text
app.py
index.html
setup_site_web_teleconsultation.md
passation_ia_sae501_teleconsultation.md
```

Organisation attendue sur la VM web :

```text
~/teleconsultation/
├── app.py
└── templates/
    └── index.html
```

Mise en place :

```bash
mkdir -p ~/teleconsultation/templates
mv ~/Desktop/app.py ~/teleconsultation/
mv ~/Desktop/index.html ~/teleconsultation/templates/
cd ~/teleconsultation
```

### Paramètres à adapter dans app.py

```python
BROKER_HOST = "192.168.1.145"
BROKER_PORT = 8883
CA_CERT_PATH = "/home/UTILISATEUR/ca.crt"
MQTT_TOPIC = "teleconsultation/patient/mesures"
```

Pour connaître l’utilisateur Linux de la VM :

```bash
whoami
echo $HOME
```

### Installer les dépendances web

```bash
sudo apt update
sudo apt install python3-pip python3-full python3-venv
python3 -m venv ~/venv-teleconsultation
source ~/venv-teleconsultation/bin/activate
pip install flask flask-socketio paho-mqtt eventlet
```

Un environnement virtuel est particulièrement nécessaire sur Kali Linux, qui bloque l’installation `pip` globale avec l’erreur `externally-managed-environment`.

### Lancer le site

```bash
source ~/venv-teleconsultation/bin/activate
cd ~/teleconsultation
python3 app.py
```

Le site doit être accessible sur :

```text
http://IP_DE_LA_VM_WEB:5000
```

Test MQTT depuis la VM web avant lancement :

```bash
mosquitto_sub -h 192.168.1.145 -p 8883   --cafile ~/ca.crt   --insecure   -t teleconsultation/patient/mesures -v
```

## Réseau et VirtualBox

Le mode NAT a été utilisé pour faciliter l’accès Internet et l’installation des dépendances, mais il ne convient pas à la communication directe finale entre l’ESP32, le broker et la VM web.

Pour la démonstration finale, le mode recommandé est **Accès par pont** sur les deux VM :

```text
VirtualBox > Configuration > Réseau > Adaptateur 1
Mode d’accès réseau : Accès par pont
Nom : Intel(R) Wi-Fi 6 AX201 160MHz
Câble branché : activé
```

Le PC hôte doit être connecté au réseau `SAE-501-Teleconsultation` avant le démarrage des VM. Ainsi, chaque VM reçoit une IP sur le même réseau local que l’ESP32.

Le réseau universitaire a provoqué des difficultés possibles de bridge, DHCP et DNS, en raison de mécanismes de sécurité et de filtrage. Le routeur DD-WRT dédié est donc préférable pour les tests finaux.

## Problèmes rencontrés

- Erreur `Protocol error` pendant les premiers tests TLS.
- Faute de frappe `--inscure` au lieu de `--insecure`.
- Décalage entre l’IP contenue dans le certificat et l’IP actuelle du broker.
- Difficultés de bridge VirtualBox sur le réseau universitaire.
- Limites du NAT pour l’architecture multi-équipements.
- Problèmes DNS sur certaines VM : accès IP fonctionnel mais résolution de nom défaillante.
- Erreur `externally-managed-environment` avec Kali Linux et `pip3 install`.
- Modèle exact du capteur I²C pas encore confirmé.

## ESP32 et capteur

Les paramètres réseau prévus dans le code ESP32 sont :

```cpp
const char* WIFI_SSID = "SAE-501-Teleconsultation";
const char* WIFI_PASSWORD = "<MOT_DE_PASSE_WIFI>";

const char* MQTT_BROKER = "192.168.1.145";
const uint16_t MQTT_PORT = 8883;
const char* MQTT_TOPIC = "teleconsultation/patient/mesures";
```

Le capteur disponible possède les broches :

```text
VIN / GND / SDA / SCL
```

Il s’agit donc d’un capteur I²C. Le câblage ESP32 proposé est :

| Capteur I²C | ESP32 |
|---|---|
| VIN | 3V3 |
| GND | GND |
| SDA | GPIO 21 |
| SCL | GPIO 22 |

### Point bloquant à résoudre

Le modèle du capteur I²C n’est pas encore identifié. Il ne faut pas finaliser le code capteur sans connaître son marquage.

| Capteur possible | Bibliothèque Arduino | Mesures |
|---|---|---|
| BME280 | Adafruit BME280 | Température, humidité, pression |
| BMP280 | Adafruit BMP280 | Température, pression |
| SHT30 / SHT31 | Adafruit SHT31 | Température, humidité |
| MLX90614 | Adafruit MLX90614 | Température infrarouge |
| MCP9808 | Adafruit MCP9808 | Température |

La prochaine action est d’identifier le module par une photo nette ou en lisant le marquage imprimé sur sa carte.

## Plan de réalisation restant

1. Identifier le modèle exact du capteur I²C.
2. Adapter le programme ESP32 avec la bonne bibliothèque.
3. Vérifier l’IP réelle actuelle de la VM Mosquitto.
4. Mettre les VM Mosquitto et WebServer en bridge sur le routeur DD-WRT.
5. Vérifier que toutes les machines reçoivent une IP du même réseau.
6. Copier `ca.crt` de la VM Mosquitto vers la VM web.
7. Installer et lancer le serveur Flask sur la VM web.
8. Tester l’abonnement MQTT/TLS depuis la VM web.
9. Flasher le programme ESP32 adapté au capteur.
10. Vérifier l’arrivée des messages avec `mosquitto_sub`.
11. Vérifier la mise à jour temps réel sur le site web.
12. Régénérer le certificat TLS avec le bon SAN/IP.
13. Retirer `--insecure` et `secureClient.setInsecure()`.

## Résultat final attendu

```text
Capteur → ESP32 → MQTT/TLS → Mosquitto → VM Web → Navigateur
```

Exemple de message JSON attendu :

```json
{
  "temperature": 22.5,
  "humidite": 45.0,
  "pression": 1013.2
}
```

Le site final doit afficher les mesures actualisées en temps réel, l’état de connexion, l’heure de dernière réception et une alerte si une température anormale est détectée.

## Confidentialité

Le mot de passe Wi-Fi de test et les certificats ne doivent pas être publiés dans un dépôt Git public, une capture de soutenance publique ou une documentation diffusée hors du groupe. Il est préférable d’utiliser un fichier de configuration ignoré par Git pour les secrets.
