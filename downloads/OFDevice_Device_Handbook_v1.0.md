# OFDevice · Device integration handbook

Version 1.0 · 4 November 2026 · Frontend and backend handoff

Build a device page that reads the current snapshot, subscribes to profile updates and submits asynchronous commands. This handbook describes the implemented service, including its current test environment. Open the HTML directly in a browser; it works offline, with no external scripts or fonts. The companion ZIP contains the same handbook, typed client examples, Postman requests and machine-readable references.

## 1. Start here

The designated device is already enrolled and has a continuous TCP emulator writing simulated telemetry. Start with reads and a battery query. Registration, configuration, SIM activation and calls are separate workflows; opening a device page should not trigger them.

| Setting | Current value |
| --- | --- |
| API base | `https://devices.oceanfriendswebsocket.com` |
| Realtime base | `wss://device-events.oceanfriendswebsocket.com` |
| API explorer | `https://devices.oceanfriendswebsocket.com/docs` |
| OpenAPI | `https://devices.oceanfriendswebsocket.com/openapi.json` |
| Tracker TCP destination | `tracker.oceanfriendswebsocket.com:6060` |
| Client authentication | Temporary public test mode; no token required |
| Browser HTTP credentials | Omit cookies; CORS currently permits any origin |

### Test identity

| Identifier | Value | Where to use it |
| --- | --- | --- |
| deviceId | `cfb2f962-9551-5684-898b-2809dd4cee92` | Canonical `/v1/devices/{device_id}` HTTP routes |
| profileId | `39804d16-943d-4b62-9335-2fda2185ccc7` | WebSocket, legacy API and master profile association |
| IMEI | `866484030221567` | Physical tracker identity; keep as a string |
| Device phone | `+14695400192` | Assigned device number in E.164 format |
| Center | `999` | WebSocket URL segment from master profile metadata |
| ICCID | Unknown | Do not fabricate one or re-register this device to fill it |

These identifiers are different. A profile UUID is not an IMEI, and the device UUID is not the profile UUID. The Telnyx sender number is not the device number. Keep phone and center values as strings and URL-encode path segments when constructing a socket URL.

```text
wss://device-events.oceanfriendswebsocket.com/websocket/39804d16-943d-4b62-9335-2fda2185ccc7/999/+14695400192
```

### First three requests

Readiness checks service dependencies; it does not prove device connectivity.

```sh
curl https://devices.oceanfriendswebsocket.com/readyz
curl https://devices.oceanfriendswebsocket.com/v1/devices/cfb2f962-9551-5684-898b-2809dd4cee92
curl 'https://devices.oceanfriendswebsocket.com/v1/devices/cfb2f962-9551-5684-898b-2809dd4cee92/commands?limit=100'
```

For a deliberate new battery query, generate a fresh UUID for `Idempotency-Key`. Keep that key and the exact body for a network retry of the same intent.

```sh
curl --request POST \
  https://devices.oceanfriendswebsocket.com/v1/devices/cfb2f962-9551-5684-898b-2809dd4cee92/commands \
  --header 'Content-Type: application/json' \
  --header 'Idempotency-Key: replace-with-a-new-uuid' \
  --data-raw '{"type":"battery","sms":"battery","params":{},"transport":"tcp"}'
```

HTTP 202 means the command was accepted. Use the returned `commandId` to find its latest record in GET `/commands`; show success only after its status becomes `succeeded`.

## 2. Frontend implementation checklist

1. Receive the assigned `deviceId`, `profileId`, center and phone from your application backend/profile mapping. For the test device, use Section 1. GET `/v1/devices` can discover canonical devices; public mode currently returns the operator-visible list.
2. Load GET `/v1/devices/{device_id}`, `/commands?limit=100`, `/alerts?limit=100` and `/provisioning`. The snapshot is a direct object; these three history endpoints wrap lists in `{ "data": [...] }`.
3. Open the profile WebSocket, handle its initial resync hint, then refresh the HTTP state. Continue with a JSON ping every 20 seconds. Clean up sockets, requests and timers on route changes/unmount.
4. Display current location, battery, data timestamps, simulation provenance, command progress and alerts. Keep WebSocket connection state separate from device freshness.
5. Submit user actions with a retained idempotency key and an explicit transport. Store `commandId` locally for the pending UI; re-fetch history after hints and during pending-command polling.
6. Show `needs_reconciliation` as an unknown outcome requiring backend review. A browser timeout or lost socket is not evidence that a setting failed or that it is safe to send it again.
7. Keep SMS, calls, SIM lifecycle, configuration and privileged settings behind the application's intended operator workflow. The current public test bypass does not define the eventual end-user permissions.

### Suggested device screen

| UI element | Source / behavior |
| --- | --- |
| Device heading | Profile metadata plus IMEI/deviceId; never use phone as an immutable key |
| Map marker | `lastLocation.latitude` and `.longitude`; accept zero as a valid coordinate |
| Location time | `lastLocation.timestamp` (measurement) and `.receivedAt` (ingestion) |
| Simulated badge | `lastLocation.simulation === true`; also show active emulator context if present |
| Battery | Numeric `batteryLevel`, with `batteryUpdatedAt`; display unknown when missing |
| Last communication | `lastSeen`; derive a clearly labeled freshness indicator using the configured reporting/heartbeat interval |
| Realtime badge | Socket connecting / connected / reconnecting / authentication required; independent of hardware status |
| Command controls | Pending state by commandId; disable duplicate clicks while submitting the same intent |
| Alerts | GET `/alerts`; deduplicate by event identity when available and preserve UTC timestamps |
| Setup progress | GET `/provisioning`; show jobId, status, step and active/blocked command IDs |

A connected WebSocket only proves a browser-to-server connection. Carrier inventory marked `active` only describes SIM lifecycle state. Neither is proof that a physical tracker is currently online. Treat absent fields as unknown rather than zero, false or success.

### Snapshot contract

This is an illustrative subset, not a live response or a complete schema. Fields can be absent until observed. In particular, ownership fields are not yet bound on the owned test profile.

```json
{
  "schemaVersion": 1,
  "deviceId": "cfb2f962-9551-5684-898b-2809dd4cee92",
  "profileId": "39804d16-943d-4b62-9335-2fda2185ccc7",
  "imei": "866484030221567",
  "phoneNumber": "+14695400192",
  "enrollmentActive": true,
  "version": 123,
  "lastSeen": "2026-11-04T02:52:49Z",
  "batteryLevel": 100,
  "batteryUpdatedAt": "2026-11-04T02:52:49Z",
  "lastLocation": {
    "latitude": 47.4753678,
    "longitude": -121.7363316,
    "timestamp": "2026-11-04T02:52:49Z",
    "receivedAt": "2026-11-04T02:52:50Z",
    "simulation": true,
    "simulationRunId": "example-emulation-run"
  },
  "simulation": { "active": true, "writesPersistAfterStop": true }
}
```

Use `lastLocation` and `batteryLevel` for the primary UI. Compatibility fields such as `Currstatus.battery` may be strings. `telemetry` stores parsed command/TLV observations; build a deliberate adapter before using it for a settings form. `commands.<type>.params` represents submitted intent and is not necessarily confirmed readback. The snapshot `version` is a resync/version counter, not the firmware version.

Format times in the user's timezone while retaining the original UTC values. Historical/out-of-order readings do not normally rewind the current projection. Render location accuracy only with a verified unit: this protocol field is not documented here as a meter radius. Do not draw a meter-based accuracy circle by assuming its unit.

## 3. Realtime integration

The socket carries invalidation hints. HTTP snapshots and histories are the state used to render the page. Notifications can be missed, repeated or reordered; reconnect always triggers a fresh HTTP resync.

### Public test handshake

```js
const ws = new WebSocket(
  'wss://device-events.oceanfriendswebsocket.com/websocket/' +
  '39804d16-943d-4b62-9335-2fda2185ccc7/999/' +
  encodeURIComponent('+14695400192')
);
// No first authentication message is required in current public test mode.
// Every 20 seconds after open:
// ws.send(JSON.stringify({ type: 'ping' }));
// Use examples/device-client.ts for cleanup, reconnect and secure-mode support.
```

Initial server message:

```json
{"type":"resync_required","deviceId":"cfb2f962-9551-5684-898b-2809dd4cee92","version":123,"publicTestMode":true}
```

Subsequent messages may use an envelope:

```json
{
  "eventId": "example-event-id",
  "profileId": "39804d16-943d-4b62-9335-2fda2185ccc7",
  "payload": {
    "type": "device_command",
    "event": "command_updated",
    "commandId": "example-command-id",
    "status": "succeeded"
  }
}
```

| Message | Client action |
| --- | --- |
| Top-level `type: resync_required` | Refresh snapshot, command history, alerts and jobs |
| `payload.event: resync_required` | Same; subscription recovery can send this without eventId |
| `device_status_update` | Refresh snapshot |
| `device_command / command_updated` | Refresh commands and snapshot |
| `device_alert / alarm` | Refresh alerts and snapshot |
| `device_alert / geofence_transition` | Refresh snapshot and geofence history; preserve transition direction/time |
| `call_event` | Refresh call status/events/history for the active call |
| `type: pong` | Heartbeat response; no HTTP refresh needed |

Debounce bursts and allow only one refresh per profile at a time. A periodic HTTP refresh (for example every 30 seconds while the page is visible) recovers missed hints. While a command is pending, a bounded 2-second history poll is a reasonable UI fallback; after its UI deadline, keep its real backend status and offer refresh. These intervals are client recommendations, not server SLAs. The supplied hook uses a 3-second pending poll, then returns to 30 seconds.

Reconnect with jittered exponential backoff, capped around 30 seconds. Handle close 1013 by resyncing and reconnecting; 1001 usually indicates shutdown/restart. For 1008, refresh/revalidate authentication and inputs instead of retrying forever. In secure mode the first message must arrive within 10 seconds:

```json
{"type":"authenticate","token":"<FIREBASE_ID_TOKEN>"}
```

Refresh credentials and reconnect proactively before the server's approximately 55-minute connection lifetime. Only `authenticate` (during handshake/current public compatibility) and `ping` are supported client messages. Do not send commands over this socket or put a token into the URL.

## 4. Commands, outcomes and retries

POST `/v1/devices/{device_id}/commands` takes this JSON contract:

| Field | Requirement |
| --- | --- |
| type | Required string, 1–64 characters; command service accepts alphanumeric/underscore names, case-insensitive |
| sms | Required nonempty string, maximum 500 characters; also supplies syntax to the TCP converter |
| params | Object, default `{}`; command-specific values shown in the catalog |
| transport | `auto`, `tcp` or `sms`; default `auto` |
| Idempotency-Key header | Required, 8–128 characters; one stable key per deliberate intent |

`tcp` rejects an unsupported binary mapping with 422. `sms` forces the provider path and requires an assigned destination. `auto` selects a known TCP mapping, otherwise SMS; it does not switch an offline TCP command to SMS just because the tracker is disconnected. An unknown type can become an SMS command under `auto`, so expose an explicit UI catalog rather than accepting arbitrary user type names.

Illustrative subset of an accepted response:

```json
{
  "commandId": "example-command-id",
  "schemaVersion": 1,
  "deviceId": "cfb2f962-9551-5684-898b-2809dd4cee92",
  "profileId": "39804d16-943d-4b62-9335-2fda2185ccc7",
  "type": "battery",
  "command": "battery",
  "delivery": "tcp",
  "transportPreference": "tcp",
  "status": "queued"
}
```

The request field is `sms`; the stored/response text field is `command`. The returned commandId is opaque. GET `/commands?limit=100` returns `{ "data": [...] }`, sorted newest first; locate this commandId in the array. There is no GET `/commands/{commandId}` endpoint. v1 histories are bounded lists, without cursor pagination; use the existing legacy history endpoint with skip/limit for deeper operator history.

| Backend status | Suggested label | FE behavior |
| --- | --- | --- |
| queued | Queued | Persisted; wait for dispatch |
| in_flight | Sending | Worker owns a frame attempt |
| dispatching | Sending | Provider submission underway |
| awaiting_device | Waiting for device | Await device evidence; provider acceptance alone is insufficient |
| succeeded | Completed | Refresh displayed state; for settings, read back if supported |
| failed | Failed | Show available error/evidence; a deliberate new attempt gets a new key |
| expired | Expired | No completed outcome before backend expiry; inspect before repeating an action |
| needs_reconciliation | Outcome unknown | Stop automatic resend and ask backend/operator to inspect evidence |
| cancelled | Cancelled | Closed by backend/operator workflow; there is no generic public cancel route |

Commands run serially per device; multi-frame reads can take longer than one request. The transport ACK identifies a frame attempt using the device/session/sequence. A config/GPS reply can have its own sequence; read completion uses expected data keys/slots in the response window. A read may be satisfied by a compatible observation after the request, so completion does not prove a uniquely solicited GPS measurement.

Only read/query types have bounded automatic retries. The worker does not blindly repeat uncertain setting writes. Transport ACK completion for a setter is not a readback of every resulting setting. SMS provider acceptance is not device application. Record the outcome visible in the API rather than inventing a frontend `acknowledged` or `done` status.

If POST times out, retain the intent and retry the identical request with the same key. Never create a new key automatically to solve an unknown delivery outcome. Capabilities are included in compilation; do not change device capabilities while recovering an idempotent request. The example clients expose intent objects and polling separately so a UI deadline cannot silently issue a second action.

## 5. Settings forms and command examples

All 44 catalog entries and complete request bodies are in Section 10. Use the underscore `type` values there, rather than legacy route names such as `setMode`.

| Control | Request example / constraint |
| --- | --- |
| Refresh | `battery`, `location`, `status`, `check_contacts`, `check_gprs`; current emulator version read is incomplete |
| Tracking mode | `mode` + `mode2,01M,10M`; SMS-format intervals and params must agree |
| Continuous location | `continuous_locate`; interval 10–600 seconds, duration 60–1800 seconds |
| Emergency contact | `contact_1` through `contact_10`; API contact numbering is 1-based |
| Capabilities contact slots | Wire indexes 0–9; these are 0-based and separate from contact command numbering |
| Geofence | `geofence_1` through `_10`; explicit coordinates/radius/direction in params as well as matching SMS text |
| LED | `led1` / `led0` selects the state; do not rely on params.enable for this converter |
| Device TTS | `tts_speak`, with `params.text`; this is different from a provider TTS phone call |
| DND | `ND1` for default 22:00–07:00; `ND0` disables; custom TCP schedules currently have a parsing bug |
| Reboot / off / flush | Explicit user/operator actions; never fire on page load or as an automatic network retry |

### Geofence example

```json
{
  "type": "geofence_1",
  "sms": "Geo1,1,1,300m",
  "params": {
    "enabled": true,
    "trigger_type_code": 1,
    "distance_meters": 300,
    "latitude": 47.4753678,
    "longitude": -121.7363316,
    "name": "Test fence"
  },
  "transport": "tcp"
}
```

Direction 1=enter and 2=exit have TCP mappings. Direction 3=both uses SMS/server behavior; it is not a working TCP emulator action. Keep params and text consistent. GET `/api/getGeofenceData?profileId=...&startDate=2026-11-04T00%3A00%3A00Z` supplies transition history (default: the last day, maximum 1000 results). Use profileId, not IMEI, on this route; startDate accepts ISO 8601 or MM-DD-YYYY. Alert history and geofence transition history are distinct reads.

### Current command limitations

Custom `ND1,22:00,07:00` currently produces a disabled TCP flag because of a converter parsing bug. Use `ND1` for the default window, or a deliberately chosen SMS path for a firmware-supported custom schedule until fixed. Power-on/off alert SMS strings are placeholders that must be filled from the actual firmware's command reference. Alarm-clock slot support is firmware-dependent. BLE disable has no verified TCP mapping. Volume converters clamp values; validate the intended range in the form before submitting.

## 6. Backend integration and onboarding

Keep credentials and carrier/provider integrations on the backend. Your application backend should resolve the user's profile/device mapping, enforce application permissions and expose only the controls intended for that user. FE should never access MongoDB or carry an operator/internal/provider secret.

### New physical device workflow

1. Obtain the real 15-digit IMEI, 19/20-digit ICCID, existing application profileId, tenantId, userId and IANA timezone. Confirm identity before binding; phones can change. The current owned emulator is already set up and does not have a real ICCID yet.
2. Operator: POST `/v1/devices` with the enrollment body below. Store the returned deviceId. Enrollment does not accept phoneNumber or centerId; it does not activate a SIM.
3. If SIM inventory/activation is needed, POST `/verizon/qr/generate` with the real IMEI/ICCID/profileId/tenantId. Store qrId; GET `/verizon/qr/{qr_id}?format=json` reads it, or default PNG yields the QR image.
4. For a SIM that actually needs activation, POST `/verizon/activate` with an idempotency key. Poll GET `/verizon/request/{operationId}`. A carrier callback/poll reconciles provider requestId, lifecycle state and assigned MDN into canonical identity. For an already-active SIM, use the approved backend identity/setup path to bind its actual phone; do not reactivate it merely to populate a number.
5. Once the profile's exact assigned phone matches, POST `/webhook/configure` with a distinct idempotency key. Poll GET `/v1/devices/{device_id}/provisioning` and command history. Activation defaults can also schedule provisioning when skipConfigure=false; inspect existing jobs to avoid duplicate setup.
6. Power/connect the real tracker to `tracker.oceanfriendswebsocket.com:6060`. Stop the owned-device emulator before handing this same IMEI to hardware. Verify fresh, non-simulated lastLocation and command readback.
7. Bind tenantId/userId and the master profile's center metadata before turning secure FE access on. Verify ownership with the real user's token. Reboot is a separate deliberate action after configuration verification.

Enrollment body (placeholders must be replaced with real values):

```json
{
  "profileId": "<EXISTING_PROFILE_ID>",
  "imei": "<REAL_15_DIGIT_IMEI>",
  "iccid": "<REAL_19_OR_20_DIGIT_ICCID>",
  "tenantId": "<TENANT_ID>",
  "userId": "<FIREBASE_UID>",
  "timezone": "America/Los_Angeles"
}
```

Configuration body:

```json
{"profileId":"39804d16-943d-4b62-9335-2fda2185ccc7","phoneNumber":"+14695400192"}
```

POST `/webhook/configure` returns 202 `{ "jobId": "…", "status": "queued" }`. Its 14 sequential steps cover contact, APN, TCP destination, GPRS, Wi-Fi, version, mode, timezone, low battery, answer, call-in, GPS URL, heartbeat and status. Some steps are SMS and the version step needs ICCID: this full workflow is not a TCP-emulator-only test. Jobs expose queued/running/succeeded/failed/needs_reconciliation; a blocked child must be reviewed before continuing.

### Command workflow through the services

```text
FE or application BE
  -> POST /v1/devices/{device_id}/commands  [Idempotency-Key]
  -> API writes DeviceCommands in OceanFriendsDB; returns 202 + commandId
  -> worker owns serialized device slot; creates DeviceAttempts
  -> MemoryDB device stream -> gateway's leased TCP socket -> tracker
  <- tracker transport ACK and/or data frames
  -> gateway validates frame/CRC; appends durable in-VPC MemoryDB ingress
  -> gateway sends requested protocol ACK after durable append
  -> worker processes DeviceInbox; updates snapshot, command evidence, alerts
  -> NotificationOutbox -> Redis PubSub -> profile WebSocket hint
  -> FE fetches GET snapshot /commands /alerts /provisioning
  -> AnalyticsOutbox -> BigQuery asynchronously
```

MongoDB/BigQuery downstream commits are outside the device's ACK latency path. The Redis append must be the configured durable MemoryDB path; a volatile local Redis list is not an equivalent durability guarantee.

### Authentication now and after testing

Current public mode grants client routes operator-equivalent access, including mutations. HTTP omits Authorization; CORS permits `*` without cookies. The backend includes `X-OFDevice-Access-Mode: public-test` on client responses; this header is not currently exposed by CORS for browser JavaScript, so do not use it as the FE's only mode detector. Configure the client's mode explicitly; the WebSocket hello also declares publicTestMode.

After secure mode is restored, send `Authorization: Bearer <FIREBASE_ID_TOKEN>` from FE for v1 reads and the seven read/query commands: battery, status, version, location, gps_location, check_contacts and check_gprs. Matching tenantId and userId ownership are required. Setting/action commands, explicit SMS, enrollment, capabilities, legacy routes and carrier/voice routes need operator privileges. Use the application backend to authorize those operations; never embed an operator token in an FE bundle. Browser origin must also be allowed. The owned profile needs its actual tenant/user binding before secure end-user tests.

Provider callbacks remain authenticated in public mode. Telnyx validates signature + timestamp over the original raw body. Verizon uses the dedicated echoed callback credential and account binding. Internal `/pubsub` and `/internal/*` still require a service bearer token. Neither FE nor an ordinary application client should impersonate a provider callback.

### Backend recovery

GET `/api/operations/health` shows queue depth, worker heartbeats and uncertain outcomes. If a command truly needs reconciliation, an operator may POST `/api/operations/commands/{command_id}/resolve` with outcome `succeeded`, `failed` or `cancelled`, plus 20–2000 characters of observed evidence. This endpoint records an audited resolution; it does not send another device action. There is no public generic cancellation or force-retry endpoint for an active v1 command.

## 7. SMS, calls and Verizon

### SMS

Use the same canonical command route with `transport: sms` for a deliberate SMS action. The command is persisted, the provider is called by the worker, and signed inbound/status events flow through TelnyxInbox. Provider delivery and a device's response are different evidence. Legacy POST `/customCommand` also queues an SMS intent, but v1 provides explicit transport and a caller-controlled idempotency key.

SMS examples in the command catalog are real provider actions when executed. The TCP emulator does not emulate SMS receipt/reply. Do not use SMS-only command success as an emulator acceptance test.

### Voice

| Need | Endpoint and body |
| --- | --- |
| Call assigned device | POST `/api/call/by-profile` · `{ "profile_id": "<PROFILE_ID>" }` |
| Dial explicit device phone | POST `/api/call/initiate` · `{ "phone_number": "<DEVICE_E164>", "profile_id": "<PROFILE_ID>" }` |
| Call colleague then bridge to device | POST `/api/call/two-leg-by-profile` · `{ "your_phone_number": "<COLLEAGUE_E164>", "profile_id": "<PROFILE_ID>" }` |
| Explicit two-leg bridge | POST `/api/call/connect` · `{ "your_phone_number": "<COLLEAGUE_E164>", "device_phone_number": "<DEVICE_E164>", "profile_id": "<PROFILE_ID>" }` |
| Hang up | POST `/api/call/hangup` · `{ "call_control_id": "<CALL_CONTROL_ID>" }` |
| Transfer | POST `/api/call/transfer?call_control_id=...&to_number=...` · query parameters |
| Speak via a phone call | POST `/webhook/ttsSpeakCall` · `{ "profile_id": "<PROFILE_ID>", "text": "Hello", "voice": "female", "language": "en-US", "repeat": 1 }` |
| Browser WebRTC token | POST `/api/webrtc/token`; requires a correctly configured Telnyx credential connection |
| Observe lifecycle | GET `/api/call/status/{call_control_id}`, `/events/{call_control_id}` and `/api/call/history?profile_id=...` |

Use the exact returned call_control_id for later operations; call acceptance is not an answered/bridged call. Voice uses specialized call/voice records and statuses, not the v1 device-command state enum. A socket call_event is a refresh hint. Provider actions can place real calls; emulator testing has not verified live call/SMS delivery or browser WebRTC. These routes do not declare a client Idempotency-Key contract: do not blindly retry a timed-out call/transfer/TTS request. Inspect history/status first. On-device `tts_speak` is a separate TCP command.

### Verizon ThingSpace lifecycle

Verizon SIM management is a backend integration with ThingSpace, rather than a browser connection to a Verizon API. Use local inventory projections for routine UI refreshes; they are distinct from live provider queries and may be stale. Full carrier testing is pending UWS credentials, service plan and MDN ZIP supplied by the owner. The current SIM is marked active based on the owner's report, not a verified carrier activation.

```json
{
  "qrId": "<SAVED_QR_ID>",
  "profileId": "<PROFILE_ID>",
  "servicePlan": "<VERIFIED_SERVICE_PLAN>",
  "mdnZipCode": "<5_DIGIT_ZIP>",
  "skipConfigure": true
}
```

POST the above to `/verizon/activate` only for a device that needs activation, with Idempotency-Key. HTTP 202 returns an operation record; inspect its status even on acceptance, since a competing SIM operation can be rejected. Poll GET `/verizon/request/{operationId}`; the same route also accepts a provider requestId once known. Local operationId, provider requestId and device commandId are separate identifiers. Lifecycle completion is asynchronous; do not assume HTTP 202 means active.

Suspend/restore/deactivate use `{ "identifier": "<IMEI_OR_ICCID_OR_QR_ID>", "withBilling": false, "reasonCode": "FF", "etfWaiver": false, "deleteAfterDeactivation": false }` and their own Idempotency-Key. A profileId can be supplied in its separate field instead. Do not hardcode unverified carrier reason codes/plan choices into the FE; let backend/operator configuration supply them. Destructive provider options are disabled unless explicitly configured.

| Read | Behavior |
| --- | --- |
| GET `/verizon/inventory/{identifier}` | Stored inventory; identifier can be IMEI, ICCID, QR ID or profileId |
| GET `/verizon/sim/{identifier}` | Stored projection by default; `live=true` calls the provider |
| GET `/verizon/location/{identifier}` | Fresh saved tracker GPS first, otherwise carrier location lookup; inspect source/errors and simulation provenance |
| GET `/verizon/usage/{identifier}` | Usage history can lag; not realtime tracker telemetry |
| GET `/verizon/connections/{identifier}` | Provider connection history, up to 90 days in this API |
| GET `/verizon/provisioning/{identifier}` | Provider provisioning history |
| GET `/verizon/plans` | Live provider plan lookup |

For provider histories, keep `latest` fixed across continuation requests, use pagination.nextEarliest as an inclusive boundary and deduplicate rows. If cursorBlocked=true, narrow the time window; do not skip unseen records with the same timestamp. Do not poll provider history/location on each GPS socket hint. A cached `/verizon/location` result can itself be simulated; source=device_gps is not proof of physical hardware.

## 8. Existing clients and database layout

Existing `/webhook/*` command routes and `/api/commands/*` still use the same current OceanFriendsDB and shared command service as v1. There is no literal `/webhook/commands` route. Update old clients to the current API host; paused App Runner/old internal Cloud Run hosts are not compatibility proxies.

```sh
curl --request POST \
  'https://devices.oceanfriendswebsocket.com/webhook/battery?profile_id=39804d16-943d-4b62-9335-2fda2185ccc7'
```

Legacy command handlers mostly take query parameters, not JSON. Their usual HTTP 200 `{ "success": true, "command": "battery" }` means accepted/queued, not completed. The old and v1 histories see the same records. New clients should prefer v1 because its explicit transport and idempotency make intent handling easier.

| Compatibility area | Integration change |
| --- | --- |
| History | GET `/api/commands/history?profile_id=...&limit=50&skip=0` returns success/count/total/commands; v1 returns data |
| Status values | Replace old pending/done comparisons with the current status table |
| Imported records | schemaVersion=0 commands remain in DeviceCommands, quarantined from dispatch |
| Configure | Required Idempotency-Key; 202 jobId/status; exact enrolled profile/phone match required |
| Resend | `/api/commands/resend` expects Mongo `_id` strings from legacy history, not commandId; excludes unsafe uncertain/imported records and currently resets transport selection to auto |
| Custom SMS | `/customCommand` returns 202 command metadata; use v1 for explicit transport/idempotency |
| Raw classified responses | `/internal/command_response` is retired with 410 |
| Realtime | Same URL shape; current contract is resync hints plus HTTP state |

The legacy settings/status/summary helpers retain older aggregation logic and may use historical field names or pending/done counters. For a new device screen, use the canonical snapshot and v1 command history described above; do not treat a legacy aggregate's counters or connection flag as the authoritative current command state.

Compatibility was exercised on 4 November 2026: one legacy battery command and one v1 battery command both completed as succeeded/tcp/schemaVersion=1 and appeared in both histories. That is evidence for the shared path, not a claim that every firmware command was physically tested.

The collection appendix lists maintained collections. MongoDB can create empty collections only when first used; it is not a current document-count report. FE uses APIs; application BE should also prefer the API for device mutations to preserve ownership, serialization and outbox behavior.

## 9. Hosting, emulator and handoff acceptance

### Where the services run

| Original server | Current role | Hosting / entry point |
| --- | --- | --- |
| TCPServer | TCP gateway | AWS ECS Fargate private task, TCP Network Load Balancer on 6060 |
| DeviceServer | Background worker | AWS ECS Fargate private task; no public HTTP service |
| Devicewebhook | HTTP API | AWS ECS Fargate private task behind HTTPS Application Load Balancer |
| ProfileListener | Realtime | Separate AWS ECS Fargate private task behind HTTPS host routing |
| Owned device emulator | Continuous TCP tracker + socket observer | Separate private Fargate task; independent of a developer laptop |

Core roles currently run one task each, 0.5 vCPU/1 GiB, in us-east-1. The emulator runs 0.25 vCPU/512 MiB. MemoryDB provides TLS ingress/transport; MongoDB Atlas stores OceanFriendsDB; BigQuery receives asynchronous analytics. DNS is in Google Cloud DNS; core compute is AWS. Hosting sizes are a dated environment snapshot, not a capacity guarantee.

### What the emulator does

It connects as the owned IMEI through the real TCP gateway, responds to supported reads/settings and writes current simulated GPS/battery through the production ingestion path. Its stationary saved seed is latitude **47.4753678**, longitude **-121.7363316**. Timestamps advance, but the coordinate is not a new physical fix. Fresh setup uses simulated 100% battery. Writes remain stored after the emulator stops; old simulated locations still need the Simulated badge.

The emulator does not activate a SIM, receive cellular SMS, place phone calls or prove hardware firmware behavior. ICCID is unset, so composite version reads can fail at that step. The actual tracker should take over only after BE/ops stops this same-IMEI emulator. A subsequent verified physical GPS session clears active simulation context; retain per-reading provenance when rendering stored history.

### FE acceptance checklist

1. Open the device page and render canonical IDs, saved map point, battery and UTC/local times; identify simulated data visibly.
2. Receive the initial socket hello, resync HTTP state, send JSON pings and see pongs. Reload/change route and ensure no orphaned sockets or duplicate timers remain.
3. Submit one TCP battery/location query; show accepted -> pending -> actual terminal result and refresh the snapshot. Double-click protection uses a single retained intent.
4. Retry a deliberately interrupted HTTP submission with the identical key/body; confirm it resolves to one commandId. A deliberate new action gets a fresh key.
5. Lose/recover network; show reconnect state, resync on recovery, preserve the displayed last-known reading with its age, and accept hints without eventId.
6. Test missing fields, valid 0/0 coordinates, 100% and 0% battery, an empty list, unknown outcomes and validation errors using local fixtures/mocks.
7. Exercise privileged settings only through the intended operator UI. Keep real provider actions out of automated simulator checks.

### BE acceptance checklist

1. Confirm profile/device/IMEI/phone association and that enrollment, inventory and ownership are consistent. No top-level versus nested phone-only duplicate profiles.
2. Check `/readyz`, worker heartbeat/backlog health, command history and the latest telemetry timestamp; a healthy API alone is insufficient.
3. Inspect a read command's terminal evidence and per-reading simulation tags. Confirm legacy/v1 histories are the same ledger when supporting older clients.
4. For secure rollout, verify the actual user's Firebase tenant/UID, allowed browser origin and v1 read permissions; keep provider/internal validation enabled.
5. Separately schedule hardware, SIM/Verizon, SMS delivery, voice/WebRTC and external push tests when those dependencies are available. External mobile push is not yet configured; durable alerts and WebSocket notifications are available.

### Errors to handle

| Result | Integration response |
| --- | --- |
| 200/201/202 | Parse the documented envelope; distinguish acceptance from completion |
| 401 | Missing/invalid token or provider/internal authentication; refresh/revalidate credentials |
| 403 | Role/tenant/origin/provider policy failure; do not retry indefinitely |
| 404 | Wrong/unowned device or missing inventory/operation; recheck ID mapping |
| 409 | Identity conflict, phone mismatch, ineligible reconciliation or carrier intent conflict |
| 410 | Retired endpoint; move to current ingestion/client API |
| 422 | Validation, unsupported TCP mapping or changed command body under the same key; display detail |
| 502/503 | Provider/dependency failure; use bounded backoff for reads and preserve intent keys for eligible writes |
| Network timeout | Outcome unknown; inspect ledger or retry an idempotent intent with the same key |

`detail` can be a string, an object or FastAPI's validation array. Some legacy/provider APIs also return HTTP 200 with success=false/error fields; inspect the body rather than assuming every 200 means a successful business action.

## 10. Complete command payload catalog

Each entry below contains the full body for POST `/v1/devices/{device_id}/commands`. Add `Content-Type: application/json` and a fresh Idempotency-Key for a new intent. TCP examples explicitly select tcp; SMS examples explicitly select sms. Commands with firmware placeholders are templates, not ready-to-send instructions. Expand families/aliases as documented; 44 catalog entries do not mean only 44 possible slot-specific type strings.

### battery

Read · TCP. Battery percentage.

```json
{
  "type": "battery",
  "sms": "battery",
  "params": {},
  "transport": "tcp"
}
```

### status

Read · TCP. Work mode, battery, reporting intervals and all configured contact slots; batch reads require the capability flag.

```json
{
  "type": "status",
  "sms": "status",
  "params": {},
  "transport": "tcp"
}
```

### version

Read · TCP. IMEI, ICCID and firmware. The current emulator has no saved ICCID and rejects that read rather than inventing one.

```json
{
  "type": "version",
  "sms": "V?",
  "params": {},
  "transport": "tcp"
}
```

### location

Read · TCP. Single GPS/location query.

```json
{
  "type": "location",
  "sms": "Loc",
  "params": {},
  "transport": "tcp"
}
```

### gps_location

Read · TCP. Single GPS query; same binary locate instruction as location.

```json
{
  "type": "gps_location",
  "sms": "Loc,gps",
  "params": {},
  "transport": "tcp"
}
```

### check_contacts

Read · TCP. Read every expected contact slot.

```json
{
  "type": "check_contacts",
  "sms": "A?",
  "params": {},
  "transport": "tcp"
}
```

### check_gprs

Read · TCP. Read APN and server configuration.

```json
{
  "type": "check_gprs",
  "sms": "S?",
  "params": {},
  "transport": "tcp"
}
```

### continuous_locate

Tracking · TCP. Interval 10–600 seconds; duration 60–1800 seconds.

```json
{
  "type": "continuous_locate",
  "sms": "CL30S,180S",
  "params": {
    "interval_seconds": 30,
    "duration_seconds": 180
  },
  "transport": "tcp"
}
```

### mode

Settings · TCP. Set mode and its reporting intervals.

```json
{
  "type": "mode",
  "sms": "mode2,01M,10M",
  "params": {
    "mode": 2
  },
  "transport": "tcp"
}
```

### internet

Settings · TCP. Set APN; optional username/password may follow in the SMS-format string.

```json
{
  "type": "internet",
  "sms": "S1,internet",
  "params": {},
  "transport": "tcp"
}
```

### webhook

Settings · TCP. Set tracker TCP destination; this is not an HTTP webhook registration.

```json
{
  "type": "webhook",
  "sms": "IP1,tracker.oceanfriendswebsocket.com,6060",
  "params": {},
  "transport": "tcp"
}
```

### timezone

Settings · TCP. Set timezone offset; the binary unit is 15 minutes.

```json
{
  "type": "timezone",
  "sms": "tz+00:00",
  "params": {},
  "transport": "tcp"
}
```

### contact_N

Contacts · TCP. Example type contact_1. API slot numbers are one based, 1–10; wire indexes are 0–9.

```json
{
  "type": "contact_1",
  "sms": "A1,1,1,+15005550006",
  "params": {},
  "transport": "tcp"
}
```

### remove_contact

Contacts · TCP. Select contact 1 in this example; the binary instruction disables that slot.

```json
{
  "type": "remove_contact",
  "sms": "A1,0,0,0000000000",
  "params": {},
  "transport": "tcp"
}
```

### geofence_N

Geofence · TCP. Example type geofence_1, N=1–10. Direction 1=enter and 2=exit map to TCP; 3=both uses SMS/server behavior.

```json
{
  "type": "geofence_1",
  "sms": "Geo1,1,1,300m",
  "params": {
    "enabled": true,
    "trigger_type_code": 1,
    "distance_meters": 300,
    "latitude": 47.4753678,
    "longitude": -121.7363316,
    "name": "Test fence"
  },
  "transport": "tcp"
}
```

### alarm_clock_N

Settings · TCP. Example type alarm_clock_1; use a clock slot supported by the firmware.

```json
{
  "type": "alarm_clock_1",
  "sms": "CLK1,1,07:30,3,1,2,3,4,5",
  "params": {},
  "transport": "tcp"
}
```

### no_disturb

Settings · TCP. ND1 enables the default 22:00–07:00 window; ND0 disables it. Current custom-schedule parsing has the caveat below.

```json
{
  "type": "no_disturb",
  "sms": "ND1",
  "params": {},
  "transport": "tcp"
}
```

### fall_alarm

Alerts · TCP. Enable fall detection, sensitivity 5, calling disabled.

```json
{
  "type": "fall_alarm",
  "sms": "fl1,5,0",
  "params": {},
  "transport": "tcp"
}
```

### motion_alarm

Alerts · TCP. Motion alarm with cooldown/duration; final flag controls calling.

```json
{
  "type": "motion_alarm",
  "sms": "mo1,05M,10S,0",
  "params": {},
  "transport": "tcp"
}
```

### no_motion_alarm

Alerts · TCP. No-motion alarm; final flag controls calling.

```json
{
  "type": "no_motion_alarm",
  "sms": "nmo1,80M,0",
  "params": {},
  "transport": "tcp"
}
```

### led

Control · TCP. led1 enables; led0 disables. The SMS suffix selects the binary state.

```json
{
  "type": "led",
  "sms": "led1",
  "params": {},
  "transport": "tcp"
}
```

### vibrate

Control · TCP. Vibration on/off.

```json
{
  "type": "vibrate",
  "sms": "Vibrate1",
  "params": {
    "enable": true
  },
  "transport": "tcp"
}
```

### findme / find_me

Control · TCP. Find-me device control; distinct from Beep.

```json
{
  "type": "findme",
  "sms": "findme",
  "params": {},
  "transport": "tcp"
}
```

### reboot

Control · TCP. Reboot request.

```json
{
  "type": "reboot",
  "sms": "Reboot",
  "params": {},
  "transport": "tcp"
}
```

### turn_off

Control · TCP. Power-off request.

```json
{
  "type": "turn_off",
  "sms": "off",
  "params": {},
  "transport": "tcp"
}
```

### flush

Control · TCP. Reset/flush recorded device data.

```json
{
  "type": "flush",
  "sms": "flush",
  "params": {},
  "transport": "tcp"
}
```

### scan_ble / ble

Control · TCP. The enabled path performs a BLE scan. Disabling has no verified TCP mapping and falls back to SMS in auto mode.

```json
{
  "type": "scan_ble",
  "sms": "BLE1",
  "params": {
    "enable": true
  },
  "transport": "tcp"
}
```

### tts_speak

Audio · TCP. On-device TTS. Supply params.text; a Telnyx TTS phone call uses a separate API.

```json
{
  "type": "tts_speak",
  "sms": "tts:Hello",
  "params": {
    "text": "Hello"
  },
  "transport": "tcp"
}
```

### ringtone_volume

Audio · TCP. Ringtone level, clamped to 0–100.

```json
{
  "type": "ringtone_volume",
  "sms": "RT50",
  "params": {},
  "transport": "tcp"
}
```

### mic_volume

Audio · TCP. Microphone level, clamped to 0–15.

```json
{
  "type": "mic_volume",
  "sms": "Micvolume10",
  "params": {},
  "transport": "tcp"
}
```

### speaker_volume

Audio · TCP. Speaker level, clamped to 0–100.

```json
{
  "type": "speaker_volume",
  "sms": "speakervolume50",
  "params": {},
  "transport": "tcp"
}
```

### voice_volume

Audio · TCP. Voice-prompt level, clamped to 0–100.

```json
{
  "type": "voice_volume",
  "sms": "voice50",
  "params": {},
  "transport": "tcp"
}
```

### language

Settings · TCP. Device language code.

```json
{
  "type": "language",
  "sms": "Loen-us",
  "params": {},
  "transport": "tcp"
}
```

### wifi

SMS · SMS. Wi-Fi toggle retains SMS semantics; it is not a one-off binary scan.

Real SMS provider action; not emulated.

```json
{
  "type": "wifi",
  "sms": "Wifi1",
  "params": {},
  "transport": "sms"
}
```

### beep

SMS · SMS. Beep prompt on/off; never translated to a siren.

Real SMS provider action; not emulated.

```json
{
  "type": "beep",
  "sms": "Beep1",
  "params": {},
  "transport": "sms"
}
```

### low_battery

SMS · SMS. Low0,15 disables. SMS preserves other power-alert flags.

Real SMS provider action; not emulated.

```json
{
  "type": "low_battery",
  "sms": "Low1,15",
  "params": {},
  "transport": "sms"
}
```

### answer

SMS · SMS. Automatic-answer setting; Answer1 disables it. SMS preserves unrelated phone flags.

Real SMS provider action; not emulated.

```json
{
  "type": "answer",
  "sms": "Answer0,5",
  "params": {},
  "transport": "sms"
}
```

### callin

SMS · SMS. Call whitelist setting; SMS preserves unrelated phone flags.

Real SMS provider action; not emulated.

```json
{
  "type": "callin",
  "sms": "callin1",
  "params": {},
  "transport": "sms"
}
```

### power_on_alert

SMS · SMS. Passthrough only; replace the placeholder with the model-supported SMS instruction.

Real SMS provider action; not emulated.

```json
{
  "type": "power_on_alert",
  "sms": "<FIRMWARE_POWER_ON_SMS>",
  "params": {},
  "transport": "sms"
}
```

### power_off_alert

SMS · SMS. Passthrough only; replace the placeholder with the model-supported SMS instruction.

Real SMS provider action; not emulated.

```json
{
  "type": "power_off_alert",
  "sms": "<FIRMWARE_POWER_OFF_SMS>",
  "params": {},
  "transport": "sms"
}
```

### heartbeat

SMS · SMS. Application/provisioning command name; no binary converter exists for this type.

Real SMS provider action; not emulated.

```json
{
  "type": "heartbeat",
  "sms": "GPRSHB10M",
  "params": {},
  "transport": "sms"
}
```

### gps_url

SMS · SMS. Application/provisioning command name; SMS passthrough.

Real SMS provider action; not emulated.

```json
{
  "type": "gps_url",
  "sms": "GPSURLwww.google.com/maps?q=%.7f,%.7f",
  "params": {},
  "transport": "sms"
}
```

### gprs

SMS · SMS. Application/provisioning command name; SMS passthrough.

Real SMS provider action; not emulated.

```json
{
  "type": "gprs",
  "sms": "S2",
  "params": {},
  "transport": "sms"
}
```

### custom

SMS · SMS. Any valid alphanumeric/underscore type without a binary mapping also follows SMS under auto mode.

Real SMS provider action; not emulated.

```json
{
  "type": "custom",
  "sms": "<DEVICE_SMS_COMMAND>",
  "params": {},
  "transport": "sms"
}
```


## 11. API endpoint and payload appendix

This appendix includes every declared HTTP operation in the API service, the realtime health/publication routes and the profile WebSocket. Provider callbacks and internal/recovery routes are labeled; they are not ordinary FE actions. The optional undeployed raw HTTP adapter and automatic framework documentation routes are excluded. `/docs`, `/redoc` and `/openapi.json` remain available on the API host.

Field requirements/defaults/limits are taken from the mounted OpenAPI schema. Declared response codes are not exhaustive: middleware and business logic can return additional errors in Section 9. For raw Request handlers, documented JSON examples come from the maintained endpoint inventory rather than inferred OpenAPI body fields. New-device/firmware/provider placeholders must be replaced before use. These examples never execute from the handbook.

### GET /v1/devices

FE/BE · settings require operator in secure mode. Scope by immutable deviceId and tenant/owner. Read snapshots/history/alerts/jobs, enroll identity, or update operator-verified capabilities/enrollment state.

Access: Public test mode enabled; secure-mode rule: Firebase tenant + owner, or operator

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `limit` | query | no | integer | default=100; minimum=1; maximum=500  |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/v1/devices?limit=100'
```

### POST /v1/devices

BE/operator · public during current test mode. Scope by immutable deviceId and tenant/owner. Read snapshots/history/alerts/jobs, enroll identity, or update operator-verified capabilities/enrollment state.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profileId` | body (application/json) | yes | string | minLength=1  |
| `imei` | body (application/json) | yes | string | pattern="^\\d{15}$"  |
| `iccid` | body (application/json) | yes | string | pattern="^\\d{19,20}$"  |
| `tenantId` | body (application/json) | yes | string | minLength=1  |
| `userId` | body (application/json) | yes | string | minLength=1  |
| `timezone` | body (application/json) | no | string | default="UTC"  |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/v1/devices \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "profileId": "sample-profile-001",
  "imei": "990000000000099",
  "iccid": "8900000000000000099",
  "tenantId": "test-tenant",
  "userId": "test-user",
  "timezone": "UTC"
}'
```

### GET /v1/devices/{device_id}

FE/BE · settings require operator in secure mode. Scope by immutable deviceId and tenant/owner. Read snapshots/history/alerts/jobs, enroll identity, or update operator-verified capabilities/enrollment state.

Access: Public test mode enabled; secure-mode rule: Firebase tenant + owner, or operator

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `device_id` | path | yes | string |   |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/v1/devices/<DEVICE_ID>'
```

### GET /v1/devices/{device_id}/alerts

FE/BE · settings require operator in secure mode. Scope by immutable deviceId and tenant/owner. Read snapshots/history/alerts/jobs, enroll identity, or update operator-verified capabilities/enrollment state.

Access: Public test mode enabled; secure-mode rule: Firebase tenant + owner, or operator

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `device_id` | path | yes | string |   |
| `limit` | query | no | integer | default=100; minimum=1; maximum=500  |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/v1/devices/<DEVICE_ID>/alerts?limit=100'
```

### PATCH /v1/devices/{device_id}/capabilities

BE/operator · public during current test mode. Scope by immutable deviceId and tenant/owner. Read snapshots/history/alerts/jobs, enroll identity, or update operator-verified capabilities/enrollment state.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `device_id` | path | yes | string |   |
| `batch_read_verified` | body (application/json) | no | boolean | default=false  |
| `contact_slots` | body (application/json) | no | array<integer> | minItems=1; maxItems=10  |

```sh
curl --request PATCH 'https://devices.oceanfriendswebsocket.com/v1/devices/<DEVICE_ID>/capabilities' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "batch_read_verified": false,
  "contact_slots": [
    0,
    1,
    2
  ]
}'
```

### GET /v1/devices/{device_id}/commands

FE/BE · settings require operator in secure mode. Scope by immutable deviceId and tenant/owner. Read snapshots/history/alerts/jobs, enroll identity, or update operator-verified capabilities/enrollment state.

Access: Public test mode enabled; secure-mode rule: Firebase tenant + owner, or operator

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `device_id` | path | yes | string |   |
| `limit` | query | no | integer | default=100; minimum=1; maximum=500  |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/v1/devices/<DEVICE_ID>/commands?limit=100' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "type": "battery",
  "sms": "battery",
  "params": {},
  "transport": "tcp"
}'
```

### PATCH /v1/devices/{device_id}/enrollment

BE/operator · public during current test mode. Scope by immutable deviceId and tenant/owner. Read snapshots/history/alerts/jobs, enroll identity, or update operator-verified capabilities/enrollment state.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `device_id` | path | yes | string |   |
| `active` | body (application/json) | yes | boolean |   |

```sh
curl --request PATCH 'https://devices.oceanfriendswebsocket.com/v1/devices/<DEVICE_ID>/enrollment' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "active": true
}'
```

### GET /v1/devices/{device_id}/provisioning

FE/BE · settings require operator in secure mode. Scope by immutable deviceId and tenant/owner. Read snapshots/history/alerts/jobs, enroll identity, or update operator-verified capabilities/enrollment state.

Access: Public test mode enabled; secure-mode rule: Firebase tenant + owner, or operator

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `device_id` | path | yes | string |   |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/v1/devices/<DEVICE_ID>/provisioning'
```

### POST /customCommand

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profileId` | body (application/json) | yes | string |   |
| `command` | body (application/json) | yes | string |   |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/customCommand \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "profileId": "<PROFILE_ID>",
  "command": "battery"
}'
```

### POST /v1/devices/{device_id}/commands

FE/BE · settings require operator in secure mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Tenant read/query types; operator for settings/actions

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `device_id` | path | yes | string |   |
| `Idempotency-Key` | header | yes | string | minLength=8; maxLength=128  |
| `type` | body (application/json) | yes | string | minLength=1; maxLength=64  |
| `sms` | body (application/json) | yes | string | minLength=1; maxLength=500  |
| `params` | body (application/json) | no | object |   |
| `transport` | body (application/json) | no | string | default="auto"; enum=["auto", "tcp", "sms"]  |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/v1/devices/<DEVICE_ID>/commands' \
  --header 'Idempotency-Key: <NEW_UUID_PER_INTENT>' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "type": "battery",
  "sms": "battery",
  "params": {},
  "transport": "tcp"
}'
```

### POST /webhook/alarmClock

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `clock_number` | query | yes | integer |   |
| `enable` | query | yes | boolean |   |
| `time` | query | no | string |   |
| `alarm_type` | query | no | integer | default=1  |
| `days` | body (multipart/form-data) | no | array<object> |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/alarmClock?profile_id=%3CPROFILE_ID%3E&clock_number=1&enable=true&time=%3CTIME%3E&alarm_type=1'
```

### POST /webhook/answer

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `auto_answer` | query | yes | boolean |   |
| `seconds` | query | no | integer | default=5  |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/answer?profile_id=%3CPROFILE_ID%3E&auto_answer=true&seconds=5'
```

### POST /webhook/battery

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/battery?profile_id=%3CPROFILE_ID%3E'
```

### POST /webhook/beep

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `enable` | query | yes | boolean |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/beep?profile_id=%3CPROFILE_ID%3E&enable=true'
```

### POST /webhook/callin

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `authorized_only` | query | yes | boolean |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/callin?profile_id=%3CPROFILE_ID%3E&authorized_only=true'
```

### POST /webhook/changeheartbeat

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `interval_seconds` | query | yes | integer |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/changeheartbeat?profile_id=%3CPROFILE_ID%3E&interval_seconds=1'
```

### POST /webhook/continuousLocate

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `interval_seconds` | query | yes | integer |   |
| `duration_seconds` | query | yes | integer |   |
| `name` | query | no | string |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/continuousLocate?profile_id=%3CPROFILE_ID%3E&interval_seconds=30&duration_seconds=180&name=%3CNAME%3E'
```

### POST /webhook/fallAlarm

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `enable` | query | yes | boolean |   |
| `sensitivity` | query | no | integer | default=5  |
| `call` | query | no | boolean | default=true  |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/fallAlarm?profile_id=%3CPROFILE_ID%3E&enable=true&sensitivity=5&call=true'
```

### POST /webhook/findme

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/findme?profile_id=%3CPROFILE_ID%3E'
```

### POST /webhook/flush

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/flush?profile_id=%3CPROFILE_ID%3E'
```

### POST /webhook/geofence

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `fence_number` | query | yes | integer |   |
| `enable` | query | yes | boolean |   |
| `leave_or_enter` | query | yes | integer |   |
| `distance_meters` | query | yes | integer |   |
| `name` | query | no | string |   |
| `lat` | query | no | number |   |
| `lng` | query | no | number |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/geofence?profile_id=%3CPROFILE_ID%3E&fence_number=1&enable=true&leave_or_enter=1&distance_meters=300&name=%3CNAME%3E&lat=47.4753678&lng=-121.7363316'
```

### POST /webhook/gpsLocation

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/gpsLocation?profile_id=%3CPROFILE_ID%3E'
```

### POST /webhook/language

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `language_code` | query | yes | string |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/language?profile_id=%3CPROFILE_ID%3E&language_code=%3CLANGUAGE_CODE%3E'
```

### POST /webhook/led

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `enable` | query | yes | boolean |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/led?profile_id=%3CPROFILE_ID%3E&enable=true'
```

### POST /webhook/location

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/location?profile_id=%3CPROFILE_ID%3E'
```

### POST /webhook/lowBattery

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `enable` | query | yes | boolean |   |
| `level` | query | no | integer | default=15  |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/lowBattery?profile_id=%3CPROFILE_ID%3E&enable=true&level=15'
```

### POST /webhook/micVolume

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `level` | query | yes | integer |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/micVolume?profile_id=%3CPROFILE_ID%3E&level=1'
```

### POST /webhook/motionAlarm

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `enable` | query | yes | boolean |   |
| `static_time` | query | yes | string |   |
| `duration_time` | query | yes | string |   |
| `call` | query | no | boolean | default=true  |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/motionAlarm?profile_id=%3CPROFILE_ID%3E&enable=true&static_time=%3CSTATIC_TIME%3E&duration_time=%3CDURATION_TIME%3E&call=true'
```

### POST /webhook/noDisturb

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `enable` | query | yes | boolean |   |
| `start_time` | query | no | string |   |
| `end_time` | query | no | string |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/noDisturb?profile_id=%3CPROFILE_ID%3E&enable=true&start_time=%3CSTART_TIME%3E&end_time=%3CEND_TIME%3E'
```

### POST /webhook/noMotionAlarm

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `enable` | query | yes | boolean |   |
| `static_time` | query | yes | string |   |
| `call` | query | no | boolean | default=true  |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/noMotionAlarm?profile_id=%3CPROFILE_ID%3E&enable=true&static_time=%3CSTATIC_TIME%3E&call=true'
```

### POST /webhook/reboot

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/reboot?profile_id=%3CPROFILE_ID%3E'
```

### POST /webhook/ringtoneVolume

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `level` | query | yes | integer |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/ringtoneVolume?profile_id=%3CPROFILE_ID%3E&level=1'
```

### POST /webhook/setContact

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `contact_number` | query | yes | integer |   |
| `phone` | query | yes | string |   |
| `sms_enabled` | query | no | integer | default=1  |
| `call_enabled` | query | no | integer | default=1  |
| `name` | query | no | string | default=""  |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/setContact?profile_id=%3CPROFILE_ID%3E&contact_number=1&phone=%2B15005550006&sms_enabled=1&call_enabled=1&name='
```

### POST /webhook/setMode

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `mode` | query | yes | integer |   |
| `motion_interval` | query | no | string |   |
| `no_motion_interval` | query | no | string |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/setMode?profile_id=%3CPROFILE_ID%3E&mode=1&motion_interval=%3CMOTION_INTERVAL%3E&no_motion_interval=%3CNO_MOTION_INTERVAL%3E'
```

### POST /webhook/speakerVolume

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `level` | query | yes | integer |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/speakerVolume?profile_id=%3CPROFILE_ID%3E&level=1'
```

### POST /webhook/status

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/status?profile_id=%3CPROFILE_ID%3E'
```

### POST /webhook/timezone

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `tz_offset` | query | yes | integer |   |
| `minutes` | query | no | integer | default=0  |
| `timezone_name` | query | no | string |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/timezone?profile_id=%3CPROFILE_ID%3E&tz_offset=1&minutes=0&timezone_name=%3CTIMEZONE_NAME%3E'
```

### POST /webhook/ttsSpeak

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `text` | query | yes | string |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/ttsSpeak?profile_id=%3CPROFILE_ID%3E&text=%3CTEXT%3E'
```

### POST /webhook/turnOff

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/turnOff?profile_id=%3CPROFILE_ID%3E'
```

### POST /webhook/vibrate

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `enable` | query | yes | boolean |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/vibrate?profile_id=%3CPROFILE_ID%3E&enable=true'
```

### POST /webhook/voiceVolume

BE/operator · public during current test mode. CommandService.submit persists DeviceCommands before delivery → worker → DeviceAttempts → per-device Redis Stream → owned TCP socket or explicit SMS → device evidence → terminal status.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | yes | string |   |
| `level` | query | yes | integer |   |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/webhook/voiceVolume?profile_id=%3CPROFILE_ID%3E&level=1'
```

### POST /api/geofence/check

BE/operator · public during current test mode. Read UTC-filtered geofence history or evaluate shared freshness/coordinate/hysteresis guards. State transition and pending notification intent are stored atomically.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `imei` | body (application/json) | yes | string |   |
| `latitude` | body (application/json) | yes | number |   |
| `longitude` | body (application/json) | yes | number |   |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/api/geofence/check \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "imei": "990000000000099",
  "latitude": 47.4753678,
  "longitude": -121.7363316
}'
```

### GET /api/getGeofenceData

BE/operator · public during current test mode. Read UTC-filtered geofence history or evaluate shared freshness/coordinate/hysteresis guards. State transition and pending notification intent are stored atomically.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profileId` | query | yes | string |   |
| `startDate` | query | no | string or null |   |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/api/getGeofenceData?profileId=%3CPROFILE_ID%3E&startDate=%3CSTARTDATE%3E'
```

### GET /healthz

Health check. Liveness is independent of downstream delivery. Readiness checks required Mongo/Redis connectivity; inspect worker heartbeats and backlogs separately.

Access: Public health/readiness

```sh
curl --request GET https://devices.oceanfriendswebsocket.com/healthz
```

### GET /readyz

Health check. Liveness is independent of downstream delivery. Readiness checks required Mongo/Redis connectivity; inspect worker heartbeats and backlogs separately.

Access: Public health/readiness

```sh
curl --request GET https://devices.oceanfriendswebsocket.com/readyz
```

### GET /api/commands/history

BE/operator · public during current test mode. Operator-only compatibility API. Reads profile/status/history projections; use canonical /v1/devices endpoints for tenant clients.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | no | string or null |  Filter by profile ID |
| `status` | query | no | string or null |  Filter by status (pending, done, failed) |
| `limit` | query | no | integer | default=50; minimum=1; maximum=500 Number of commands to return |
| `skip` | query | no | integer | default=0; minimum=0 Number of commands to skip |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/api/commands/history?profile_id=%3CPROFILE_ID%3E&status=%3CSTATUS%3E&limit=50&skip=0'
```

### GET /api/commands/pending

BE/operator · public during current test mode. Operator-only compatibility API. Reads profile/status/history projections; use canonical /v1/devices endpoints for tenant clients.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | no | string or null |  Filter by profile ID |
| `limit` | query | no | integer | default=100; minimum=1; maximum=500 Number of commands to return |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/api/commands/pending?profile_id=%3CPROFILE_ID%3E&limit=100'
```

### POST /api/commands/resend

BE/operator · public during current test mode. Operator-only compatibility API. Reads profile/status/history projections; use canonical /v1/devices endpoints for tenant clients.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `command_ids` | body (application/json) | yes | array<string> |  List of command IDs to resend |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/api/commands/resend \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "command_ids": [
    "<COMMAND_IDS>"
  ]
}'
```

### GET /api/device/profile/{profile_id}

BE/operator · public during current test mode. Operator-only compatibility API. Reads profile/status/history projections; use canonical /v1/devices endpoints for tenant clients.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | path | yes | string |   |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/api/device/profile/<PROFILE_ID>'
```

### GET /api/device/profiles

BE/operator · public during current test mode. Operator-only compatibility API. Reads profile/status/history projections; use canonical /v1/devices endpoints for tenant clients.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `limit` | query | no | integer | default=50; minimum=1; maximum=500 Number of profiles to return |
| `skip` | query | no | integer | default=0; minimum=0 Number of profiles to skip |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/api/device/profiles?limit=50&skip=0'
```

### GET /api/device/settings/{profile_id}

BE/operator · public during current test mode. Operator-only compatibility API. Reads profile/status/history projections; use canonical /v1/devices endpoints for tenant clients.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | path | yes | string |   |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/api/device/settings/<PROFILE_ID>'
```

### GET /api/device/status/{profile_id}

BE/operator · public during current test mode. Operator-only compatibility API. Reads profile/status/history projections; use canonical /v1/devices endpoints for tenant clients.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | path | yes | string |   |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/api/device/status/<PROFILE_ID>'
```

### GET /api/device/summary/{profile_id}

BE/operator · public during current test mode. Operator-only compatibility API. Reads profile/status/history projections; use canonical /v1/devices endpoints for tenant clients.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | path | yes | string |   |
| `live` | query | no | boolean | default=false Include live Verizon SIM state |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/api/device/summary/<PROFILE_ID>?live=false'
```

### GET /api/messages/history

BE/operator · public during current test mode. Operator-only compatibility API. Reads profile/status/history projections; use canonical /v1/devices endpoints for tenant clients.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | no | string or null |  Filter by profile ID |
| `phone_number` | query | no | string or null |  Filter by phone number |
| `limit` | query | no | integer | default=50; minimum=1; maximum=500 Number of messages to return |
| `skip` | query | no | integer | default=0; minimum=0 Number of messages to skip |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/api/messages/history?profile_id=%3CPROFILE_ID%3E&phone_number=%2B15005550006&limit=50&skip=0'
```

### POST /webhook/checkDeviceStatus

BE/operator · public during current test mode. Operator-only compatibility API. Reads profile/status/history projections; use canonical /v1/devices endpoints for tenant clients.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/webhook/checkDeviceStatus \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "profileId": "<PROFILE_ID>"
}'
```

### POST /api/operations/carrier/{operation_id}/resolve

BE/operator · public during current test mode. Reads recovery/backlog evidence or records an explicit audited reconciliation decision. Resolution endpoints send no device/carrier action.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `operation_id` | path | yes | string |   |
| `requestId` | body (application/json) | no | string or null |   |
| `verifiedNotApplied` | body (application/json) | no | boolean | default=false  |
| `evidence` | body (application/json) | yes | string | minLength=20; maxLength=2000  |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/api/operations/carrier/<OPERATION_ID>/resolve' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "requestId": "<REQUESTID>",
  "verifiedNotApplied": false,
  "evidence": "<EVIDENCE>"
}'
```

### POST /api/operations/commands/{command_id}/resolve

BE/operator · public during current test mode. Reads recovery/backlog evidence or records an explicit audited reconciliation decision. Resolution endpoints send no device/carrier action.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `command_id` | path | yes | string |   |
| `outcome` | body (application/json) | yes | string | pattern="^(succeeded/failed/cancelled)$"  |
| `evidence` | body (application/json) | yes | string | minLength=20; maxLength=2000  |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/api/operations/commands/<COMMAND_ID>/resolve' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "outcome": "failed",
  "evidence": "<REPLACE_WITH_AT_LEAST_20_CHARACTERS_OF_VERIFIED_EVIDENCE>"
}'
```

### POST /api/operations/enrollment/rebuild

BE/operator · public during current test mode. Reads recovery/backlog evidence or records an explicit audited reconciliation decision. Resolution endpoints send no device/carrier action.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/api/operations/enrollment/rebuild
```

### GET /api/operations/health

BE/operator · public during current test mode. Reads recovery/backlog evidence or records an explicit audited reconciliation decision. Resolution endpoints send no device/carrier action.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

```sh
curl --request GET https://devices.oceanfriendswebsocket.com/api/operations/health
```

### POST /accept

Provider callback · verified sender only. Verify raw Telnyx signature before JSON processing. SMS/voice/status receipts enter TelnyxInbox; workers apply device/call evidence. /failure currently acknowledges a verified notification without initiating work.

Access: Telnyx Ed25519 signature + 300s timestamp window

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/accept \
  --header 'telnyx-timestamp: <PROVIDER_TIMESTAMP>' \
  --header 'telnyx-signature-ed25519: <SIGNATURE_OVER_RAW_BODY>' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "data": {
    "id": "sample-provider-event",
    "event_type": "message.received",
    "payload": {
      "from": {
        "phone_number": "+15005550006"
      },
      "to": [
        {
          "phone_number": "<TELNYX_NUMBER>"
        }
      ],
      "text": "Battery: 100%"
    }
  }
}'
```

### POST /failure

Provider callback · verified sender only. Verify raw Telnyx signature before JSON processing. SMS/voice/status receipts enter TelnyxInbox; workers apply device/call evidence. /failure currently acknowledges a verified notification without initiating work.

Access: Telnyx Ed25519 signature + 300s timestamp window

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/failure \
  --header 'telnyx-timestamp: <PROVIDER_TIMESTAMP>' \
  --header 'telnyx-signature-ed25519: <SIGNATURE_OVER_RAW_BODY>' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "data": {
    "id": "sample-provider-event",
    "event_type": "message.received",
    "payload": {
      "from": {
        "phone_number": "+15005550006"
      },
      "to": [
        {
          "phone_number": "<TELNYX_NUMBER>"
        }
      ],
      "text": "Battery: 100%"
    }
  }
}'
```

### POST /verizon/accept

Provider callback · verified sender only. Validate echoed secret/account → recursively sanitize/deduplicate VerizonInbox → 2xx durable receipt → worker reconcile/reducer → inventory/profile and optional provisioning job.

Access: Verizon echoed callback secret + account binding

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/verizon/accept \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "requestId": "<PROVIDER_REQUEST_ID>",
  "password": "<CALLBACK_SECRET>",
  "accountName": "<ACCOUNT_NAME>",
  "status": "Success",
  "deviceIds": [
    {
      "kind": "imei",
      "id": "990000000000099"
    }
  ]
}'
```

### POST /webhook/call

Provider callback · verified sender only. Verify raw Telnyx signature before JSON processing. SMS/voice/status receipts enter TelnyxInbox; workers apply device/call evidence. /failure currently acknowledges a verified notification without initiating work.

Access: Telnyx Ed25519 signature + 300s timestamp window

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/webhook/call \
  --header 'telnyx-timestamp: <PROVIDER_TIMESTAMP>' \
  --header 'telnyx-signature-ed25519: <SIGNATURE_OVER_RAW_BODY>' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "data": {
    "id": "sample-provider-event",
    "event_type": "call.answered",
    "payload": {
      "call_control_id": "<CALL_CONTROL_ID>"
    }
  }
}'
```

### POST /webhook/sms

Provider callback · verified sender only. Verify raw Telnyx signature before JSON processing. SMS/voice/status receipts enter TelnyxInbox; workers apply device/call evidence. /failure currently acknowledges a verified notification without initiating work.

Access: Telnyx Ed25519 signature + 300s timestamp window

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/webhook/sms \
  --header 'telnyx-timestamp: <PROVIDER_TIMESTAMP>' \
  --header 'telnyx-signature-ed25519: <SIGNATURE_OVER_RAW_BODY>' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "data": {
    "id": "sample-provider-event",
    "event_type": "message.received",
    "payload": {
      "from": {
        "phone_number": "+15005550006"
      },
      "to": [
        {
          "phone_number": "<TELNYX_NUMBER>"
        }
      ],
      "text": "Battery: 100%"
    }
  }
}'
```

### POST /webhook/telenyxstatus

Provider callback · verified sender only. Verify raw Telnyx signature before JSON processing. SMS/voice/status receipts enter TelnyxInbox; workers apply device/call evidence. /failure currently acknowledges a verified notification without initiating work.

Access: Telnyx Ed25519 signature + 300s timestamp window

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/webhook/telenyxstatus \
  --header 'telnyx-timestamp: <PROVIDER_TIMESTAMP>' \
  --header 'telnyx-signature-ed25519: <SIGNATURE_OVER_RAW_BODY>' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "data": {
    "id": "sample-provider-event",
    "event_type": "message.received",
    "payload": {
      "from": {
        "phone_number": "+15005550006"
      },
      "to": [
        {
          "phone_number": "<TELNYX_NUMBER>"
        }
      ],
      "text": "Battery: 100%"
    }
  }
}'
```

### POST /webhook/configure

BE/operator · public during current test mode. create_job → ProvisioningJobs → worker.schedule/provision → one child command at a time; inspect the job before completion.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `Idempotency-Key` | header | yes | string | minLength=8; maxLength=128  |
| `phoneNumber` | body (application/json) | yes | string |   |
| `profileId` | body (application/json) | yes | string |   |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/webhook/configure \
  --header 'Idempotency-Key: <NEW_UUID_PER_INTENT>' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "phoneNumber": "+15005550006",
  "profileId": "<PROFILE_ID>"
}'
```

### POST /internal/command_response

Internal service · not FE. Always 410. Classified responses cannot safely correlate attempts; use the TCP gateway/raw durable inbox.

Access: Internal bearer token

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `imei` | body (application/json) | yes | string |   |
| `response_type` | body (application/json) | yes | string |   |
| `data` | body (application/json) | no | object | default={}  |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/internal/command_response \
  --header 'Authorization: Bearer <INTERNAL_TOKEN>' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "imei": "990000000000099",
  "response_type": "<RESPONSE_TYPE>",
  "data": {}
}'
```

### GET /verizon/callbacks

BE/operator · public during current test mode. Operator-only provider callback administration. Registration requires the exact approved HTTPS URL; results redact credentials. Deletion is disabled unless explicitly configured.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

```sh
curl --request GET https://devices.oceanfriendswebsocket.com/verizon/callbacks
```

### POST /verizon/callbacks/register

BE/operator · public during current test mode. Operator-only provider callback administration. Registration requires the exact approved HTTPS URL; results redact credentials. Deletion is disabled unless explicitly configured.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `serviceName` | body (application/json) | yes | string |   |
| `url` | body (application/json) | yes | string |   |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/verizon/callbacks/register \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "serviceName": "<SERVICENAME>",
  "url": "<URL>"
}'
```

### DELETE /verizon/callbacks/{service_name}

BE/operator · public during current test mode. Operator-only provider callback administration. Registration requires the exact approved HTTPS URL; results redact credentials. Deletion is disabled unless explicitly configured.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `service_name` | path | yes | string |   |

```sh
curl --request DELETE 'https://devices.oceanfriendswebsocket.com/verizon/callbacks/<SERVICE_NAME>'
```

### POST /verizon/activate

BE/operator · public during current test mode. VerizonService.submit → VerizonRequests + DeviceInventory pendingOperation → 202 operationId → worker dispatch → provider requestId → callback/poll reducer.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `Idempotency-Key` | header | yes | string | minLength=8; maxLength=128  |
| `qrId` | body (application/json) | no | string or null |   |
| `profileId` | body (application/json) | no | string or null |   |
| `imei` | body (application/json) | no | string or null |   |
| `iccid` | body (application/json) | no | string or null |   |
| `servicePlan` | body (application/json) | no | string or null |   |
| `mdnZipCode` | body (application/json) | no | string or null |   |
| `skipConfigure` | body (application/json) | no | boolean | default=false  |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/verizon/activate \
  --header 'Idempotency-Key: <NEW_UUID_PER_INTENT>' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "qrId": "<QRID>",
  "profileId": "<PROFILE_ID>",
  "imei": "990000000000099",
  "iccid": "8900000000000000099",
  "servicePlan": "<SERVICEPLAN>",
  "mdnZipCode": "<MDNZIPCODE>",
  "skipConfigure": false
}'
```

### POST /verizon/deactivate

BE/operator · public during current test mode. VerizonService.submit → VerizonRequests + DeviceInventory pendingOperation → 202 operationId → worker dispatch → provider requestId → callback/poll reducer.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `Idempotency-Key` | header | yes | string | minLength=8; maxLength=128  |
| `identifier` | body (application/json) | no | string or null |   |
| `profileId` | body (application/json) | no | string or null |   |
| `withBilling` | body (application/json) | no | boolean | default=false  |
| `reasonCode` | body (application/json) | no | string | default="FF"  |
| `etfWaiver` | body (application/json) | no | boolean | default=false  |
| `deleteAfterDeactivation` | body (application/json) | no | boolean | default=false  |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/verizon/deactivate \
  --header 'Idempotency-Key: <NEW_UUID_PER_INTENT>' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "identifier": "<IDENTIFIER>",
  "profileId": "<PROFILE_ID>",
  "withBilling": false,
  "reasonCode": "FF",
  "etfWaiver": false,
  "deleteAfterDeactivation": false
}'
```

### POST /verizon/restore

BE/operator · public during current test mode. VerizonService.submit → VerizonRequests + DeviceInventory pendingOperation → 202 operationId → worker dispatch → provider requestId → callback/poll reducer.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `Idempotency-Key` | header | yes | string | minLength=8; maxLength=128  |
| `identifier` | body (application/json) | no | string or null |   |
| `profileId` | body (application/json) | no | string or null |   |
| `withBilling` | body (application/json) | no | boolean | default=false  |
| `reasonCode` | body (application/json) | no | string | default="FF"  |
| `etfWaiver` | body (application/json) | no | boolean | default=false  |
| `deleteAfterDeactivation` | body (application/json) | no | boolean | default=false  |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/verizon/restore \
  --header 'Idempotency-Key: <NEW_UUID_PER_INTENT>' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "identifier": "<IDENTIFIER>",
  "profileId": "<PROFILE_ID>",
  "withBilling": false,
  "reasonCode": "FF",
  "etfWaiver": false,
  "deleteAfterDeactivation": false
}'
```

### POST /verizon/suspend

BE/operator · public during current test mode. VerizonService.submit → VerizonRequests + DeviceInventory pendingOperation → 202 operationId → worker dispatch → provider requestId → callback/poll reducer.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `Idempotency-Key` | header | yes | string | minLength=8; maxLength=128  |
| `identifier` | body (application/json) | no | string or null |   |
| `profileId` | body (application/json) | no | string or null |   |
| `withBilling` | body (application/json) | no | boolean | default=false  |
| `reasonCode` | body (application/json) | no | string | default="FF"  |
| `etfWaiver` | body (application/json) | no | boolean | default=false  |
| `deleteAfterDeactivation` | body (application/json) | no | boolean | default=false  |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/verizon/suspend \
  --header 'Idempotency-Key: <NEW_UUID_PER_INTENT>' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "identifier": "<IDENTIFIER>",
  "profileId": "<PROFILE_ID>",
  "withBilling": false,
  "reasonCode": "FF",
  "etfWaiver": false,
  "deleteAfterDeactivation": false
}'
```

### GET /verizon/connections/{identifier}

BE/operator · public during current test mode. Query provider history; return fixed-window continuation metadata, hasMoreData and inclusive-boundary limitations.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `identifier` | path | yes | string |   |
| `days` | query | no | integer | default=7; minimum=1; maximum=90  |
| `earliest` | query | no | string or null |   |
| `latest` | query | no | string or null |   |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/verizon/connections/<IDENTIFIER>?days=7&earliest=%3CEARLIEST%3E&latest=%3CLATEST%3E'
```

### GET /verizon/inventory

BE/operator · public during current test mode. Read inventory/operation projection, QR metadata or the requested live provider information. HTTP success is separate from carrier completion.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `status` | query | no | string or null |   |
| `limit` | query | no | integer | default=100; minimum=1; maximum=500  |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/verizon/inventory?status=%3CSTATUS%3E&limit=100'
```

### GET /verizon/inventory/{identifier}

BE/operator · public during current test mode. Read inventory/operation projection, QR metadata or the requested live provider information. HTTP success is separate from carrier completion.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `identifier` | path | yes | string |   |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/verizon/inventory/<IDENTIFIER>'
```

### GET /verizon/location/{identifier}

BE/operator · public during current test mode. Use fresh cached tracker GPS when available; otherwise POST ThingSpace synchronous location using IMEI + MDN. Return each provider result/error without asserting offline reachability.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `identifier` | path | yes | string |   |
| `max_local_age_seconds` | query | no | integer | default=120; minimum=0; maximum=3600  |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/verizon/location/<IDENTIFIER>?max_local_age_seconds=120'
```

### GET /verizon/plans

BE/operator · public during current test mode. Read inventory/operation projection, QR metadata or the requested live provider information. HTTP success is separate from carrier completion.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

```sh
curl --request GET https://devices.oceanfriendswebsocket.com/verizon/plans
```

### GET /verizon/provisioning/{identifier}

BE/operator · public during current test mode. Query provider history; return fixed-window continuation metadata, hasMoreData and inclusive-boundary limitations.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `identifier` | path | yes | string |   |
| `days` | query | no | integer | default=180; minimum=1; maximum=365  |
| `earliest` | query | no | string or null |   |
| `latest` | query | no | string or null |   |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/verizon/provisioning/<IDENTIFIER>?days=180&earliest=%3CEARLIEST%3E&latest=%3CLATEST%3E'
```

### POST /verizon/qr/generate

BE/operator · public during current test mode. Read inventory/operation projection, QR metadata or the requested live provider information. HTTP success is separate from carrier completion.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `imei` | body (application/json) | yes | string | pattern="^\\d{15}$"  |
| `iccid` | body (application/json) | yes | string | pattern="^\\d{19,20}$"  |
| `profileId` | body (application/json) | no | string or null |   |
| `tenantId` | body (application/json) | no | string or null |   |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/verizon/qr/generate \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "imei": "990000000000099",
  "iccid": "8900000000000000099",
  "profileId": "<PROFILE_ID>",
  "tenantId": "test-tenant"
}'
```

### GET /verizon/qr/{qr_id}

BE/operator · public during current test mode. Read inventory/operation projection, QR metadata or the requested live provider information. HTTP success is separate from carrier completion.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `qr_id` | path | yes | string |   |
| `format` | query | no | string | default="png"; pattern="^(png/json)$"  |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/verizon/qr/<QR_ID>?format=png'
```

### GET /verizon/request/{request_id}

BE/operator · public during current test mode. Read inventory/operation projection, QR metadata or the requested live provider information. HTTP success is separate from carrier completion.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `request_id` | path | yes | string |   |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/verizon/request/<REQUEST_ID>'
```

### GET /verizon/sim/{identifier}

BE/operator · public during current test mode. Read inventory/operation projection, QR metadata or the requested live provider information. HTTP success is separate from carrier completion.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `identifier` | path | yes | string |   |
| `live` | query | no | boolean | default=false  |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/verizon/sim/<IDENTIFIER>?live=false'
```

### GET /verizon/usage/{identifier}

BE/operator · public during current test mode. Query provider history; return fixed-window continuation metadata, hasMoreData and inclusive-boundary limitations.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `identifier` | path | yes | string |   |
| `days` | query | no | integer | default=30; minimum=1; maximum=365  |
| `earliest` | query | no | string or null |   |
| `latest` | query | no | string or null |   |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/verizon/usage/<IDENTIFIER>?days=30&earliest=%3CEARLIEST%3E&latest=%3CLATEST%3E'
```

### GET /api/call/active

BE/operator · public during current test mode. Read call state/history or execute the explicit Telnyx voice action. ProviderActions records precede outbound effects; durable signed callbacks drive later call state and notifications.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

```sh
curl --request GET https://devices.oceanfriendswebsocket.com/api/call/active
```

### POST /api/call/by-profile

BE/operator · public during current test mode. Read call state/history or execute the explicit Telnyx voice action. ProviderActions records precede outbound effects; durable signed callbacks drive later call state and notifications.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | body (application/json) | yes | string |  The profile ID of the device to call |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/api/call/by-profile \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "profile_id": "<PROFILE_ID>"
}'
```

### GET /api/call/config

BE/operator · public during current test mode. Read call state/history or execute the explicit Telnyx voice action. ProviderActions records precede outbound effects; durable signed callbacks drive later call state and notifications.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

```sh
curl --request GET https://devices.oceanfriendswebsocket.com/api/call/config
```

### POST /api/call/connect

BE/operator · public during current test mode. Read call state/history or execute the explicit Telnyx voice action. ProviderActions records precede outbound effects; durable signed callbacks drive later call state and notifications.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `your_phone_number` | body (application/json) | yes | string |  Your phone number - we will call you first |
| `device_phone_number` | body (application/json) | yes | string |  The device phone number to connect to |
| `profile_id` | body (application/json) | no | string or null |  Optional profile ID for tracking |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/api/call/connect \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "your_phone_number": "<COLLEAGUE_E164>",
  "device_phone_number": "<DEVICE_E164>",
  "profile_id": "<PROFILE_ID>"
}'
```

### GET /api/call/events/{call_control_id}

BE/operator · public during current test mode. Read call state/history or execute the explicit Telnyx voice action. ProviderActions records precede outbound effects; durable signed callbacks drive later call state and notifications.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `call_control_id` | path | yes | string |   |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/api/call/events/<CALL_CONTROL_ID>'
```

### GET /api/call/failures

BE/operator · public during current test mode. Read call state/history or execute the explicit Telnyx voice action. ProviderActions records precede outbound effects; durable signed callbacks drive later call state and notifications.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | no | string or null |   |
| `phone_number` | query | no | string or null |   |
| `limit` | query | no | integer | default=20; minimum=1; maximum=100  |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/api/call/failures?profile_id=%3CPROFILE_ID%3E&phone_number=%2B15005550006&limit=20'
```

### POST /api/call/hangup

BE/operator · public during current test mode. Read call state/history or execute the explicit Telnyx voice action. ProviderActions records precede outbound effects; durable signed callbacks drive later call state and notifications.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `call_control_id` | body (application/json) | yes | string |  The call control ID to hang up |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/api/call/hangup \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "call_control_id": "<CALL_CONTROL_ID>"
}'
```

### GET /api/call/history

BE/operator · public during current test mode. Read call state/history or execute the explicit Telnyx voice action. ProviderActions records precede outbound effects; durable signed callbacks drive later call state and notifications.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | query | no | string or null |   |
| `phone_number` | query | no | string or null |   |
| `limit` | query | no | integer | default=20; minimum=1; maximum=100  |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/api/call/history?profile_id=%3CPROFILE_ID%3E&phone_number=%2B15005550006&limit=20'
```

### POST /api/call/initiate

BE/operator · public during current test mode. Read call state/history or execute the explicit Telnyx voice action. ProviderActions records precede outbound effects; durable signed callbacks drive later call state and notifications.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `phone_number` | body (application/json) | yes | string |  The phone number to call (device phone number) |
| `profile_id` | body (application/json) | no | string or null |  Optional profile ID for tracking |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/api/call/initiate \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "phone_number": "+15005550006",
  "profile_id": "<PROFILE_ID>"
}'
```

### GET /api/call/status/{call_control_id}

BE/operator · public during current test mode. Read call state/history or execute the explicit Telnyx voice action. ProviderActions records precede outbound effects; durable signed callbacks drive later call state and notifications.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `call_control_id` | path | yes | string |   |

```sh
curl --request GET 'https://devices.oceanfriendswebsocket.com/api/call/status/<CALL_CONTROL_ID>'
```

### POST /api/call/transfer

BE/operator · public during current test mode. Read call state/history or execute the explicit Telnyx voice action. ProviderActions records precede outbound effects; durable signed callbacks drive later call state and notifications.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `call_control_id` | query | yes | string |  The call control ID |
| `to_number` | query | yes | string |  The number to transfer to |

```sh
curl --request POST 'https://devices.oceanfriendswebsocket.com/api/call/transfer?call_control_id=%3CCALL_CONTROL_ID%3E&to_number=%3CTO_NUMBER%3E'
```

### POST /api/call/two-leg-by-profile

BE/operator · public during current test mode. Read call state/history or execute the explicit Telnyx voice action. ProviderActions records precede outbound effects; durable signed callbacks drive later call state and notifications.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `your_phone_number` | body (application/json) | yes | string |  Your phone number - we will call you first |
| `profile_id` | body (application/json) | yes | string |  The profile ID of the device to connect to |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/api/call/two-leg-by-profile \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "your_phone_number": "<COLLEAGUE_E164>",
  "profile_id": "<PROFILE_ID>"
}'
```

### POST /api/webrtc/token

BE/operator · public during current test mode. Read call state/history or execute the explicit Telnyx voice action. ProviderActions records precede outbound effects; durable signed callbacks drive later call state and notifications.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/api/webrtc/token
```

### POST /webhook/ttsSpeakCall

BE/operator · public during current test mode. Read call state/history or execute the explicit Telnyx voice action. ProviderActions records precede outbound effects; durable signed callbacks drive later call state and notifications.

Access: Public test mode enabled; secure-mode rule: Operator bearer/API key or Firebase ofdeviceOperator claim

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `profile_id` | body (application/json) | yes | string |  The profile ID of the device |
| `text` | body (application/json) | yes | string |  Text to speak on the device via phone call |
| `voice` | body (application/json) | no | string or null | default="female" Voice type: 'male' or 'female' |
| `language` | body (application/json) | no | string or null | default="en-US" Language code (e.g., 'en-US', 'es-MX') |
| `repeat` | body (application/json) | no | integer or null | default=1 Number of times to repeat the message |

```sh
curl --request POST https://devices.oceanfriendswebsocket.com/webhook/ttsSpeakCall \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "profile_id": "<PROFILE_ID>",
  "text": "Hello",
  "voice": "female",
  "language": "en-US",
  "repeat": 1
}'
```

### GET /healthz

Health check. Liveness is independent of downstream delivery. Readiness checks required Mongo/Redis connectivity; inspect worker heartbeats and backlogs separately.

Access: Public health/readiness

```sh
curl --request GET https://device-events.oceanfriendswebsocket.com/healthz
```

### GET /readyz

Health check. Liveness is independent of downstream delivery. Readiness checks required Mongo/Redis connectivity; inspect worker heartbeats and backlogs separately.

Access: Public health/readiness

```sh
curl --request GET https://device-events.oceanfriendswebsocket.com/readyz
```

### POST /pubsub

Internal service · not FE. Persist an explicit-profile NotificationOutbox intent → 202 → notification worker → Redis global PubSub → every realtime instance → bounded authorized socket queue.

Access: Internal bearer token

| Field | Location | Required | Type | Limits / meaning |
| --- | --- | --- | --- | --- |
| `target` | body (application/json) | no | string | default="profile"  |
| `profileId` | body (application/json) | yes | string | minLength=1  |
| `payload` | body (application/json) | yes | object |   |
| `eventId` | body (application/json) | no | string or null |   |

```sh
curl --request POST https://device-events.oceanfriendswebsocket.com/pubsub \
  --header 'Authorization: Bearer <INTERNAL_TOKEN>' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "target": "profile",
  "profileId": "<PROFILE_ID>",
  "payload": {},
  "eventId": "<EVENTID>"
}'
```

### WEBSOCKET /websocket/{profile_id}/{center_id}/{phonenumber}

FE · profile hints. Authenticate within 10s, authorize profile owner, receive resync_required, then Redis hints. Queue size 64; send timeout 5s; idle timeout 300s; refresh authentication after 55min. Every exit awaits sender cleanup.

Access: Public test mode enabled; secure-mode rule: Allowed Origin + first-message Firebase/operator token; profile-scoped subscription

```text
wss://device-events.oceanfriendswebsocket.com/websocket/<PROFILE_ID>/<CENTER_ID>/<DEVICE_E164>
```


## 12. Collections and bundle contents

Both legacy and v1 use **OceanFriendsDB**. Master `profiles` and device projection `DeviceProfiles` are separate collections in that database. Do not build new FE contracts around internal collection schemas.

| Collection | Status | Purpose |
| --- | --- | --- |
| `DeviceAttempts` | New | One record per frame attempt, sequence/session correlation, evidence and deadline. |
| `CommandSlots` | New | Per-device serialized command ownership. |
| `DeviceInbox` | New | Durable raw ingress processing, deduplication, replay evidence and dead letters. |
| `AnalyticsOutbox` | New | Retryable BigQuery deliveries with stable event IDs. |
| `NotificationOutbox` | New | Durable WebSocket hints and optional push deliveries. |
| `ProvisioningJobs` | New | Sequential configuration jobs, child commands and progress. |
| `ProvisioningSlots` | New | One active configuration workflow per profile. |
| `VerizonInbox` | New | Sanitized/deduplicated carrier callback inbox and reconciliation. |
| `ProviderTokens` | New | Shared Verizon OAuth/M2M cache, refresh lock and authentication cooldown. Contains sensitive tokens. |
| `TelnyxInbox` | New | Verified SMS/voice/status callbacks awaiting durable processing. |
| `ProviderActions` | New | Outbound provider-action ledger and uncertain outcomes. |
| `CallStates` | New | Cross-process call-control context and lifecycle state. |
| `VoiceCommands` | New | TTS-via-phone-call records; this specialized collection retains its separate voice statuses. |
| `SchemaMigrations` | New | Applied schema/migration markers. |
| `DeviceSetupEvents` | New · setup tooling | Audit record of the owned-device fresh setup and backup location. |
| `DeviceProfiles` | Reused | Canonical identity, assigned phone, current state, version and per-command summaries. |
| `DeviceCommands` | Reused | Shared new command ledger for both legacy and v1 APIs; schema-version-0 imported records remain quarantined here. |
| `DeviceInventory` | Reused | SIM/device inventory, lifecycle state and per-SIM pending operation. |
| `VerizonRequests` | Reused | Carrier lifecycle operation ledger, provider request IDs and reconciliation. |
| `DeviceAlerts` | Reused | Device alarm history. |
| `GeofenceTriggers` | Reused | Timestamped geofence transition history. |
| `DeviceInboundMessages` | Reused | Received device SMS history. |
| `CallHistory / CallEvents / CallFailures` | Reused | Existing voice history, events and failures. |
| `DeviceUnhandledResponses / WebhookErrors` | Reused | Unclassified responses and legacy diagnostic/error records. |
| `profiles` | Existing shared collection | Master profile/owner/center metadata; distinct from the device projection in DeviceProfiles. |

Redis holds device/session leases, transport streams, PubSub and carrier budget counters. BigQuery analytics are asynchronous; they are not the source for immediate FE state. ProviderTokens and provider-action data stay server-side.

### Files to share

| File | Purpose |
| --- | --- |
| OFDevice_Device_Handbook_v1.0.html | Standalone searchable handbook; inline styling/scripts; printable |
| OFDevice_Device_Handbook_v1.0.md | Editable Markdown copy with the same catalog and endpoint examples |
| examples/device-client.ts | Dependency-free TypeScript HTTP + WebSocket client and device view model |
| examples/use-device.ts | React hook showing refresh/poll/socket cleanup; install in your existing React app |
| examples/DevicePanel.tsx | Minimal React screen using the real test mapping, command intent and snapshot fields |
| examples/backend_client.py | Python standard-library BE client; explicit intent + command polling |
| postman/OFDevice_v1.0.postman_collection.json | Read requests, all command bodies, legacy/carrier/voice requests and labeled callback/internal templates |
| postman/OFDevice_Test.postman_environment.json | Test IDs/hosts plus blank token variables; choose a unique idempotency_key per new action |
| reference/commands.json | Full machine-readable command catalog |
| reference/endpoints.json | Endpoint samples/parameter metadata |
| reference/api.openapi.json | API schema snapshot; framework-declared input contracts |
| reference/realtime.openapi.json | Realtime HTTP schema; WebSocket contract is in Section 3 |
| README.md | Quick handoff instructions and version notes |

The bundle contains no runtime credentials and makes no network requests when opened. Importing Postman does not execute requests. Run individual examples intentionally; do not run the whole collection against the shared device because it includes settings, power actions and real-provider templates. The HTML may be printed using the browser's Print / Save as PDF action. Use the live `/openapi.json` and backend source to resolve changes after this dated release.
