# OFDevice · Device integration handoff v1.0

Open **OFDevice_Device_Handbook_v1.0.html** in any current browser. It is a self-contained file: no server, login, font download or JavaScript dependency is needed to read it. It makes no network requests. Use the search box for command/endpoint payloads, Copy for snippets and Print / PDF for a printable copy. Share the HTML alone for reading, or this entire ZIP for the examples and Postman collection. The guide includes real test identifiers supplied by the owner; it is a colleague handoff, not a credential file.

## FE quick start

1. Read handbook Sections 1–5 and the acceptance checklist in Section 9.
2. Add examples/device-client.ts to your TypeScript app. It has no package dependencies and expects a current browser (fetch, WebSocket, crypto.randomUUID). React users may also copy examples/use-device.ts and examples/DevicePanel.tsx (React 18+, TypeScript with DOM types) and render <DevicePanel /> in an existing app.
3. Construct one memoized DeviceApi instance, use the TEST_DEVICE mapping and publicTestMode=true for the current test environment. For secure mode, supply the same memoized Firebase ID-token supplier to the HTTP client and socket hook. Do not include a server operator token.
4. Store each CommandIntent across a failed HTTP request; use a new intent only for a deliberate new action. Track the accepted commandId until terminal. Guard double clicks with a submitting flag. The hook and client never auto-submit on page load.
5. A 1008 socket close stops automatic reconnect; correct/refresh credentials or subscription inputs and remount/restart. Network loss/1013 triggers jittered reconnect and HTTP resync. Keep source objects/functions stable to avoid reopening sockets on every React render.

## BE quick start

Read Sections 6–8 and 12. examples/backend_client.py uses only the Python standard library:

```sh
python3 examples/backend_client.py
# Optional deliberate TCP battery read:
python3 examples/backend_client.py --battery
```

The default command reads the owned test profile. Importing the module makes no requests. For privileged secure-mode work, set OFDEVICE_OPERATOR_TOKEN in the server process; never share the value. The client does not automatically retry writes. Persist critical intent/key before submission so the same action can be recovered after a restart. No framework or full application is bundled.

## Postman

Import both JSON files under postman/. Choose the OFDevice public test environment. Run individual read requests first. Set idempotency_key to a fresh UUID for each deliberate new command, configure job or lifecycle action; keep it unchanged for the same HTTP retry. Different commands cannot reuse one key. Public mode needs no Authorization. For secure mode select request/folder Bearer Token with the appropriate Firebase token or backend-only operator token. Provider callback signatures and internal tokens still apply.

Do not run the entire collection as an automated test: it includes power/settings actions, SMS/voice/carrier operations and templates. Unresolved firmware/provider/new-enrollment placeholders are intentionally not usable as real configuration. The optional raw adapter is not deployed and is not included. Callback examples explain payload shape; they are not a way for FE to impersonate a provider. POST /api/webrtc/token depends on separate Telnyx credential setup and is not verified here.

## Scope and snapshot

Version 1.0, 4 November 2026. Based on mounted backend OpenAPI, current command builders, deployment records and existing integration evidence. This build does not send device/provider actions, deploy services or claim fresh physical-carrier tests. Current runtime is public for the owner's testing. The owned emulator writes simulated telemetry at 47.4753678, -121.7363316 and keeps its data when stopped. ICCID is unknown, full version read may fail, custom TCP DND schedules have a known parsing issue, and live Verizon/SMS/voice/push checks remain separate. See the handbook for details and ownership setup before restoring secure FE access.

reference/api.openapi.json is the complete API input-schema snapshot; some legacy response models are unconstrained. reference/endpoints.json adds input placement/limits and maintained raw-handler examples. Framework-generated docs and the optional raw adapter are omitted from its 103-entry integration inventory. There are 99 API HTTP operations, 3 realtime HTTP operations and 1 profile WebSocket. There are 44 command catalog entries including families and aliases.

All bundle files are explicitly allowlisted. No .env, service-account file, Mongo connection string, AWS credentials, runtime secret or backup is included. MANIFEST.sha256 records content hashes. To rebuild inside the OFDevice repository: .venv/bin/python -m scripts.build_device_handbook.
