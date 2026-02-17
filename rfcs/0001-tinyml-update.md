# RFC 0001: TinyML on-device model update & rollback

Summary
- Propose an on-device model update and rollback flow for TinyML models used in our embedded clients. The design covers update packaging, secure delivery, atomic activation, rollback, and validation.

Motivation
- Devices run models locally; we need a safe, auditable, and reversible way to ship model updates to fix bugs or improve accuracy without bricking devices or degrading performance.

Goals
- Securely deliver model updates to devices.
- Ensure atomic swap/activation of new models to avoid partial-state exposures.
- Provide a reliable rollback mechanism if the new model misbehaves.
- Minimize device downtime and bandwidth usage.

Non-Goals
- Full OTA firmware update system (out of scope).
- Distributed model training or federated aggregation.

Design
1) Packaging
   - Model bundle: { model.bin, metadata.json, signature }.
   - metadata.json: version, checksum, compat_matrix (min-fw, max-fw), rollout flags, required resources.
   - Sign bundles using repo-private signing key; devices verify before applying.

2) Delivery
   - Updates hosted on CDN/secure storage. Device polls update service or receives push notifications via existing control channel.
   - Device downloads to a temporary location and verifies signature and checksum.

3) Atomic activation
   - Use double-buffered model storage: active_model and staging_model.
   - After validation, move staging -> inactive slot, update pointer atomically (e.g., write new symlink or atomic metadata transaction).
   - Ensure fallback remains in place until device confirms successful inference/health checks with the new model.

4) Rollback strategy
   - Health-check window: after activation, run a suite of lightweight smoke inferences and performance checks for N minutes or M inferences.
   - If health checks fail or the device crashes, auto-roll back to previous model from inactive slot.
   - Maintain last-known-good model(s) and up to K previous versions on device depending on storage.

5) Telemetry & Metrics
   - Devices send anonymized success/failure telemetry (model_version, errors, perf metrics) to backend with rate-limiting and privacy safeguards.
   - Rollout controller aggregates signals and can pause/rollback staged rollouts server-side.

6) Rollout process
   - Canary rollout: target a small subset of devices (e.g., 1%) first, monitor metrics, then progressively increase.
   - Support staged rollouts based on device attributes (region, fw version, hardware capability).

Testing
- Unit tests for bundle validation and signature verification.
- Integration tests for download, staging, activation, and rollback paths using a device emulator.
- End-to-end staged rollout tests in a pre-production fleet.

Security & Privacy
- All bundles signed; devices reject unsigned or tampered bundles.
- Telemetry is limited and respects user privacy; no raw input data leaves device.
- Keys stored in secure keystore on devices; use hardware-backed key storage where available.

Implementation Plan
- Phase 0: Define bundle format, signing, and metadata schema (2 weeks).
- Phase 1: Implement staging and atomic activation on device (3 weeks).
- Phase 2: Implement server-side rollout controller and telemetry ingestion (3 weeks).
- Phase 3: Run canary rollouts and iterate on health checks (2 weeks).

Open Questions
- How many rollback versions should devices retain by default? (storage tradeoff)
- Do we require a signed manifest of allowed model versions per device cohort?

Alternatives Considered
- In-place replace without double-buffering (rejected due to risk of partial update).
- Pull-only updates vs push notifications (we support both; start with pull to simplify infra).

Appendix: Example metadata.json
{
  "version": "1.0.3",
  "checksum": "sha256:...",
  "min_fw": "1.2.0",
  "max_fw": "2.0.x",
  "rollout": { "phase": "canary", "percent": 1 },
  "signature": "..."
}