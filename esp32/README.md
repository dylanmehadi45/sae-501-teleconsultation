# Code ESP32

Programme Arduino de l'ESP32 : lecture du capteur I²C et publication des mesures sur le broker MQTT en TLS.

## À déposer ici
- le sketch `.ino` (code capteur + client MQTT/TLS) ;
- le certificat CA est chargé dans le code mais **n'est pas versionné** (voir `.gitignore`).

## Câblage du capteur I²C
| Capteur | ESP32 |
|---------|-------|
| VIN | 3V3 |
| GND | GND |
| SDA | GPIO 21 |
| SCL | GPIO 22 |

## Paramètres à renseigner (sans les publier en clair)
```cpp
const char* WIFI_SSID     = "SAE-501-Teleconsultation";
const char* WIFI_PASSWORD = "<MOT_DE_PASSE_WIFI>";   // à garder hors dépôt
const char* MQTT_BROKER   = "<IP_DU_BROKER>";
const uint16_t MQTT_PORT  = 8883;
const char* MQTT_TOPIC    = "teleconsultation/patient/mesures";
```

> Le modèle du capteur doit être confirmé avant de figer la bibliothèque Arduino.
