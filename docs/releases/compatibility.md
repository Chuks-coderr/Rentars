# Compatibility Requirements for ABI and Schema Changes

## Why this matters

Rentars deploys three independently versioned components:

- **Backend** (`apps/backend`) — Express API, Bun runtime
- **Frontend** (`apps/web`) — Next.js, Yarn
- **Contracts** (`apps/contracts`) — Soroban/Stellar WASM, immutable after deployment
  unless explicitly declared upgradeable (see [Contract upgrade and migration policy](#contract-upgrade-and-migration-policy))

A database migration or contract ABI change can silently break a deployed client
or block a rollback.  Every such change therefore requires explicit release notes
and passing compatibility checks before merging.

## What triggers the check

The `check-abi-migration-changelog.sh` CI script fails a PR if any of the
following files change without a corresponding CHANGELOG.md update:

| Changed files | Category |
|---|---|
| `apps/contracts/*.json` | Contract ABI |
| `apps/backend/database/migrations/*.sql` | Database schema |
| `apps/contracts/upgrade-policy.json` | Contract upgradeability / authority |

## What to write in the changelog

Add an entry under `## [Unreleased]` with at least one of these headings:

### Additive changes (backwards-compatible)

These are safe to deploy without coordination.  Label them under `### Added`.

```markdown
### Added
- Migration 00025: add `cancelled_at` column to `bookings` (nullable, defaults
  to NULL for existing rows — backwards compatible with the deployed backend).
```

### Breaking changes (require coordinated rollout)

Label them under `### Changed` or `### Removed` and include:

1. **What changed** — the field, table, or contract entrypoint affected.
2. **Incompatible client versions** — which deployed version of backend/web/contract
   is no longer compatible after this change.
3. **Rollout order** — which component must be deployed first.
4. **Rollback boundary** — whether rolling back crosses this migration boundary
   (if so, a data migration or contract re-deploy may be required).

```markdown
### Changed
- Contract ABI: `create_booking` entrypoint now requires `escrow_agent` parameter.
  Incompatible with backend < 1.4.0.  Deploy backend 1.4.0 before this contract.
  Rolling back the contract is not possible post-deploy; rollback requires
  re-deploying the prior WASM from the release manifest.
```

## Contract upgrade and migration policy

Deployed Soroban contracts and their stored state may outlive application
versions.  The policy below makes upgradeability and upgrade authority explicit
so operators can always answer: *can this contract be upgraded, and who may
authorize it?*

### Upgradeability and authority

Every contract's upgradeability is declared in
`apps/contracts/upgrade-policy.json`.  Each entry records:

- `contract` — contract name / WASM artifact
- `upgradeable` — `true` or `false` (immutable)
- `admin` — the address (or multisig) authorized to upgrade, or `null` when
  immutable
- `timelock_seconds` — delay between upgrade proposal and execution (`0` when
  no timelock applies)
- `migration_entrypoint` — the contract entrypoint that migrates stored state
  between versions, or `null` when no migration is supported
- `storage_version` — the current on-chain storage schema version

Immutable contracts must set `upgradeable: false`, `admin: null`, and
`migration_entrypoint: null`.  Any change to this file is a breaking change and
must be documented in the changelog (see below).

### Client compatibility check

Before release, run the compatibility check against the release manifest and
the current client version assumptions:

```bash
bun run scripts/check-contract-compatibility.ts --manifest docs/releases/manifest-<version>.json
```

The script compares the recorded `contracts.abi_sha256` values and
`storage_version` entries against the ABIs and versions the backend and web
clients were built against.  It exits non-zero when a deployed client is
incompatible with the contracts in the manifest, so incompatibility is detected
before release rather than after deployment.  This check runs in CI on every
release PR and must pass before the manifest is published.

### Required changelog content for upgrades and migrations

Contract upgrades and state migrations are breaking changes.  In addition to
the standard fields above, the changelog entry must state:

1. **Upgradeability** — whether the contract is upgradeable and, if so, the
   `admin` and `timelock_seconds` from `upgrade-policy.json`.
2. **State migration** — the `migration_entrypoint` invoked, the
   `storage_version` transition (e.g. `v1 → v2`), and confirmation that the
   migration was tested against old state in a sandbox.
3. **Rollback / forward-fix plan** — whether rollback is possible (immutable
   contracts cannot roll back) and, if not, the forward-fix procedure.

```markdown
### Changed
- Contract upgrade: `rent_escrow` upgraded from storage `v1` to `v2`.
  Upgradeable; admin `G...` with a 48h timelock.  State migrated via
  `migrate_v2`, tested against a v1 sandbox snapshot.  Rollback is not
  supported after `migrate_v2` runs; forward-fix only.
```

## Release manifest

Every release generates `docs/releases/manifest-<version>.json` via:

```bash
bun run release:manifest --version X.Y.Z
```

The manifest records:
- `release.commit` — exact git SHA
- `database.latest_migration` — last applied migration filename
- `contracts.abi_sha256` — SHA-256 of each ABI file at release time
- `components.*` — per-component version

Use these fields during rollback to identify a compatible set.  See
`docs/releases/rollback.md` for the full procedure.

## Suppressing the check

If the ABI or migration change is already covered by a prior changelog entry
(e.g., a stacked PR), set the `BASE_SHA` environment variable to the commit
that already includes the changelog update:

```bash
BASE_SHA=<prior-commit> bash scripts/check-abi-migration-changelog.sh
```

Do not bypass the check with `git commit --no-verify`.
