# Eufy Smart Lock C30 in Home Assistant through Security MQTT

Community guide based on a real installation validated on July 29, 2026 with
an Eufy Smart Lock C30, model `T85D0`.

## Validated result

Lock and unlock were validated end to end:

1. Home Assistant called `lock.lock` or `lock.unlock`;
2. the client published the command through Eufy Security MQTT;
3. the lock returned a command acknowledgement;
4. the lock returned a heartbeat containing its actual state;
5. the Home Assistant entity was updated;
6. a person physically confirmed that the mechanism moved.

The Home Assistant optimistic state was never treated as physical proof.

## Why the standard integration fails

The Wi-Fi C30 is usually discovered by `eufy-security-client`, but its
traditional P2P control fails:

```text
get_dsk_keys: code 20028
```

The C30 does not receive the DSK material required by the legacy P2P path.
Adding device type `202` to another lock condition can expose a lock entity,
but it does not make the mechanism move.

A legacy Cloud parameter update may also return success without actuating the
lock. A successful HTTP response is therefore not physical proof.

The working control path is:

```text
Home Assistant
  -> eufy-security-ws
  -> eufy-security-client
  -> encrypted FF09 BLE frame
  -> Eufy Security MQTT
  -> lock Wi-Fi connection
  -> physical mechanism
```

The North American broker is:

```text
security-mqtt-us.anker.com:8883
```

The expected European variant is:

```text
security-mqtt-eu.anker.com:8883
```

Do not confuse it with `aiot-mqtt-us.anker.com`. MQTT client certificates are
provisioned by an AIOT service, but the C30 command topics are hosted by the
**Security MQTT** broker.

## Upstream work

This solution uses:

- `bropat/eufy-security-client` PR 797 by `nick-pape`, implementing C30
  Security MQTT control;
- `bropat/eufy-security-client` PR 939 by `lenoxys`, implementing Eufy Mega
  v6 authentication and transport;
- `bropat/eufy-security-ws`;
- the `fuatakgun/eufy_security` Home Assistant integration.

References:

- <https://github.com/bropat/eufy-security-client/pull/797>
- <https://github.com/bropat/eufy-security-client/pull/939>
- <https://github.com/bropat/eufy-security-client>
- <https://github.com/bropat/eufy-security-ws>
- <https://github.com/fuatakgun/eufy_security>

PR 797 provides the Security MQTT service, BLE frame codec, MQTT envelope,
heartbeat parser, C30 routing, and state synchronization used here.

## Requirements

- Home Assistant OS or Supervised;
- the Home Assistant `eufy_security` integration;
- the `eufy-security-ws` add-on;
- a dedicated secondary Eufy account;
- the lock shared to that account with control permission;
- the account country configured correctly;
- a Home Assistant backup before changing the add-on.

Avoid using the primary household account in an unofficial integration.

## Secondary Eufy account

1. Create a service email address for Home Assistant.
2. Create an Eufy account using that address.
3. Share the C30 from the primary account.
4. Grant the required control permissions.
5. Accept the invitation.
6. Confirm that the C30 is visible in the official app when signed in as the
   secondary account.

Example add-on options:

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

Never publish the actual password, authentication tokens, MQTT certificate or
private key, user identifiers, device serial, IP address, or SSID.

## Building the C30 MQTT client

The add-on in this repository builds the exact tested commit automatically.
For a manual build:

```sh
git clone https://github.com/bropat/eufy-security-client.git
cd eufy-security-client
git fetch origin refs/pull/797/head:pr-797
git switch pr-797
git checkout 1187cf64b201922b99ae360693455505bce2aa09
npm install --ignore-scripts
npm run build
npm pack
```

`npm pack` produces an archive similar to:

```text
eufy-security-client-3.8.0.tgz
```

Always record the exact commit used for a deployment.

## Local Home Assistant add-on

Place the `eufy_security_ws_c30` directory in:

```text
/addons/local/eufy_security_ws_c30
```

The included Dockerfile:

1. clones `bropat/eufy-security-client`;
2. checks out the pinned PR 797 commit;
3. builds and packs the client;
4. installs `eufy-security-ws`;
5. installs the C30 MQTT client at the application root;
6. removes the incompatible nested client dependency.

### Why the nested dependency is removed

`eufy-security-ws` may require a newer semantic version of
`eufy-security-client`. NPM can then retain:

- the C30 MQTT build at the root;
- a different client nested below `eufy-security-ws/node_modules`.

The WebSocket server loads the nested version and silently bypasses the C30
implementation. Removing that nested copy makes Node resolve the tested C30
MQTT build installed at the root.

If this step is missed, commands keep using the old P2P path and the logs do
not show `SecurityMQTT`.

## Installation

Make a full backup first:

```sh
ha backups new --name pre-eufy-c30-mqtt
```

Reload the add-on store:

```sh
ha store reload
```

Install the local add-on from the Home Assistant interface. When updating a
local checkout:

```sh
ha apps info local_eufy_security_ws_c30
ha apps update local_eufy_security_ws_c30
```

The CLI terminology may differ across Home Assistant releases.

## Successful startup

The final logs should contain:

```text
SecurityMQTT connecting to security-mqtt-us.anker.com:8883
SecurityMQTT connected successfully
SecurityMQTT subscribed: cmd/eufy_security/T85D0/SERIAL/res
SecurityMQTT subscribed: cmd/eufy_security/T85D0/SERIAL/req
SecurityMQTT lock status query published
SecurityMQTT queried lock status
```

The C30 patch no longer starts the incompatible P2P/DSK path and no longer
registers the lock on the legacy `smart_lock` MQTT topic. A clean startup
should therefore contain neither error `20028` nor the legacy subscribe error.

## MQTT topics and messages

The C30 uses exact topics:

```text
cmd/eufy_security/T85D0/DEVICE_SERIAL/req
cmd/eufy_security/T85D0/DEVICE_SERIAL/res
```

Broker ACLs commonly reject wildcard subscriptions such as `#`. Subscribe to
each device's exact topics.

Commands use a JSON envelope containing session metadata, account and device
identifiers, a Base64 `trans` value, and an encrypted FF09 BLE frame.
Lock/unlock uses the `ON_OFF_LOCK` function. Use the upstream codec instead of
constructing unencrypted commands.

## Heartbeats and real state

C30 heartbeat frames include:

- battery in TLV tag `0xA1`;
- lock state in TLV tag `0xA2`.

For the validated C30:

```text
0x03 = unlocked
0x04 = locked
```

Heartbeats correct optimistic state and report changes made by the official
app or at the physical lock.

On connection and periodic refresh, the add-on also sends the read-only
`QUERY_STATUS_IN_LOCK` request (BLE code `34`). Its encrypted response provides
the current battery and lock state immediately, without waiting for a
spontaneous heartbeat and without actuating the mechanism.

## Mandatory validation procedure

Perform all tests with a person beside the lock.

### Lock test

1. Physically place the lock in the unlocked position.
2. Confirm that Home Assistant reports `unlocked`.
3. Call:

```yaml
action: lock.lock
target:
  entity_id: lock.front_door
```

4. Confirm these stages in the logs:

```text
SecurityMQTT publishing lock command
SecurityMQTT lock command published
SecurityMQTT received lock command response
SecurityMQTT heartbeat
```

5. Confirm that Home Assistant reports `locked`.
6. Physically confirm that the mechanism moved.

### Unlock test

1. Start from the physically locked position.
2. Call:

```yaml
action: lock.unlock
target:
  entity_id: lock.front_door
```

3. Confirm the four MQTT stages.
4. Confirm that Home Assistant reports `unlocked`.
5. Physically confirm that the mechanism moved.

Control is considered validated only after both physical directions succeed.

## Troubleshooting

### Home Assistant changes state but the lock does not move

The integration is probably still using optimistic P2P or Cloud behavior.
Look for `SecurityMQTT` log messages and verify that the nested
`eufy-security-client` copy was removed during the image build.

### DSK error 20028

This is the C30's missing P2P DSK path. NAT rules and a simple type-202
condition do not fix it. Use Security MQTT.

### AIOT broker connects but subscriptions fail

Verify the broker host. C30 control uses
`security-mqtt-REGION.anker.com`, even when the certificate API reports
`aiot-mqtt-REGION.anker.com`.

### The lock is missing

- verify the configured country;
- verify that the invitation was accepted;
- verify the secondary member's permissions;
- check visibility in the official app under the secondary account;
- verify the model and serial.

### Intermittent commands

- check the lock's Wi-Fi connection and battery;
- wait for the `/res` acknowledgement and heartbeat;
- avoid sending rapid repeated commands;
- compare behavior with the official app.

## Rollback

Before replacing an existing local add-on:

```sh
cp -a \
  /addons/local/eufy_security_ws_c30 \
  /addons/local/eufy_security_ws_c30.before-mqtt
```

To roll back, stop the add-on, restore the saved directory, reload the add-on
store, rebuild or update, restart, and verify the entity. Keep a complete Home
Assistant backup as a second recovery path.

## Security before sharing logs

Always remove:

- email addresses and passwords;
- Eufy authentication and FCM tokens;
- MQTT certificates and private keys;
- user IDs and device serial numbers;
- IP addresses and SSIDs;
- household information in screenshots.

After diagnostics:

1. set `debug: false`;
2. rotate every secret exposed in a console or conversation;
3. remove temporary secret files;
4. retain only sanitized documentation;
5. create a post-validation backup.

## Limitations

This project relies on unofficial Eufy services that may change without
notice. The historical client is also transitioning toward Eufy Mega.

Before every update:

- read the upstream pull requests and release notes;
- build from an identified commit;
- test on physically accessible hardware;
- preserve a rollback path.

This repository is intended to help community testing and upstream
contribution, not to become a permanent unmaintained fork.
