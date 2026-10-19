# Serveur web — Flask + Socket.IO

Application qui s'abonne au broker MQTT (TLS), reçoit les mesures JSON et les affiche en direct dans le navigateur, sans rechargement de page.

## À déposer ici
```
serveur-web/
├── app.py
└── templates/
    └── index.html
```

## Installation
```bash
sudo apt update
sudo apt install python3-pip python3-full python3-venv
python3 -m venv ~/venv-teleconsultation
source ~/venv-teleconsultation/bin/activate
pip install flask flask-socketio paho-mqtt eventlet
```

## Paramètres à adapter dans app.py
```python
BROKER_HOST = "<IP_DU_BROKER>"
BROKER_PORT = 8883
CA_CERT_PATH = "<CHEMIN_VERS_ca.crt>"   # hors dépôt
MQTT_TOPIC = "teleconsultation/patient/mesures"
```

## Lancement
```bash
source ~/venv-teleconsultation/bin/activate
cd ~/teleconsultation
python3 app.py
# → http://IP_DE_LA_VM_WEB:5000
```
