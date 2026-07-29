# Eufy Smart Lock C30 (`T85D0`) for Home Assistant

[English](#english) · [Français](#français) ·
[Full English guide](docs/GUIDE_EN.md) ·
[Guide français complet](docs/GUIDE_FR.md)

## English

Experimental Home Assistant add-on that adds working lock/unlock control for
the Eufy Smart Lock C30 through Eufy's Security MQTT channel.

The C30 is visible through the legacy Eufy integration, but P2P control fails
with DSK error `20028`. This add-on builds the implementation from
[`bropat/eufy-security-client` PR 797](https://github.com/bropat/eufy-security-client/pull/797),
which tunnels the lock's encrypted BLE frames through
`security-mqtt-{region}.anker.com`.

## Status

Validated on real C30 hardware:

- lock command acknowledged and physically confirmed;
- unlock command acknowledged and physically confirmed;
- actual state synchronized through MQTT heartbeat;
- Home Assistant entity updated in both directions;
- add-on reconnect tested after restart.

## Important

This is an unofficial, experimental integration for a physical access-control
device. Test with someone beside the lock. Never trust an optimistic UI state
without a device response, heartbeat, and physical validation.

Use a dedicated secondary Eufy account shared from the primary account. Never
publish account credentials, authentication tokens, certificates, private
keys, device serial numbers, IP addresses, SSIDs, or unredacted logs.

## Install

Copy `eufy_security_ws_c30` into:

```text
/addons/local/eufy_security_ws_c30
```

Reload the add-on store, install the local add-on, configure the secondary
account, and start it.

Example options:

```yaml
username: account-ha@example.com
password: USE_A_STRONG_SECRET
country: CA
port: 3000
polling_interval: 10
accept_invitations: true
debug: false
ipv4first: true
event_duration: 10
stations:
  - serial_number: YOUR_C30_SERIAL
    ip_address: YOUR_C30_IP
```

Expected successful log lines:

```text
SecurityMQTT connecting to security-mqtt-us.anker.com:8883
SecurityMQTT connected successfully
SecurityMQTT subscribed: cmd/eufy_security/T85D0/SERIAL/res
SecurityMQTT subscribed: cmd/eufy_security/T85D0/SERIAL/req
```

Legacy P2P error `20028` may still appear during initialization. It does not
block the MQTT path when the SecurityMQTT connection and exact subscriptions
succeed.

## Documentation

The complete English guide is in [docs/GUIDE_EN.md](docs/GUIDE_EN.md).

## Upstream and attribution

- [C30 Security MQTT implementation — PR 797](https://github.com/bropat/eufy-security-client/pull/797)
- [Eufy Mega v6 transport — PR 939](https://github.com/bropat/eufy-security-client/pull/939)
- [eufy-security-ws](https://github.com/bropat/eufy-security-ws)
- [Home Assistant eufy_security integration](https://github.com/fuatakgun/eufy_security)

The add-on builds the MIT-licensed upstream client from a pinned commit.

## Disclaimer

Not affiliated with Eufy or Anker. Cloud endpoints and protocols may change
without notice. Keep a rollback path and verify every update on accessible
hardware.

---

## Français

Module complémentaire Home Assistant expérimental ajoutant le verrouillage et
le déverrouillage fonctionnels de la Eufy Smart Lock C30 au moyen du canal
Security MQTT d'Eufy.

La C30 apparaît dans l'intégration Eufy historique, mais son contrôle P2P
échoue avec l'erreur DSK `20028`. Ce module construit l'implémentation de la
[PR 797 de `bropat/eufy-security-client`](https://github.com/bropat/eufy-security-client/pull/797),
qui transporte les trames BLE chiffrées de la serrure par
`security-mqtt-{region}.anker.com`.

### État de la validation

Validation effectuée sur une vraie C30 :

- commande de verrouillage acquittée et confirmée physiquement;
- commande de déverrouillage acquittée et confirmée physiquement;
- état réel synchronisé par heartbeat MQTT;
- entité Home Assistant mise à jour dans les deux directions;
- reconnexion du module testée après redémarrage.

### Avertissement

Cette intégration non officielle contrôle un accès physique. Faites les essais
avec une personne près de la serrure. Ne faites jamais confiance à un état
optimiste de l'interface sans réponse de l'appareil, heartbeat et validation
physique.

Utilisez un compte Eufy secondaire consacré à Home Assistant. Ne publiez
jamais les identifiants, jetons, certificats, clés privées, numéros de série,
adresses IP, SSID ou journaux non expurgés.

### Installation

Copiez `eufy_security_ws_c30` dans :

```text
/addons/local/eufy_security_ws_c30
```

Rechargez ensuite la boutique des modules complémentaires, installez le
module local, configurez le compte secondaire et démarrez-le.

Exemple d'options :

```yaml
username: compte-ha@example.com
password: UTILISER_UN_SECRET_FORT
country: CA
port: 3000
polling_interval: 10
accept_invitations: true
debug: false
ipv4first: true
event_duration: 10
stations:
  - serial_number: NUMERO_DE_SERIE_C30
    ip_address: ADRESSE_IP_C30
```

Le guide français complet se trouve dans
[docs/GUIDE_FR.md](docs/GUIDE_FR.md).

### Avis

Ce projet n'est affilié ni à Eufy ni à Anker. Les services Cloud et les
protocoles peuvent changer sans préavis. Conservez une procédure de retour
arrière et validez chaque mise à jour sur une serrure physiquement accessible.
