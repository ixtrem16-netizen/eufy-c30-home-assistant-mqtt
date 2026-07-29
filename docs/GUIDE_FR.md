# Eufy Smart Lock C30 dans Home Assistant avec Security MQTT

Guide communautaire fondé sur une intégration réelle validée le 29 juillet
2026 avec une Eufy Smart Lock C30, modèle `T85D0`.

## Résultat obtenu

Le verrouillage et le déverrouillage depuis Home Assistant ont été validés de
bout en bout :

1. appel du service `lock.lock` ou `lock.unlock` dans Home Assistant;
2. publication de la commande sur le courtier Security MQTT d'Eufy;
3. réception d'un accusé de commande provenant de la serrure;
4. réception d'un heartbeat MQTT contenant l'état réel;
5. mise à jour de l'entité Home Assistant;
6. confirmation physique du mouvement de la serrure.

L'état final n'a jamais été considéré comme valide sur la seule base de
l'affichage optimiste de Home Assistant.

## Pourquoi l'intégration standard échoue

La C30 Wi-Fi `T85D0` est généralement découverte par
`eufy-security-client`, mais son contrôle P2P échoue :

```text
get_dsk_keys: code 20028
```

Cette serrure ne reçoit pas les clés DSK nécessaires au chemin P2P classique.
Ajouter son type d'appareil aux conditions des autres serrures peut faire
apparaître une entité `lock`, mais ne suffit pas à actionner le pêne.

Une commande Cloud comme `upload_devs_params` peut également retourner un
succès sans mouvement physique. Ce retour ne constitue donc pas une preuve de
verrouillage.

Le vrai canal de contrôle de la C30 est :

```text
Home Assistant
  -> eufy-security-ws
  -> eufy-security-client
  -> trame BLE FF09 chiffrée
  -> Security MQTT
  -> Wi-Fi de la serrure
  -> mécanisme physique
```

Le courtier utilisé pour le contrôle est :

```text
security-mqtt-us.anker.com:8883
```

Pour une installation européenne, la variante attendue est :

```text
security-mqtt-eu.anker.com:8883
```

Il ne faut pas confondre ce courtier avec
`aiot-mqtt-us.anker.com`. Les certificats mTLS sont obtenus à travers le
service AIOT, mais les sujets de commande de la C30 se trouvent sur le
courtier **Security MQTT**.

## Travaux amont utilisés

Cette solution s'appuie sur les travaux communautaires suivants :

- `bropat/eufy-security-client`, PR 797, par `nick-pape` :
  contrôle Security MQTT des serrures C30 `T85D0`;
- `bropat/eufy-security-client`, PR 939, par `lenoxys` :
  transport et authentification Eufy Mega v6;
- `bropat/eufy-security-ws`;
- intégration Home Assistant `fuatakgun/eufy_security`.

La PR 797 contient notamment :

- `SecurityMQTTService`;
- le codec des trames BLE de serrure;
- la création des messages MQTT;
- le décodage des heartbeats;
- le routage spécifique des C30 vers MQTT au lieu de P2P;
- la synchronisation de l'état et de la batterie.

Références :

- <https://github.com/bropat/eufy-security-client/pull/797>
- <https://github.com/bropat/eufy-security-client/pull/939>
- <https://github.com/bropat/eufy-security-client>
- <https://github.com/bropat/eufy-security-ws>
- <https://github.com/fuatakgun/eufy_security>

## Prérequis

- Home Assistant OS ou Supervised;
- intégration `eufy_security` dans Home Assistant;
- add-on `eufy-security-ws`;
- compte Eufy secondaire consacré à l'intégration;
- serrure partagée vers ce compte avec les droits suffisants;
- pays du compte identique à celui configuré dans l'add-on;
- Node.js compatible avec la branche utilisée;
- sauvegarde Home Assistant avant modification.

Il est préférable de ne pas utiliser le compte Eufy principal dans une
intégration non officielle.

## Création du compte Eufy secondaire

1. Créer une adresse de service réservée à Home Assistant.
2. Créer un compte Eufy avec cette adresse.
3. Depuis le compte principal, partager la serrure avec ce compte.
4. Donner les autorisations nécessaires au contrôle.
5. Accepter l'invitation.
6. Vérifier que la C30 apparaît dans l'application du compte secondaire.

Dans l'add-on :

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

Ne jamais publier le mot de passe, les jetons, les certificats, la clé privée
MQTT, le numéro de série réel ou l'adresse IP réelle.

## Construction de la bibliothèque C30-MQTT

Récupérer la branche de la PR 797 :

```sh
git clone https://github.com/bropat/eufy-security-client.git
cd eufy-security-client
git fetch origin refs/pull/797/head:pr-797
git switch pr-797
npm install --ignore-scripts
npm run build
npm pack
```

La commande `npm pack` produit une archive semblable à :

```text
eufy-security-client-3.8.0.tgz
```

Conserver le SHA du commit réellement construit afin que le déploiement soit
reproductible.

## Add-on Home Assistant local

Créer une copie locale de l'add-on `eufy-security-ws`, par exemple :

```text
/addons/local/eufy_security_ws_c30/
```

Placer l'archive construite dans ce dossier et utiliser un `Dockerfile` de ce
type :

```dockerfile
ARG BUILD_FROM
FROM $BUILD_FROM

ARG EUFY_SECURITY_WS_VERSION

WORKDIR /usr/src/app
COPY eufy-security-client-3.8.0-c30-mqtt.tgz /tmp/eufy-security-client-3.8.0-c30-mqtt.tgz

RUN \
    set -x \
    && apk add --no-cache \
        jq \
        nodejs \
        npm \
    && node --version \
    && npm install --force \
        "eufy-security-ws@${EUFY_SECURITY_WS_VERSION}" \
    && npm install --force \
        /tmp/eufy-security-client-3.8.0-c30-mqtt.tgz \
    && rm -rf \
        node_modules/eufy-security-ws/node_modules/eufy-security-client \
    && rm -f \
        /tmp/eufy-security-client-3.8.0-c30-mqtt.tgz

COPY run.sh /
RUN chmod a+x /run.sh

WORKDIR /data
CMD ["/run.sh"]
```

### Pourquoi supprimer la dépendance imbriquée

`eufy-security-ws` peut demander une version plus récente de
`eufy-security-client`. NPM conserve alors :

- la version C30-MQTT à la racine;
- une autre version imbriquée sous `eufy-security-ws/node_modules`.

Le serveur WebSocket charge la version imbriquée et ignore le correctif.
Supprimer cette copie force la résolution vers la bibliothèque C30-MQTT
installée à la racine.

Sans cette étape, les journaux montrent encore le chemin Mega/P2P habituel et
aucune ligne `SecurityMQTT` lors d'une commande.

## Exemple de `config.yaml`

```yaml
name: eufy-security-ws C30 MQTT
description: eufy-security-ws avec contrôle Security MQTT du C30
version: 3.1.0-c30.1
slug: eufy_security_ws_c30
url: https://github.com/bropat/hassio-eufy-security-ws
init: false
arch:
  - aarch64
  - amd64
stage: experimental
host_network: true
timeout: 30
startup: services
boot: auto
```

Reprendre le schéma d'options de l'add-on officiel pour `username`,
`password`, `country`, `port`, `polling_interval`, `debug`, `ipv4first`,
`event_duration` et `stations`.

## Exemple de `build.yaml`

La branche utilisée lors de cette validation nécessitait Node.js 24 :

```yaml
build_from:
  aarch64: ghcr.io/home-assistant/aarch64-base:3.23
  amd64: ghcr.io/home-assistant/amd64-base:3.23
args:
  EUFY_SECURITY_WS_VERSION: 3.1.0
```

Toujours vérifier la contrainte `engines.node` du commit construit. Ces
versions évolueront.

## Installation et mise à jour

Après avoir copié l'add-on local :

```sh
ha store reload
ha apps info local_eufy_security_ws_c30
ha apps update local_eufy_security_ws_c30
```

Pour une première installation, utiliser la commande d'installation
correspondant à la version de Home Assistant CLI ou passer par
**Paramètres > Modules complémentaires > Boutique > Vérifier les mises à
jour**.

Avant toute mise à jour :

```sh
ha backups new --name pre-eufy-c30-mqtt
```

## Signes d'un démarrage correct

Les journaux doivent finir par contenir des messages semblables à :

```text
SecurityMQTT connecting to security-mqtt-us.anker.com:8883
SecurityMQTT connected successfully
SecurityMQTT subscribed: cmd/eufy_security/T85D0/SERIAL/res
SecurityMQTT subscribed: cmd/eufy_security/T85D0/SERIAL/req
```

Les erreurs P2P `get_dsk_keys` avec le code `20028` peuvent encore apparaître
durant l'initialisation. Elles ne bloquent pas le contrôle MQTT si les lignes
ci-dessus confirment la connexion et les abonnements Security MQTT.

Une erreur d'abonnement transitoire peut précéder les abonnements réussis. Il
faut vérifier l'état final, pas uniquement la première erreur.

## Sujets MQTT utilisés

Pour une C30 :

```text
cmd/eufy_security/T85D0/NUMERO_DE_SERIE/req
cmd/eufy_security/T85D0/NUMERO_DE_SERIE/res
```

Les jokers MQTT comme `#` sont généralement refusés par les règles du
courtier. Il faut s'abonner aux sujets exacts de chaque serrure.

Les messages utilisent une enveloppe JSON contenant :

- un en-tête de session;
- l'identifiant du compte;
- le numéro de série;
- une charge `trans` encodée en Base64;
- une trame BLE FF09 chiffrée.

La commande de verrouillage/déverrouillage correspond au code de fonction
`ON_OFF_LOCK`. La bibliothèque amont construit et chiffre cette trame; il ne
faut pas fabriquer une commande non chiffrée à la main.

## Heartbeats et état réel

La serrure publie des heartbeats contenant notamment :

- batterie, tag `0xA1`;
- état de serrure, tag `0xA2`.

Pour la C30 testée :

```text
0x03 = déverrouillée
0x04 = verrouillée
```

Le heartbeat permet de corriger l'état optimiste et de refléter les mouvements
effectués depuis l'application Eufy ou physiquement.

## Procédure de validation obligatoire

Effectuer les tests alors qu'une personne peut observer la serrure.

### Test 1 — verrouillage

1. Placer physiquement la serrure en position déverrouillée.
2. Vérifier que HA affiche `unlocked`.
3. Appeler :

```yaml
action: lock.lock
target:
  entity_id: lock.porte_d_entree
```

4. Vérifier dans les journaux :

```text
SecurityMQTT publishing lock command
SecurityMQTT lock command published
SecurityMQTT received lock command response
SecurityMQTT heartbeat
```

5. Vérifier que HA passe à `locked`.
6. Confirmer physiquement que le pêne a bougé.

### Test 2 — déverrouillage

1. Partir de la position physiquement verrouillée.
2. Appeler :

```yaml
action: lock.unlock
target:
  entity_id: lock.porte_d_entree
```

3. Vérifier les mêmes quatre étapes MQTT.
4. Vérifier que HA passe à `unlocked`.
5. Confirmer physiquement que le pêne est rentré.

Le contrôle n'est considéré comme complet qu'après réussite physique dans les
deux directions.

## Dépannage

### HA change d'état, mais la serrure ne bouge pas

Le chemin utilisé est probablement encore P2P ou Cloud optimiste. Rechercher
les lignes `SecurityMQTT`. Vérifier également que la copie imbriquée de
`eufy-security-client` a été supprimée pendant la construction.

### Erreur 20028

Elle correspond à l'absence de DSK P2P de la C30. Ne pas chercher à réparer le
P2P avec une règle NAT ou une simple détection de type 202. Utiliser le
transport Security MQTT.

### Connexion au courtier AIOT, mais abonnement refusé

Vérifier l'hôte. Le contrôle C30 doit utiliser
`security-mqtt-REGION.anker.com`, même si l'API de certificats retourne
`aiot-mqtt-REGION.anker.com`.

### La serrure n'apparaît pas

- vérifier le pays;
- vérifier que l'invitation a été acceptée;
- vérifier les droits du membre;
- vérifier la visibilité dans l'application officielle avec le compte
  secondaire;
- vérifier le numéro de série et le modèle `T85D0`.

### Le contrôle est intermittent

- vérifier la connexion Wi-Fi de la serrure;
- vérifier les accusés sur `/res`;
- attendre le heartbeat avant de conclure;
- ne pas répéter rapidement les commandes;
- vérifier le niveau de batterie;
- comparer avec le comportement de l'application officielle.

## Retour arrière

Avant modification, copier tout le dossier de l'add-on local :

```sh
cp -a \
  /addons/local/eufy_security_ws_c30 \
  /addons/local/eufy_security_ws_c30.before-mqtt
```

Pour revenir en arrière :

1. arrêter l'add-on;
2. restaurer le dossier sauvegardé;
3. recharger la boutique locale;
4. reconstruire ou mettre à jour l'add-on;
5. redémarrer l'add-on;
6. vérifier les journaux et l'entité.

Une sauvegarde complète Home Assistant doit rester disponible en complément.

## Sécurité avant publication

Expurger systématiquement :

- adresses courriel;
- mots de passe;
- jetons d'authentification Eufy;
- jetons FCM;
- certificats et clés privées MQTT;
- identifiants utilisateur;
- numéros de série;
- adresses IP et SSID;
- captures d'écran contenant des informations de domicile.

Après une session de diagnostic :

1. remettre `debug: false`;
2. changer tout mot de passe apparu dans une console ou une conversation;
3. supprimer les copies temporaires de secrets;
4. conserver uniquement les guides anonymisés;
5. créer une sauvegarde post-validation.

## Limites et maintenance

Cette solution dépend d'API Eufy non officielles et peut cesser de fonctionner
après une modification du service Cloud, des certificats, des sujets MQTT ou
du protocole.

La bibliothèque historique `eufy-security-client` est en transition vers
Eufy Mega. Avant toute mise à jour :

- consulter les PR et les notes de version amont;
- reconstruire depuis un commit identifié;
- tester sur une serrure accessible physiquement;
- ne jamais déployer une nouvelle commande de serrure sans possibilité de
  retour arrière.

## Validation de référence

La configuration à l'origine de ce guide a obtenu :

- découverte Cloud du C30;
- connexion mTLS à Security MQTT;
- abonnement exact aux sujets `req` et `res`;
- verrouillage physique confirmé;
- déverrouillage physique confirmé;
- mise à jour HA par heartbeat réel;
- redémarrage de l'add-on avec reconnexion MQTT;
- sauvegarde Home Assistant après validation.

Ce guide est destiné à faciliter les tests communautaires et la contribution
vers une solution amont maintenable, plutôt qu'à créer un fork permanent.
