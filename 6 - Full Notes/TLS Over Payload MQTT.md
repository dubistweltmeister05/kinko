[[Titan]]
# Payload MQTT TLS Migration Summary

## Objective
Enable TLS for Payload MQTT using certificates uploaded from the mobile app over BLE.

Incoming files:
- ca.crt
- client.crt
- client.key

All incoming certificate data is PEM text and arrives in BLE chunks.

## Discussion Summary
### 1. BLE Certificate Intake
- BLE data is chunked and must be reconstructed on device.
- Current debug work proved that the chunk reception is flawed and needs to be looked into.


### 2. The Target of Incoming certificates - Payload Server
- Payload path currently has no TLS and is now the priority integration target.

### 3. SPIFFS Storage Decision
- Existing partition layout already includes a 4 MB SPIFFS partition named torbuffer.
- Current buffer logic reserves about 3.0 MB for payload buffering, leaving practical headroom for cert files.
- Decision: no new partition required for rollout 1.
- Planned cert location: /buffer/certs

### 4. Activation Model
- Uploaded cert files are staged first.
- Explicit commit command from app will activate new cert set.
- Commit must validate all three files before activation.

### 5. TLS Fallback Requirement
To avoid service outage when TLS fails in field, fallback policy was defined:
- PAYLOAD_TLS_DISABLED: non-TLS only.
- PAYLOAD_TLS_PREFERRED: try TLS first; downgrade to non-TLS on TLS failure.
- PAYLOAD_TLS_REQUIRED: TLS only; no plaintext downgrade.

Recommended default for rollout 1: PAYLOAD_TLS_PREFERRED.

## Proposed Implementation Plan
## Phase 1: Scope and Safety Baseline
- Implement TLS enablement in Payload MQTT connection path.
- Keep current non-TLS behavior available as fallback.

## Phase 2: SPIFFS Cert Staging
- Save incoming files under:
  - /buffer/certs/ca.crt
  - /buffer/certs/client.crt
  - /buffer/certs/client.key
- Write strategy:
  - write to temporary file
  - validate PEM envelope
  - flush and sync
  - atomic rename

## Phase 3: Commit and Activation
- Add commit command in existing command flow.
- On commit:
  - verify presence and validity of all files
  - mark cert set active
  - trigger Payload MQTT reconnect

## Phase 4: Payload MQTT TLS Wiring
- Build mqtt client config with SPIFFS-loaded CA, client cert, and client key when TLS mode is active.
- If TLS setup fails and mode is preferred, fallback to non-TLS path.
- Log source and mode on each connection attempt.

## Phase 5: Reliability Controls
- Add mutex around cert read/write and reconnect crossover.
- Enforce file size limits.
- Ignore stale temp files at boot.
- Keep diagnostic error reporting explicit.

## Phase 6: Validation
- Happy path: upload 3 files, commit, reconnect with TLS.
- Failure path: malformed or missing certs, interrupted write, oversize files.
- Fallback path: verify downgrade in preferred mode.
- Persistence path: reboot and verify expected mode and cert source.

## Risks and Mitigations
### Risk: TLS handshake failures in field
Mitigation: preferred-mode downgrade with strong diagnostics.

### Risk: SPIFFS space contention with payload buffer
Mitigation: enforce free-space threshold before accepting cert writes.

### Risk: Partial writes/power loss during upload
Mitigation: temp files plus atomic rename and commit marker written last.

### Risk: Runtime race between upload and reconnect
Mitigation: mutex around cert lifecycle operations.

## Key Decisions Confirmed
- Payload MQTT is the first TLS target.
- Existing SPIFFS partition is sufficient for initial rollout.
- Cert activation requires explicit commit command.
- Rollout default mode is TLS preferred with non-TLS fallback.

## Open Items for Senior Approval
1. Should broker URI be auto-upgraded from mqtt:// to mqtts:// when TLS mode is active?
2. Should non-TLS fallback remain enabled in production, or only in pilot builds?
3. Should a future phase isolate cert storage into a dedicated partition for stronger separation?

## Suggested Rollout
1. Pilot firmware with TLS preferred mode and diagnostics enabled.
2. Observe handshake/fallback telemetry across field devices.
3. Move selected fleets to TLS required mode after soak period.
