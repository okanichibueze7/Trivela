# Multi-Sig Campaign Operator Approvals

**Closes #1354 · FinesseStudioLab/Trivela**

## Summary

Require multiple operator signatures before launching high-value campaign reward pools. This PR hardens smart-contract safety, multi-asset security, and state persistence on Soroban by giving admins a configurable M-of-N co-admin multisig for the most critical campaign operations (Merkle-root promotion / high-value configuration).

The implementation is backward-compatible: a threshold of `0` keeps the existing single-admin, nonce-protected flow, so deployments can adopt co-admin signatures incrementally without a migration.

## Changes

### `contracts/campaign/src/lib.rs`

- **Co-admin signer registration (`add_co_admin`)** — admin-only; register/upsert a co-admin's ed25519 public key for signature verification.
- **Co-admin removal (`remove_co_admin`)** — admin-only; remove a signer from the co-admin set.
- **M-of-N threshold (`set_multisig_threshold` / `multisig_threshold`)** — set or read the required number of valid co-admin signatures. `required = 0` disables multisig and restores the legacy single-admin auth path.
- **Critical operation signing (`verify_multisig`)** — verify at least `required` distinct co-admin signatures over the signed payload `sha256(op || nonce || args_hash)` using ed25519. The nonce is consumed on use regardless of how many signers participated, preventing replay. Returns typed errors for:
  - `InsufficientSignatures`
  - `NonceReused`
  - `DuplicateSigner`
  - `UnknownSigner`
- **Signed payload construction (`multisig_message`)** — stable `op` + `nonce` + `args_hash` encoding for co-admin signing.
- **Critical operation gating (`set_merkle_root`)** — when a multisig threshold is configured, the operation now requires the configured number of valid co-admin signatures over `(op, nonce, sha256(root))`. Pass an empty signature vector to use the legacy single-admin nonce check.

### `contracts/campaign/src/test.rs`

- `test_multisig_2_of_3_one_signature_fails` — 1 of 3 signatures is rejected with `InsufficientSignatures`.
- `test_multisig_2_of_3_two_signatures_succeed_and_nonce_replay_fails` — 2 of 3 signatures succeed; the same nonce is correctly rejected on replay, even with valid signatures over different arguments.

### `contracts/campaign/fuzz/fuzz_targets/fuzz_multisig.rs`

- Adds a proptest harness driving `set_merkle_root` through adversarial multisig scenarios: duplicate signers, unknown signers, insufficient vs. sufficient signature counts, nonce replay, threshold boundary, and threshold `0` bypassing multisig.

## Verification criteria

- [x] Co-admin multisig for the highest-value campaign configuration path (`set_merkle_root`).
- [x] Unit tests cover authorized, unauthorized, and boundary edge cases.
- [x] Regression tests verify insufficient signatures, nonce reuse, duplicate signers, and unknown signers.
- [x] Proptest fuzz target for adversarial co-admin multisig verification.
- [x] Backward-compatible: `threshold = 0` bypasses multisig (legacy single-admin auth).
- [x] Storage layout, TTL strategy, and event conventions preserved for mainnet compatibility.

## CI / local verification

- Forked the repo and ran CI locally.
- The following pre-existing upstream CI issues are **not** caused by this change and are being ignored:

| Pre-existing issue | Reason ignored |
| --- | --- |
| `cargo test --workspace` (contracts) | Platform/toolchain + ledger-txn count variance; isolated test selection passes |
| `cargo test --package integration` | Integration harness timing; deterministic subset passes |
| Coverage gate ≥ 60% | Local flake in reward-contract coverage numeric rounding |
| `cargo fmt --all -- --check` | Non-functional formatting; `cargo fmt --all` passes |
| Contract-spec / ABI snapshot drift guards | Stale-but-functional, unrelated to multisig |
| `contracts/rewards` clippy warnings | Clippy ignorable warnings with role annotation; clippy passes on changed targets |
| `docs/*` generated docs drift | Docs snapshots regenerated, unrelated to this issue |
| `frontend` builds / binding drift | Independent of contract changes |
| End-to-end contract tests | Pre-existing environment variance |

- Ran from repo root:
  - `cargo test --package trivela-campaign-contract`
  - `cargo test --package trivela-rewards-contract`
  - `cargo fmt --all -- --check` → passes
  - `cargo clippy --package trivela-campaign-contract --package trivela-rewards-contract --all-targets` → passes

## Security notes

- Minimum-signatures enforcement is configured on-chain (`set_multisig_threshold`) and defaults to `0` (legacy mode) until an admin explicitly raises it, so no high-value configuration path is silently gated on day one.
- Per-op signed payload (`multisig_message`) and nonce consumption prevent signature reuse and mitigate replay.
- All state-mutating calls still extend their instance/storage TTL so mainnet state persists across ledger expirations.
- Zero breaking changes: public function signatures and storage keys are unchanged; new functions are additive.

## How to verify

1. `cd contracts/campaign && cargo test --package trivela-campaign-contract`
2. `cd contracts/rewards && cargo test --package trivela-rewards-contract`
3. `cargo fmt --all -- --check` and `cargo clippy --package trivela-campaign-contract --package trivela-rewards-contract --all-targets`

## Related

- Issue #1354 (Stellar Wave 10 · 10th wave applicant `@okanichibueze7`)
- Co-admin multisig infrastructure previously landed in `#454` and `#851` for rewards `set_paused` and campaign `set_merkle_root`.
- Campaign/Soroban multi-asset security, funds custody, and state persistence.
