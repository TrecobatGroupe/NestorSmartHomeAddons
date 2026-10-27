# Changelog

## 0.12.2

- Correctif : la mise à jour à distance d'un add-on Nestor échouait toujours en « Registre
  inattendu ». L'hôte du registre était lu dans l'image renvoyée par `/addons/self/info`, que le
  Supervisor n'expose pas. Il vient maintenant de la nouvelle option `registry_server`, réécrite
  à la publication comme `image:` et `registry.credential_url`. Option vide ou non réécrite :
  échec « Registre des add-ons non configuré », sans demande de credential.

## 0.12.1

- image publiée dans le registry `distribution` de l'environnement (Container App) au lieu
  d'ACR ; `image:` et `registry.credential_url` réécrites à la publication. Aucun changement
  fonctionnel.

## 0.12.0

- credential de registre obtenu par l'add-on lui-même : pour une mise à jour d'add-on Nestor
  (`*_nestor_ha_addon`, `*_nestor_bootstrap`), `POST {registry.credential_url}` en mTLS avec le
  certificat de la Box (clé TPM comprise), puis pose dans le Supervisor le temps de la mise à
  jour. L'ordre `ExecuteUpdate` ne porte plus de `registry` (champ ignoré s'il est reçu).
- nouvelle option `registry.credential_url` (URL HTTPS de registry-auth) ; vide, les mises à
  jour d'add-on Nestor échouent. core, supervisor, os et add-ons tiers : jamais de credential.

## 0.11.0

- tunnel SSH inverse supprimé (option `ssh_tunnel`, paquets `openssh-client` et `sshpass`).
- option `signalr.home_id` supprimée : le HomeId vient uniquement du CN de `/ssl/client.crt`.
  Sans certificat, l'add-on attend l'enrôlement par Nestor Bootstrap.
- option `ha.long_lived_token` supprimée : authentification HA par `SUPERVISOR_TOKEN` seulement.
- onglet Configuration allégé : seuls `negotiate_url`, `entity_filter`, `enabled` et `hostname`
  sont affichés ; `port`, `client_id`, `cert_path`, `key_path`, `use_tls`, `tpm_tcti` et les
  intervalles deviennent facultatifs avec leurs défauts. Traductions orphelines retirées.

## 0.10.4

- certificat client (MQTT et negotiate) : l'Intermediate de `/ssl/ca.pem` est envoyé avec le leaf.
  `client.crt` de nestor_bootstrap ne contient que le leaf, et la SignalR API ne connaît que la
  Root : sans l'Intermediate, le negotiate device était refusé (401).
- dev (TCTI `swtpm:`) : au démarrage, vidage des objets transitoires laissés dans swtpm par
  l'exécution précédente (sinon `TPM_RC_OBJECT_MEMORY` après un redémarrage de l'add-on).
  Sans effet sur une Box (`/dev/tpmrm0`).

## 0.10.3
 - Fix dockerfile build


## 0.10.2

- negotiate SignalR : plus de `?homeId=` dans l'URL, l'identité vient uniquement du certificat
  de la Box (la SignalR API en Azure ignorait déjà ce paramètre).
- certificat serveur du negotiate accepté via `/ssl/ca.pem` (CA Nestor de dev), comme pour MQTT.
  Dev local : `negotiate_url` devient `https://signalr-ingress.dns.podman:7443/api/negotiate`.

## 0.10.1

- nouvelle option `tpm_tcti` : TCTI du TPM (`TPM2OPENSSL_TCTI`) qui porte la clé mTLS. Vide :
  `device:/dev/tpmrm0` de l'image, inchangé sur une Box. Dev (VM HAOS d'Aspire) :
  `swtpm:host=10.0.2.2,port=2321`.

## 0.10.0

> **À déployer et confirmer sur toute la flotte TPM avant nestor_bootstrap 0.3.0** (voir
> `docs/runbooks/deploiement-correctifs-tpm.md`) : 0.9.x ne lit pas le blob TSS2 écrit par
> nestor_bootstrap 0.3.0 et perdrait MQTT et le negotiate SignalR. Une fois ce blob présent sur la
> Box, pas de retour arrière sous 0.10.0.
- clé mTLS dans le TPM (ADR-019, nestor_bootstrap 0.3.0) : si `/ssl/private.key`
  est un blob TSS2, la connexion MQTT et le negotiate SignalR font signer le
  handshake TLS par le TPM (`/dev/tpmrm0`) ; la clé n'est jamais en clair ;
- une seule opération TPM à la fois dans l'add-on (verrou partagé par MQTT et
  negotiate) ; la clé chargée est réutilisée tant que le blob ne change pas ;
- une clé logicielle (PKCS#8, Box sans TPM) reste prise en charge.

## 0.9.1
- correction de l'identité MQTT après enrôlement TPM : Client ID égal au
  HomeId et `authenticationName` égal au CN `home-{homeId}` ;
- le nouveau défaut `client_id` vide utilise automatiquement le HomeId.
  L'ancienne valeur `addon-nestor` des installations existantes est également
  migrée à l'exécution ;
- alignement des autorisations Mosquitto locales et des clients Azure Event
  Grid sur ce contrat.

## 0.9.0
- `home_id` facultatif : lu dans le CN de `/ssl/client.crt` émis par nestor_bootstrap (ADR-019) ;
  l'add-on attend le certificat si aucun n'est présent.

## 0.8.0
- administration distante depuis l'admin Nestor de tous les add-ons installés :
  consultation et modification de leurs options JSON brutes via le Supervisor,
  avec contrôle de révision pour éviter d'écraser une modification concurrente.
- consultation et modification de `/config/configuration.yaml` grâce au montage
  `config:rw` ; l'écriture est atomique, conserve cinq sauvegardes dans
  `/data/configuration-backups`, lance le contrôle Home Assistant et restaure
  automatiquement la version précédente si la validation échoue.
- redémarrage distinct de Home Assistant Core et de l'hôte Home Assistant OS
  depuis l'admin, avec commandes corrélées et dédupliquées.
- sécurisation du canal d'administration SignalR : résultats acceptés uniquement
  depuis une requête upstream Azure SignalR signée et l'identité
  `device-{homeId}` correspondante. Cette version nécessite
  `signalRAuthMode=key` ; le mode `identity` échoue explicitement au démarrage.
- configurations et secrets jamais stockés dans le jumeau ni dans l'Admin API ;
  logs et audit limités aux métadonnées. Les contenus sont limités à 512 Kio
  UTF-8 et au budget réel du message SignalR.
- prérequis à valider sur une Box avant déploiement : le rôle Supervisor de
  l'add-on doit autoriser la lecture/écriture des options, `/core/check`,
  `/core/restart` et `/host/reboot`.

## 0.7.0
- mise à jour d'add-on depuis l'admin : `ExecuteUpdate` accepte un champ
  `registry` (`server`, `username`, `password`, `expiresAt`) et `batchId`.
  L'add-on enregistre temporairement ce registre dans le Supervisor
  (`/docker/registries`) avant le pull puis le retire ; un credential expiré
  ou un serveur différent de celui de l'image est refusé.
- `auto_update` est désactivé au démarrage sur les add-ons Nestor : seul
  l'admin déclenche une mise à jour.
- auto-mise à jour : l'ordre et la version de départ
  (`fromVersion`) sont persistés ; au redémarrage, le registre est retiré et
  le statut de l'ordre est réconcilié (réussi / non appliqué) puis republié
  dans le jumeau.
- toutes les mises à jour (core, supervisor, os, add-on) sont demandées à
  Home Assistant Core par WebSocket (`hassio/update/addon`,
  `hassio/update/core`, `supervisor/api` sur `/supervisor/update` et
  `/os/update`) et non plus à l'API REST du Supervisor, qui refuse qu'un
  add-on se mette à jour lui-même (403). L'auto-mise à jour de HaAddon depuis
  l'admin est donc possible ; son issue est réconciliée au redémarrage.
  L'utilisateur Home Assistant de l'add-on doit être administrateur (sinon
  l'ordre échoue avec un message explicite). Voir ADR 12.016.
- core, supervisor, os : leur mise à jour coupe la connexion WebSocket
  (redémarrage de Core, du Supervisor ou de la Box). La version installée est
  lue et persistée avant la commande ; sans réponse (connexion perdue ou délai
  de 45 min), l'ordre reste « running » puis est réconcilié d'après la version
  (réussi / non appliqué) au démarrage de l'add-on, après chaque reconnexion
  à Home Assistant, ou tout de suite sur délai dépassé, puis republié dans le
  jumeau.
- connexion WebSocket Home Assistant : « connecté » signifie authentifié avec
  la pompe de lecture en marche ; une pompe arrêtée (message illisible…)
  abandonne la socket pour forcer la reconnexion, et une requête émise sans
  connexion prête échoue aussitôt sans être envoyée.

## 0.6.1
- jumeau numérique : l'inventaire remonte désormais **toutes** les entités et
  tous les devices qui en portent. `entity_filter` ne s'applique plus qu'à la
  liste temps réel envoyée à l'app React (`RequestEntityList`) ; les devices
  climate, cover, fan, media_player, vacuum… apparaissent dans l'admin.

## 0.6.0
- jumeau numérique publié par SignalR (`ReportInventory`) et non plus par
  MQTT : accusé `InventoryAck` attendu 30 s, republication sinon.
- negotiate SignalR authentifié par le certificat client de la Box
  (`GET {negotiate_url}/device`, mTLS) : l'identité `device-{homeId}` n'est
  plus déclarée par l'add-on.
- télémétrie MQTT : le CloudEvent porte `subject` = topic MQTT.

## 0.5.9
- mises à jour pilotées depuis l'admin : message SignalR `ExecuteUpdate`
  (émis par l'Admin API uniquement) → API Supervisor `core/update`,
  `supervisor/update`, `os/update` ou `store/addons/{slug}/update`. Un seul
  ordre à la fois ; l'issue (`runtime.lastUpdateCommand` : running, succeeded,
  failed, interrupted) est persistée dans `/data` et le jumeau est republié au
  début et à la fin (`trigger` `update-started` / `update-completed`).
- inventaire : `platform.components` remonte version installée, dernière
  version et `updateAvailable` pour Core, Supervisor et OS.
- le jumeau est republié (`trigger` `update-available`) quand une entité HA
  `update.*` change, avec 30 s d'anti-rebond : une mise à jour découverte par
  le Supervisor après le démarrage de l'add-on apparaît désormais dans l'admin
  sans attendre un redémarrage.
- correction : `runtime.addonVersion` valait toujours `1.0.0` (version
  d'assembly par défaut). Le Dockerfile injecte désormais `BUILD_VERSION`
  (version de `config.yaml`, fournie par le Supervisor au build).

## 0.5.8
- valeurs par défaut `cert_path` / `key_path` alignées sur Nestor Bootstrap :
  `/ssl/client.crt` et `/ssl/private.key` (au lieu de `/ssl/addon-nestor.crt` /
  `.key`, hérités du script `generate-client-cert.sh`). **Une installation
  existante garde ses anciennes valeurs** : les remplacer dans l'onglet
  Configuration.
- certificat serveur du broker accepté via le magasin système (Event Grid)
  **ou** via le bundle CA Nestor `ca.pem` du dossier de `cert_path` (déposé par
  Nestor Bootstrap), avec usage serverAuth exigé. Permet de viser le Mosquitto
  mTLS de l'AppHost Aspire sans aucune option supplémentaire.
- correction : le certificat client mTLS était libéré avant le handshake
  (régression de 0.5.6, `use_tls`). SslStream n'envoyait alors aucun
  certificat et le broker refusait la connexion (« peer did not return a
  certificate »), Event Grid compris.
- un message refusé par le broker (PUBACK en échec, ex. `NotAuthorized` pour un
  topic hors du CN) est désormais journalisé en warning au lieu d'être compté
  comme publié.

## 0.5.7
- publication automatique au démarrage d'un inventaire Digital Twin (`reported`,
  schema `2.0`) sur `nestor/{homeId}/state/inventory`, en MQTT v5 QoS 1.
- inventaire des devices et entités Home Assistant, add-ons Supervisor,
  métadonnées de plateforme et compteurs dérivés.
- réutilisation de la configuration `event_grid_mqtt` existante ; activer
  `event_grid_mqtt.enabled` et renseigner le broker/certificat mTLS pour
  publier l'inventaire.

## 0.5.6
- ajout de l'option `event_grid_mqtt.use_tls` (defaut `true`, comportement inchange) pour pouvoir viser un broker MQTT local sans TLS en dev (Mosquitto via HAOS/Aspire).

## 1.0.2
- correction de la mise a jour des compteurs du monitoring ingress pour eviter les valeurs figees alors que les tableaux continuent a se rafraichir.
- ajout d'une pagination JavaScript sur les deux tableaux de la page de monitoring pour ameliorer la lisibilite des traces.
- formatage JSON de la colonne payload du premier tableau pour un affichage multi-lignes plus lisible.

## 1.0.1
- suppression automatique du prefixe statique des topics de souscription Azure (ex: `devices/homeassistant/commands/`) avant republication sur le broker local.
- support du pilotage direct du topic final depuis Azure Event Grid (ex: `devices/homeassistant/commands/zigbee2mqtt/prise/set` -> local `zigbee2mqtt/prise/set`).
- `bridge.local_publish_prefix` conserve un fonctionnement optionnel et reste vide par defaut pour une republication locale sans prefixe ajoute.

## 1.0.0
- optional prefix for local MQTT topics to avoid conflicts with other Mosquitto clients.

## 0.6.4

- ajout d'un mode d'authentification Azure Event Grid par certificat client (mTLS) en alternative au JWT.
- validation de configuration assouplie: l'add-on accepte desormais soit `jwt`/`jwt_file_path`, soit `client_cert_path` + `client_key_path`.
- ajout des champs `client_cert_path` et `client_key_path` dans `config.yaml` et la documentation.

## 0.6.3

- fiabilisation du diagnostic de connexion MQTT Event Grid: `NotAuthorized` est maintenant loggue comme probleme d'authentification JWT/autorisation au lieu d'un faux positif TLS.
- ajout d'un pre-diagnostic JWT au demarrage (claims `iss`, `sub`, `aud`, `exp`, `nbf`) avec avertissements explicites en cas d'audience incorrecte, token expire/non valide, ou `sub` non aligne avec `client_id`.

## 0.6.2

- fix arm64 build : correction du Dockerfile pour utiliser `--platform=${TARGETPLATFORM}` et `--self-contained true` afin d'eviter les erreurs `exec format error` sur architectures ARM64.

## 0.5.0

- refacto dockerfile  

## 0.4.0

- Support multi-architecture: build pour linux/amd64 et linux/arm64 avec les RID (.NET Runtime Identifiers) appropriés.
- Dockerfile corrigé pour utiliser `--platform=${TARGETPLATFORM}` et `--self-contained true` afin d'eviter les erreurs `exec format error` sur architectures ARM64.

## 0.3.0

- Suppression du script `run.sh` : le binaire .NET est desormais lance directement via `ENTRYPOINT` dans le Dockerfile, eliminant les erreurs `exec format error` liees aux fins de ligne CRLF sous Windows.

## 0.2.0

- Page ingress Home Assistant pour visualiser les messages recus et envoyes sur les deux brokers (Azure Event Grid et Mosquitto local).
- API REST `GET /api/messages/snapshot` exposant les 250 derniers messages traces avec horodatage, broker, sens, topic, payload, QoS, retain et statut.
- Correction du demarrage du conteneur Linux : normalisation des fins de ligne CRLF dans `run.sh` et invocation via `/bin/sh` pour eviter les erreurs `exec format error` lors d'un build depuis Windows.

## 0.1.0

- Premiere version du relais minimal Azure Event Grid MQTT JWT vers Mosquitto local.


 az acr build --registry acrnestorpoc --image ha-eventgrid-bridge:1.0.1-arm64 --platform linux `  --build-arg "DOTNET_RID=linux-arm64" `  --build-arg "BUILD_FROM=mcr.microsoft.com/dotnet/aspnet:10.0-noble-arm64v8" `  --file addon-jwt-relay/Dockerfile .


  az acr build --registry trecobatpoc --image nestor-signalr-poc-addon:0.3.0 --platform linux/amd64 --file addon-jwt-relay/Dockerfile .