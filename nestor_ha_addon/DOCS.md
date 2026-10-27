# Nestor HA Addon

Add-on Home Assistant qui :

- Connecte Home Assistant (WebSocket) à **Azure SignalR Serverless**.
- Transmet les state changes des entités filtrées vers la React app.
- Exécute les commandes reçues depuis la React app (ex : allumer une lumière).
- Expose une page ingress de monitoring.
- Optionnel : connexion directe MQTT vers Azure Event Grid (mTLS).

## Paramètres

L'identifiant du logement (HomeId) n'est pas configurable : il est lu dans le
CN `home-{homeId}` de `/ssl/client.crt`, émis par Nestor Bootstrap (ADR-019).
Sans certificat, l'add-on attend l'enrôlement. L'authentification auprès de
Home Assistant utilise le `SUPERVISOR_TOKEN` injecté par le Supervisor.

Les options facultatives n'apparaissent dans l'onglet Configuration qu'avec
« Afficher les options facultatives inutilisées » ; vides, elles prennent leur défaut.

### `tpm_tcti` (facultatif)

TCTI du TPM qui porte la clé mTLS. Vide : `/dev/tpmrm0` (Box). Dev (VM HAOS d'Aspire) : `swtpm:host=10.0.2.2,port=2321`.

### `signalr`

| Clé | Description |
|-----|-------------|
| `negotiate_url` | URL de l'endpoint `negotiate` Azure Functions (ex : `https://<functions>.azurewebsites.net/api/negotiate`) |

### `ha`

| Clé | Description |
|-----|-------------|
| `entity_filter` | Liste de préfixes de domaines à observer (ex : `light.`, `switch.`, `sensor.`, `binary_sensor.`) |
| `temperature_history_hours` | Facultatif. Heures d'historique de température chargées au démarrage (1–48, défaut : 12) |

### `registry`

Registre privé des add-ons Nestor. Pour mettre à jour un add-on Nestor
(`*_nestor_ha_addon`, `*_nestor_bootstrap`), l'add-on obtient lui-même un
credential de pull éphémère auprès de registry-auth, en mTLS avec le
certificat de la Box (même chemin, TPM compris, que le `negotiate` SignalR). Il le
pose dans le Supervisor le temps de la mise à jour puis le retire. Aucun
credential n'est reçu dans l'ordre `ExecuteUpdate`. core, supervisor, os et les
add-ons tiers n'en obtiennent jamais.

| Clé | Description |
|-----|-------------|
| `credential_url` | URL HTTPS de `POST /credential` de registry-auth (ex : `https://<registry-auth>/credential`). Vide : les mises à jour d'add-on Nestor échouent (« Identifiants de registre indisponibles »). |

`credential_url` est fixée à l'installation : le Supervisor conserve les options
de l'add-on et ne les réécrit pas lors d'une mise à jour. Un changement de FQDN de
registry-auth exige donc de corriger l'option sur chaque Box ou de réinstaller
l'add-on. Conserver un FQDN stable par environnement (domaine personnalisé plutôt
que le FQDN généré par Container Apps, qui change si l'environnement est recréé).

### `event_grid_mqtt`

Connexion directe MQTT vers Azure Event Grid via certificat client (mTLS).

| Clé | Description |
|-----|-------------|
| `enabled` | Activer la connexion Event Grid MQTT |
| `hostname` | Hostname du namespace Event Grid MQTT |
| `port` | Facultatif. Port MQTT (défaut : 8883) |
| `client_id` | Facultatif. Identifiant client MQTT ; vide, le HomeId. L'ancien défaut `addon-nestor` est également remplacé par le HomeId. |
| `cert_path` | Facultatif. Certificat client (défaut : `/ssl/client.crt`, déposé par Nestor Bootstrap) |
| `key_path` | Facultatif. Clé privée client (défaut : `/ssl/private.key`, déposée par Nestor Bootstrap) |
| `use_tls` | Facultatif. TLS + certificat client (défaut : `true`). `false` seulement pour un broker local en clair. |
| `temperature_report_interval_minutes` | Facultatif. Intervalle de remontée de température vers Azure (1–1440 min, défaut : 15) |

Le certificat serveur du broker est accepté s'il est valide pour le magasin
système (Event Grid) **ou** s'il est signé par la CA Nestor du fichier `ca.pem`
situé dans le dossier de `cert_path` (déposé par Nestor Bootstrap). Dans ce
second cas, le certificat doit porter l'usage serverAuth. Aucune option à
régler : la même configuration vaut contre Event Grid et contre le Mosquitto
mTLS de l'AppHost Aspire.

## Ingress

La page ingress est disponible sur le port `8099` et permet de visualiser les messages échangés avec SignalR et Event Grid.