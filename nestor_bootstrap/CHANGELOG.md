# Changelog

## 0.3.2

- image publiée dans le registry `distribution` de l'environnement (Container App) au lieu
  d'ACR ; `image:` réécrite à la publication. Aucun changement fonctionnel.

## 0.3.1

- Nouvelle option `tpm_tcti` : TCTI du TPM (`TPM2OPENSSL_TCTI`). Vide : `device:/dev/tpmrm0` de
  l'image, inchangé sur une Box. Dev (VM HAOS d'Aspire, sans pilote TPM) :
  `swtpm:host=10.0.2.2,port=2321`. Remplace l'`environment` injecté par `addon-repo`.

## 0.3.0

> **Ordre de déploiement obligatoire** (voir `docs/runbooks/deploiement-correctifs-tpm.md`) : serveur
> (infra, EST, Function SignalR, Admin API) d'abord ; puis HaAddon **0.10.0** confirmé sur toute la
> flotte TPM ; **seulement ensuite** Bootstrap 0.3.0. HaAddon 0.9.x ne lit pas le blob TSS2 que cette
> version écrit dans `/ssl/private.key` : il perdrait MQTT et le negotiate SignalR (et possiblement le
> canal de mise à jour). Un EST antérieur refuse les CSR EC. Chaque Box TPM doit aussi avoir la SRK
> `0x81000002` (add-on TPM factory 0.2.0). Pas de retour arrière une fois le blob TSS2 écrit.

- Box TPM : la clé mTLS est une clé EC P-256 générée dans le TPM, sous la SRK `0x81000002`
  (nouvelle option `mtls_key_parent`). `/ssl/private.key` contient un blob `TSS2 PRIVATE KEY`
  chiffré par la SRK : copié hors de la Box, il est inutilisable.
- Box sans TPM : clé RSA 2048 logicielle, inchangée.
- Image : ajout du CLI `openssl` et de la TCTI `device`.
- Toutes les opérations TPM d'un tick sont sérialisées par un verrou unique.
- SRK `0x81000002` absente (Box provisionnée par l'usine d'avant 0.2.0) : message `Critical` explicite
  à chaque tick demandant de re-provisionner la Box avec l'add-on TPM factory 0.2.0.

## 0.2.1

- Image : ajout de la TCTI `swtpm` (`libtss2-tcti-swtpm0t64`) pour joindre le TPM simulé
  en TCP dans l'environnement de dev (`HAOS_TPM=net`). Sans effet en production.

## 0.2.0

- Enrôlement, renouvellement et reprise par l'identité TPM de la Box (ADR-019) :
  plus de token ni de `home_id` à saisir. Le `homeId` est fourni par le serveur.
- `home_id` devient facultatif ; il reste requis sans identité TPM (chemin token).
- Nouvelles options `identity_provider` et `identity_key_uri` ; accès à `/dev/tpmrm0`.
- Box sans TPM : vider `identity_provider` et `identity_key_uri` pour utiliser le token d'enrôlement.
- Mise à jour depuis 0.1.0 : une Box sans TPM ou absente de l'inventaire continue sur le chemin
  token/mTLS tant que `home_id` est renseigné.
- Après un échec (Box pas encore associée, réseau absent), nouvel essai rapide : 1, 2, 4, 8 puis
  15 minutes au plus, au lieu d'attendre `check_interval_hours`.

## 0.1.0
- première version de l'add-on NESTOR Bootstrap : provisioning initial et
  renouvellement du certificat mTLS de la Box (enrôlement EST).
