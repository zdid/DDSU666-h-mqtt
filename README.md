# DDSU666-h-mqtt

Lit un compteur d'énergie monophasé **DDSU666-H** (Huawei / Chint) sur son bus **RS485 (Modbus RTU)** et publie les
mesures sur **MQTT**, avec **découverte automatique Home Assistant** : un appareil et 10 capteurs apparaissent seuls.

- Un seul fichier Python, **sans dépendance** : `python3` et `mosquitto_pub` (paquet `mosquitto-clients`) suffisent.
- Fonctionne sur les petites cartes **32 bits** (Raspberry Pi 3 sous Raspbian 32 bits…), où l'image Docker de
  `modbus2mqtt` n'existe pas pour l'architecture armv7.
- **Lecture seule** : le script n'envoie que des requêtes de lecture, il n'écrit jamais dans le compteur.

## Matériel

- Un compteur DDSU666-H (réglage usine : **adresse 11, 9600 bauds, 8N1** sur les Huawei ; adresse 1 sur les Chint).
- Un adaptateur **USB ↔ RS485** (puce CH340, CP2102…), relié aux bornes **A** et **B** du compteur.
- Un seul maître sur le bus : ne lancez pas deux programmes sur le même port série.

## Installation

```bash
sudo apt install python3 mosquitto-clients
sudo mkdir -p /opt/ddsu666h-mqtt
sudo cp ddsu666h-mqtt.py /opt/ddsu666h-mqtt/
```

Essai (une lecture, sans rien envoyer à MQTT, avec l'affichage des messages) :

```bash
python3 ddsu666h-mqtt.py --device /dev/ttyUSB0 --address 11 --once --dry-run
```

Service systemd (adaptez les options dans `ddsu666h-mqtt.service`) :

```bash
sudo cp ddsu666h-mqtt.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now ddsu666h-mqtt
```

## Options

Les réglages se donnent par la **ligne de commande**, par un **fichier de configuration**, ou les deux (voir plus bas).

| Option | Défaut | Rôle |
|---|---|---|
| `--config` | | fichier de configuration INI (voir plus bas) |
| `--device` | `/dev/ttyUSB0` | port série de l'adaptateur RS485 |
| `--baud` | `9600` | vitesse (2400, 4800, 9600, 19200, 38400) |
| `--parity` | `N` | parité : `N` (8N1) ou `E` (8E1) |
| `--address` | `11` | adresse Modbus du compteur |
| `--mqtt-host` / `--mqtt-port` | `127.0.0.1` / `1883` | broker MQTT |
| `--mqtt-user` | *(vide)* | nom d'utilisateur MQTT ; vide = connexion anonyme |
| `--mqtt-password` | *(vide)* | mot de passe MQTT (demande `--mqtt-user`) |
| `--prefix` | `homeassistant` | préfixe de découverte de Home Assistant |
| `--topic` | `ddsu666h` | thème MQTT de l'état (`<thème>/state`) |
| `--node` | `ddsu666h` | identifiant du nœud dans la découverte |
| `--name` | `Compteur DDSU666-H` | nom de l'appareil dans Home Assistant |
| `--interval` | `5` | secondes entre deux lectures (1 à 60) |
| `--once` | | une seule lecture puis sortie |
| `--dry-run` | | affiche les messages au lieu de les envoyer |

## Fichier de configuration

`--config fichier.conf` lit un fichier INI avec une section `[ddsu666h]` (modèle : `ddsu666h-mqtt.conf.example`). Les clés
sont les noms des options sans `--` (`mqtt-host` ou `mqtt_host`, indifféremment). **La ligne de commande a la priorité sur
le fichier, qui a la priorité sur les valeurs par défaut.**

```ini
[ddsu666h]
address = 11
mqtt-host = 192.168.1.10
mqtt-user = compteur
mqtt-password = secret
interval = 5
```

```bash
sudo install -m 600 ddsu666h-mqtt.conf.example /etc/ddsu666h-mqtt.conf   # puis l'éditer
python3 ddsu666h-mqtt.py --config /etc/ddsu666h-mqtt.conf
```

**Mot de passe MQTT** : préférez le fichier (droits `600`, le script avertit s'il est lisible par d'autres). Sur la
ligne de commande, le mot de passe est visible dans la liste des processus de la machine (`ps`), y compris dans celle
de `mosquitto_pub` qui le reçoit en argument à chaque publication.

## Ce qui est publié

L'état tient dans un seul message JSON sur `<thème>/state` :

```json
{"voltage": 238.8, "current": 2.58, "power": -17.2, "reactive_power": -631.0, "apparent_power": 631.2,
 "power_factor": 0.027, "frequency": 50.03, "energy_total": 18107.19, "energy_import": 26509.33, "energy_export": 8402.14}
```

| Clé | Mesure | Unité |
|---|---|---|
| `voltage` | tension | V |
| `current` | courant | A |
| `power` | puissance active (**positive = soutirage, négative = injection**) | W |
| `reactive_power` / `apparent_power` | puissance réactive / apparente | var / VA |
| `power_factor` | facteur de puissance | |
| `frequency` | fréquence | Hz |
| `energy_import` / `energy_export` | énergie importée / exportée | kWh |
| `energy_total` | importée − exportée | kWh |

Les capteurs de Home Assistant expirent (`expire_after`, six fois l'intervalle) : si le script s'arrête, ils passent en
« indisponible » au lieu de garder une valeur périmée. Les énergies s'utilisent directement dans le tableau Énergie.

## Registres lus

Flottants 32 bits (2 registres), octets de poids fort d'abord, fonction Modbus 3.

| Registre | Mesure |
|---|---|
| `0x2000` | tension (V) |
| `0x2002` | courant (A) |
| `0x2006` | puissance active (kW) |
| `0x200C` | puissance réactive (kvar) |
| `0x2012` | puissance apparente (kVA) |
| `0x2018` | facteur de puissance |
| `0x2020` | fréquence (Hz) |
| `0x4000` | énergie totale (kWh) |
| `0x400A` | énergie importée (kWh) |
| `0x4014` | énergie exportée (kWh) |

## Sonde de diagnostic

Si le compteur ne répond pas, `outils/modbus-probe.py` cherche l'adresse et la parité qui répondent (lecture seule) :

```bash
python3 outils/modbus-probe.py /dev/ttyUSB0 9600 11 1
```

## Statut

Version **0.9** : utilisé en continu sur un Raspberry Pi 3 (Raspbian 32 bits, Python 3.7) avec un DDSU666-H Huawei, adresse 11,
9600 8N1, publication toutes les 5 s vers un broker Mosquitto et Home Assistant. Les mesures ont été comparées à celles d'un
second compteur sur la même ligne (tension et puissance concordantes ; courant environ 4 % d'écart entre les deux compteurs).

Ce dépôt est aussi le fournisseur de l'agent DDSU666-H du projet [dimotic-ha](https://github.com/zdid/dimotic-ha).

## Licence

MIT, voir [LICENSE](LICENSE).
