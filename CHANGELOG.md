# Changelog

All notable changes to the local Home Assistant add-on are documented here.

## 3.1.0-c30.8 — 2026-08-04

- Make connection-time, periodic, and Home Assistant `force_sync` refreshes
  publish a fresh read-only `QUERY_STATUS_IN_LOCK` request through Security
  MQTT instead of returning only cached state.
- Wait until Security MQTT is connected before issuing the connection-time
  query, avoiding a false startup error.
- Fetch pull request 797 explicitly so the pinned client commit remains
  reproducible even though it is not reachable from the upstream default
  branch.
- Install the undeclared `copyfiles` build dependency explicitly.
- Validate patch application, TypeScript compilation, and all 20 Security MQTT
  unit tests.
- Validate on Home Assistant OS: clean add-on start, MQTT connection and exact
  topic subscriptions, a fresh `force_sync` query, and an encrypted response
  containing the current lock state and battery level.

The 2026-08-04 c30.8 validation was read-only: it did not actuate or unlock the
physical lock.

## 3.1.0-c30.7 — 2026-07-29

- Route C30 (`T85D0`) lock and unlock commands through Eufy Security MQTT
  instead of the incompatible P2P/DSK path.
- Decode command responses and heartbeat state/battery updates.
- Avoid the legacy smart-lock MQTT registration used by other models.
- Validate physical lock and unlock, heartbeat correction, Home Assistant state
  updates, and add-on reconnection on real C30 hardware.

