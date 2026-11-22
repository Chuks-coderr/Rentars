# Rollback Runbook

This runbook covers how a Rentars release is judged healthy, who decides when it is paused or rolled back, what cannot be undone, and how the rollback itself is rehearsed. The numeric thresholds live in [`health-gates.json`](./health-gates.json); [DEPLOYMENT.md](../../DEPLOYMENT.md) section 7.6 describes them in prose.

## Ownership

| Decision | Owner | Notes |
|----------|-------|-------|
| Pause a rollout | On-call engineer | Automatic when a `pause` gate breaches. Resumed only after the gate's recovery windows hold. |
| Roll back a release | Release manager | Automatic for `rollback` gates, with notification. Manual rollbacks in production need two approvers. |
| Re-deploy after a rollback | Release manager with the on-call engineer | Requires investigation notes for any `rollback` gate. Those gates have no automatic recovery. |
| Notify finance | Release manager | Transaction and contract gates (`transaction_failure_ratio`, `contract_invocation_failure_ratio`) affect funds. |

The owners above are also recorded in `health-gates.json` under `ownership`.

## Health gates

Every rollout is checked against the gates in `health-gates.json`. The evaluator is `scripts/release-health-gate.ts`:

```bash
# Is the release healthy right now? Input is one value per one-minute window, oldest first.
bun run scripts/release-health-gate.ts --metrics metrics-snapshot.json

# May a paused rollout continue? Each gate with a recovery rule must have held for its windows.
bun run scripts/release-health-gate.ts --metrics metrics-snapshot.json --mode resume
```

Snapshot format:

```json
{
  "error_rate": [0.01, 0.06, 0.08],
  "latency_p99_ms": [240, 310, 290],
  "transaction_failure_ratio": [0, 0],
  "db_pool_usage_ratio": [0.42],
  "redis_evictions_per_second": [0],
  "contract_invocation_failure_ratio": [0, 0]
}
```

The pipeline acts on the exit code:

| Exit code | Decision | Action |
|-----------|----------|--------|
| `0` | `proceed` | Continue the rollout |
| `10` | `pause` | Hold canary traffic at its current level and alert on-call |
| `20` | `rollback` | Start the rollback procedure below and notify the release manager and finance |
| `1` | error | The policy or snapshot is invalid. Treat as a failed gate: hold the rollout. |

The evaluator fails closed. A gate with too little data pauses the rollout, so a metrics outage stops a release instead of letting it proceed. The metrics snapshot is produced by the deployment platform's monitoring; the repository does not ship a collector.

The JSON policy is authoritative. Each evaluation output includes `policy_sha256`, which identifies the exact policy in force. The release manifest records the same hash under `rollback.health_gate_policy`.

## Selecting a prior release manifest

Release manifests are stored at `docs/releases/manifest-<version>.json`. Each manifest records the exact commit, Docker image tag, migration checkpoint, ABI hashes, contract IDs and Stellar network that were live at release time, and the health-gate policy hash.

```bash
# List all manifests newest-first
ls -1t docs/releases/manifest-*.json

# Inspect a specific release
cat docs/releases/manifest-1.2.3.json | jq .
```

`rollback.previous_manifest` in the current manifest names the newest other release, which is the default rollback target. Confirm it is known-good before using it.

## Rollback procedure

1. **Identify the target manifest.** Choose the last known-good version. Use `rollback.previous_manifest` as the default.
2. **Check irreversible operations.** Before any rollback, confirm the change list in [Irreversible operations](#irreversible-operations). If the current release applied an irreversible migration or moved a contract, the rollback plan is different.
3. **Check database compatibility.** Compare `database.latest_migration` and `database.migration_count` in the target manifest with the current head. If the current head is ahead, a database downgrade is required (see migration runbooks in `docs/admin-runbooks.md`).
4. **Re-deploy the Docker image.** Use `release.image_tag` from the target manifest to pull and re-deploy the backend and frontend images.
5. **Verify dependencies.** Confirm the deployed frontend and backend point at the contract IDs in `dependencies.contract_ids` and the network in `dependencies.stellar_network`. If `components.contracts.version` differs, ABI SHA-256 hashes in `contracts.abi_sha256` confirm the correct ABI files are loaded.
6. **Clear caches.** Flush feature-flag and configuration caches as described in DEPLOYMENT.md section 7.8. Do not flush user session caches.
7. **Smoke test.** After deployment, run the preview smoke suite:
   ```bash
   SMOKE_API_URL=https://api.your-env.example.com \
   SMOKE_FRONTEND_URL=https://your-env.example.com \
   yarn workspace web playwright test e2e/smoke.spec.ts
   ```
   Core reads (listings, booking lookup) and reconciliation must recover before the rollback is closed.
8. **Update the incident record.** Note the rollback in the incident tracker with the manifest version, the gate that triggered it, and the reason.

## Contract rollback and forward-fix

Soroban contracts are not rolled back by re-deploying an older Docker image. The backend image and the on-chain contract version move independently, and a contract upgrade may already have migrated stored state. The upgrade policy (which contracts are upgradeable, who may authorize an upgrade, and how state and clients migrate) is in [`docs/contracts/upgrade-policy.md`](../contracts/upgrade-policy.md). This section covers the rollback side of that policy.

### Identifying whether a contract was upgraded

1. **Compare manifests.** `components.contracts.version` and `contracts.abi_sha256` in the current manifest must match the target manifest. A differing version or ABI hash means a contract changed between the two releases.
2. **Compare contract IDs.** If `dependencies.contract_ids` differ between manifests, the release deployed a new contract address rather than upgrading in place. That is the "Stellar contract redeploy" row in [Irreversible operations](#irreversible-operations), not an upgrade.
3. **Confirm on-chain.** Read the deployed contract's version and admin from the network recorded in `dependencies.stellar_network`:
   ```bash
   stellar contract invoke --id "$PROPERTY_LISTING_CONTRACT_ID" --network "$STELLAR_NETWORK" -- version
   stellar contract invoke --id "$PROPERTY_LISTING_CONTRACT_ID" --network "$STELLAR_NETWORK" -- admin
   ```
   The reported version must match the manifest. If it does not, the manifest is stale: stop and reconcile before continuing.
4. **Check the upgrade record.** Every authorized upgrade is logged with the approving admin, the timelock expiry, the old and new WASM hashes, and the migration entry point that ran. The record is attached to the release ticket.

### Restoring the prior WASM

Only contracts marked upgradeable in the upgrade policy can be restored. Immutable contracts cannot be changed; the only remedy is a new deployment and a client repoint.

1. **Locate the prior WASM.** The release manifest records `contracts.wasm_sha256` for each contract. The matching WASM artifact is attached to the prior release and stored alongside the manifest. Verify the hash before use:
   ```bash
   sha256sum property-listing.wasm
   ```
2. **Confirm the admin and timelock.** The restore is an upgrade and follows the same authorization as a forward upgrade: the contract admin submits it, and the timelock must elapse before it takes effect. Emergency restores still require the admin key; there is no bypass.
3. **Check state compatibility first.** A WASM restore does not revert storage. If the forward upgrade ran a state migration, the older WASM may not read the migrated state. Consult the migration notes in the upgrade policy:
   - **Additive migration** (new keys, old keys untouched): the prior WASM is safe to restore.
   - **Destructive migration** (keys rewritten or removed): the prior WASM is not safe. Use a forward-fix instead.
4. **Submit the restore** through the same upgrade entry point used for the forward upgrade, pointing at the prior WASM hash. Record the restore in the upgrade log.
5. **Verify.** Re-run the version and admin reads above, confirm the reported version matches the target manifest, and run the smoke suite. Cross-contract calls must succeed before the rollback is closed.

### Irreversible state migrations

A contract state migration that rewrites or removes keys cannot be undone by restoring the prior WASM. Treat it like a data-rewriting database migration:

| Migration kind | Rollback path |
|----------------|---------------|
| Additive (new keys only) | Restore the prior WASM; old keys are still readable. |
| Dual-write (old and new keys written) | Restore the prior WASM, then run the documented cleanup entry point once the old version is stable. |
| Destructive (keys rewritten or removed) | No WASM restore. Ship a forward-fix contract that reads the migrated state, or restore from a ledger snapshot taken before the upgrade. |

Before shipping any destructive migration, take a ledger snapshot or export of the affected contract storage and attach it to the release ticket. The forward-fix contract must be built, tested against a copy of the migrated state in a sandbox, and reviewed by the contract admin before submission.

If neither a forward-fix nor a snapshot restore is available, escalate to the release manager and the contract admin. Do not attempt to replay on-chain transactions to compensate; reconcile against the ledger instead (see [Irreversible operations](#irreversible-operations)).

## Irreversible operations

These steps cannot be undone by re-deploying an older image. Each one needs an explicit plan before the release that contains it ships.

| Operation | Why it is irreversible | Required before shipping |
|-----------|------------------------|--------------------------|
| `DROP COLUMN` or `DROP TABLE` in a migration | Data is removed from the database | Deprecate in one release, remove in a later one (DEPLOYMENT.md section 7.7) |
| Data-rewriting migrations (type changes, backfills) | Old rows are transformed in place | Dual-write or new column, and a restore point from the backup procedure |
| Stellar contract redeploy | Contracts are immutable; the new contract has a new address | Repoint `PROPERTY_LISTING_CONTRACT_ID`, `BOOKING_CONTRACT_ID`, and `REVIEW_CONTRACT_ID`, then verify cross-contract calls (DEPLOYMENT.md section 7.3) |
| Submitted on-chain transactions (escrow release, refunds) | Ledger writes are final | Reconcile against the ledger after rollback; never replay blindly |
| Email or notification sends | Messages cannot be recalled | Gate behind feature flags where possible |

Schema-level compatibility rules are in DEPLOYMENT.md section 7.7.

## Staging rollback rehearsal

A rollback that has not been rehearsed is not a plan. Before each production release that changes the database schema or a contract, rehearse the rollback in staging and keep the record:

1. Deploy the candidate release to staging and let the manifest be generated (`scripts/generate-release-manifest.ts`).
2. Deploy the previous manifest's image tag to staging, then roll forward to the candidate again.
3. Replay the rollback with the gate evaluator. Feed it a `rollback` snapshot (for example a sustained `transaction_failure_ratio` above 0.02) and confirm the decision is `rollback` with exit code `20`.
4. Run the smoke suite after rollback. Confirm that listings, booking reads, and reconciliation recover.
5. Save the rehearsal record: both manifest files, the gate output, and the smoke results. Attach it to the release ticket.

Record the rehearsal date and the staging result in the release ticket before the production rollout starts.
