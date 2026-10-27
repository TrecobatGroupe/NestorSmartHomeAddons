Resume architecture

Ce depot contient un add-on Home Assistant minimal qui fait deux choses:

- il se connecte a Azure Event Grid MQTT avec JWT (`CUSTOM-JWT`) ou certificat client (mTLS), s'abonne a des topics Azure et republie les messages recus sur Mosquitto local;
- il s'abonne aux topics d'etat locaux de Home Assistant et republie ces etats vers Azure Event Grid MQTT.

Il expose aussi une page ingress Home Assistant pour visualiser les messages recus et envoyes sur les deux brokers.

Le projet est volontairement plus simple que le bridge precedent: pas de filtrage JSON, pas de dashboard, pas d'inventaire Supervisor, seulement un worker de relais bidirectionnel.

Page ingress

- L'UI est servie par l'add-on sur l'ingress Home Assistant.
- La page affiche les quatre flux: Azure recu, Azure envoye, local recu et local envoye.
- Les traces sont conservees en memoire sur les 250 derniers messages.

Arborescence

```text
src/Nestor.HaAddon/
|-- .dockerignore
|-- README.md
|-- CHANGELOG.md
|-- DOCS.md
|-- Dockerfile
|-- config.yaml
|-- run.sh
|-- translations/
|   `-- en.yaml
|-- Program.cs
|-- Nestor.HaAddon.csproj
|-- appsettings.json
|-- Models/
|   |-- BridgeMessage.cs
|   `-- StatePublishEnvelope.cs
|-- Options/
|   `-- AppOptions.cs
`-- Services/
    |-- AzureEventGridJwtClient.cs
    |-- LocalMqttClient.cs
    |-- LocalMqttConnectionSettingsProvider.cs
    |-- MqttConnectionSettings.cs
    |-- RelayWorker.cs
    |-- ReliableMqttClient.cs
    `-- SupervisorApiClient.cs
```

Configuration utilisateur

```json
{
  "local_mqtt": {
    "host": "core-mosquitto",
    "port": 1883,
    "username": "",
    "password": "",
    "client_id": "nestor-ha-addon-local",
    "state_topics": [
      "homeassistant/+/+/state",
      "zigbee2mqtt/+"
    ],
    "qos": 1,
    "use_tls": false,
    "ca_cert_path": ""
  },
  "azure_event_grid": {
    "enabled": true,
    "host": "trecobat-event-grid-mqtt.westeurope-1.ts.eventgrid.azure.net",
    "port": 8883,
    "client_id": "home-assistant-relay",
    "username": "",
    "jwt": "",
    "jwt_file_path": "",
    "client_cert_path": "",
    "client_key_path": "",
    "subscribe_topics": [
      "devices/homeassistant/commands/#"
    ],
    "publish_topic": "devices/homeassistant/states",
    "qos": 1,
    "use_tls": true,
    "ca_cert_path": "",
    "clean_session": false
  },
  "bridge": {
    "local_publish_prefix": "eventgrid/in",
    "preserve_retain_flag": true,
    "source_name": "home-assistant"
  },
  "behavior": {
    "dry_run": false,
    "max_retry_attempts": 5,
    "base_retry_delay_seconds": 2,
    "operation_timeout_seconds": 15
  }
}
```

Important pour Event Grid

- En mode JWT, Azure Event Grid utilise MQTT v5 avec `Authentication Method = CUSTOM-JWT`.
- Le JWT est envoye dans `Authentication Data` du paquet CONNECT.
- Le claim `aud` du JWT doit cibler le namespace Event Grid, typiquement `nomnamespace.region-1.ts.eventgrid.azure.net/`.
- En mode certificat client, renseigner `client_cert_path` et `client_key_path` (fichiers montes sous `/ssl`).

Comportement du relais

- Les messages recus depuis Azure sont republies localement sous `bridge.local_publish_prefix + '/' + topic_azure`.
- Les messages recus sur `local_mqtt.state_topics` sont encapsules en JSON puis publies sur `azure_event_grid.publish_topic`.
- Les topics republies localement sous le prefixe d'entree Azure sont ignores pour l'aller-retour vers Azure afin d'eviter une boucle triviale.

Build Docker

Depuis `src/Nestor.HaAddon` :

```powershell
$RegistryName = "myregistry"
$Repository = "nestor-ha-addon"
$Version = "0.1.0"
$Image = "$RegistryName.azurecr.io/$Repository:$Version"

az acr login --name $RegistryName
docker build -f Dockerfile -t $Image .
docker push $Image
```

Tests manuels conseilles

1. Fournir un JWT valide et verifier que l'add-on se connecte a Event Grid.
2. Publier un message sur un topic Azure abonne et verifier son arrivee sur Mosquitto local sous le prefixe configure.
3. Verifier que les topics d'etat locaux retained sont bien retransmis vers `azure_event_grid.publish_topic` au demarrage.
