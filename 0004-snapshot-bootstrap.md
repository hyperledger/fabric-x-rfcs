---
layout: default
title: Snapshot Bootstrap
nav_order: 3
---

<!--
Copyright IBM Corp. All Rights Reserved.

SPDX-License-Identifier: Apache-2.0
-->

- Feature Name: snapshot_bootstrap
- Start Date: 2026-11-11
- RFC PR:
- Fabric-X Component: fabric-x-committer (database CLI, sidecar, coordinator, validator-committer, snapshot hasher), fabric-x-common
- Fabric-X Issue:

> Draft proposal. The RFC number and submission links are unassigned.

# Summary
[summary]: #summary

This RFC lets a new Fabric-X committer start from a database backup instead of replaying the entire channel history.

The source exports a checkpointed database clone together with block `S`, which contains snapshot transaction `T`. The target imports the database, verifies `S`, and uses existing recovery to find the saved outcomes of the other transactions. It then processes original `T` and starts fetching blocks from `S+1`.

Processing `T` creates the target's hashing clone. The target compares its hash with the hash in a later checkpoint transaction `C`. A match confirms the restored state's integrity under the channel's trust model; a mismatch halts processing.

This RFC builds on [RFC 0004 — Database Snapshot and Checkpoint][snapshot-rfc]. Online pruning and general block-history archives are separate future work.

# Motivation
[motivation]: #motivation

New organizations and replacement deployments need a way to restore committed state without downloading every earlier block. That state includes values, MVCC versions, namespace policies, channel configuration and transaction statuses used to detect duplicate TX IDs.

A database backup alone is not enough. The target also needs to know which snapshot transaction to process and how to verify the next block. Full signed block `S` supplies that information.

Future block pruning depends on this recovery path. **Pruning may be enabled only when a durable, restorable database backup is available for the selected checkpointed snapshot.** Its full block `S` must also be backed up. A local clone or completed hash alone is not enough.

Export copies only the selected database clone. The target uses its own signing identity, TLS credentials and connection settings. Source database roles, account passwords and unrelated databases are not copied.

This is startup of a fresh deployment, not automatic repair or rollback of a running database.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## The flow at a glance

1. **Export:** package the source's checkpointed database clone and full block `S`.
2. **Import:** restore into a new target database and clear local snapshot bookkeeping.
3. **Start:** verify `S`, recover saved outcomes, and process original snapshot transaction `T`.
4. **Continue:** fetch `S+1` onward and compare the target's clone hash at checkpoint `C`.

Here, `S` is the snapshot-create block, `T` is its accepted snapshot transaction, and `C` is the later checkpoint transaction.

## Why full block `S` is needed

Block `S` serves four purposes:

- **Identify the snapshot transaction.** Its data contains original `T`, including TX ID, request and position. The target processes that transaction, not a newly submitted request.
- **Verify the block's contents.** Its data hash and orderer signatures let the target check that the transaction bytes and header have not been changed.
- **Check the next block.** Block `S+1` must refer to the hash of `S`'s header through its `PreviousHash` field.
- **Recover transaction outcomes.** Its transaction list gives existing recovery the IDs and positions whose statuses are already in the restored database.

Block verification checks `S`, not every database value. The later comparison at `C` verifies the target's restored state. The operator must still trust the archive producer and historical channel configuration.

## Step 1: Export the source snapshot

Suppose `T` is in block 100 and its checkpoint is in block 120. Export the checkpointed clone:

```bash
committer db export-snapshot \
  --config export.yaml \
  --output snapshot-100/
```

Default selection is the latest `CHECKPOINTED` snapshot with an available clone. To select another, use either `--tx-id <snapshot-tx-id>` or `--name <recorded-clone-name>`, not both.

```text
snapshot-100/
├── manifest.json
├── database/
│   └── <PostgreSQL archive or YugabyteDB native backup>
└── boundary-block.pb
```

The source saves full `S` when it processes the snapshot block. Configure its sidecar with:

```yaml
# Under source sidecar configuration; proposed field.
snapshot-block-backup-directory: /var/lib/committer/snapshot-blocks
```

Keep this block backup while the snapshot is offered for export. Export can then work even if orderers or ordinary sidecar history no longer retain `S`.

## Step 2: Import into a fresh target

**Start only the target PostgreSQL server or YugabyteDB cluster. Do not start any committer service yet.** Run:

```bash
committer db import-snapshot \
  --config import.yaml \
  --input snapshot-100/
```

After the full restore succeeds, importer makes three changes in one SQL transaction:

1. Empty local `ns__snapshot` rows, keeping the schema.
2. Set the latest-snapshot pointer value to SQL NULL, keeping its metadata row.
3. Set the saved last-committed block number to 99 (`S-1`).

Everything else stays unchanged, including transaction statuses and MVCC versions. The source clone and archive are not changed.

## Step 3: Start and process `S`

Copy `boundary-block.pb` to the sidecar host and configure:

```yaml
# Under target sidecar configuration; proposed field.
bootstrap-snapshot-block-path: /var/lib/committer/bootstrap/boundary-block.pb

orderer:
  latest-known-config-block-path: /etc/committer/latest-config.block
```

Then start committer services. Genesis-started deployments omit the bootstrap path.

The current connectivity config supplies orderer endpoints and transport trust. The historical config in the restored database supplies the policy for verifying `S`. A later connectivity config may include a new organization; it does not replace the historical verification policy.

The source processed all other transactions in `S` before creating the clone, including transactions placed after `T` in the block. Their effects and statuses are already restored. Original `T` has not committed yet.

Reuse sidecar recovery: `appendMissingBlock` in `service/sidecar/sidecar.go` maps the block, calls `GetTransactionsStatus`, and fills results with `fillStatuses`. `mapBlock` in `service/sidecar/mapping.go` keeps accepted snapshot `T` separate from regular transactions. Bootstrap reuses that status lookup, but does not append `S`.

Sidecar processes only original `T`. This creates the target hashing clone and local snapshot record. It then records that `S` is complete. Regular writes are not executed again, and `S` is not published as ordinary local history.

## Step 4: Continue and verify the state

After `S` is complete, sidecar enables public query/delivery/notifications and fetches `S+1` onward. Hashing runs alongside normal processing.

At `C`, existing snapshot handling waits/retries if the local hash is not ready, accepts a matching hash, and halts on mismatch. There is no extra public-service gate waiting for `C`. Results served before that comparison have not yet been verified by the local snapshot hash; a later halt cannot retract them.

Ordinary local history begins at `S+1`. Older block requests return unavailable; `Oldest` delivery starts at `S+1`. Clients needing earlier blocks use an archive service.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Export: select and protect the clone

Require one matching `CHECKPOINTED` record, an existing completed clone with a computed hash, and full `S` matching original `T`. Reject corrupt records, deleted clones and missing blocks. An aborted snapshot is not eligible.

**Use a reference in the snapshot record instead of holding a lock throughout the backup.** Add a proposed local clone-operation reference containing a unique operation ID and its kind: `EXPORT` or `DELETE`. The reference does not change snapshot lifecycle status or hash. Clone deletion still allows only `CHECKPOINTED` or `ABORTED` snapshots.

1. Export atomically checks that the clone still matches the record and no operation has claimed it. It records its `EXPORT` reference before accessing the clone.
2. Clone deletion must atomically acquire a `DELETE` reference before dropping the database. It cannot proceed while an export reference exists; export cannot proceed while deletion owns the record. Both `--tx-id` and `--name` deletion paths enforce this rule. Block-backup cleanup also respects an active export reference.
3. After backup files are complete and durable, export clears only its own reference. Deletion keeps its reference until the drop and clone-name cleanup finish. If a record update response is lost, read the record to confirm ownership before proceeding.

These are short atomic record updates; no database transaction remains open during file copying. Other snapshot-record updates must preserve an active reference. All tools that delete clones must honor it before this feature is enabled.

After interruption, retain the reference until the operation is confirmed finished or stopped. Do not clear it merely because a timeout expired: the backup process may still be running. An abandoned reference safely blocks deletion until explicit recovery. The reference is local snapshot bookkeeping, excluded from the state hash; it is not a new snapshot lifecycle status.

## Export: save the block and publish the archive

The source must durably back up full block `S` before successful snapshot-block processing finishes. Keep that backup available while the snapshot is offered for export, even if ordinary block history is pruned. Export reads the backup through an authorized sidecar API; cleanup cannot remove it while export is using it. File naming and storage mechanics belong in the implementation plan.

Package the original database backup and `boundary-block.pb`. Do not package historical blockstore files or separate config/checkpoint blocks. Historical config is in restored state; checkpoint arrives from orderers.

The manifest lists format/version requirements, restore settings, file paths, sizes and SHA-256 checksums. YugabyteDB native metadata holds tablet mappings. Write the manifest last, after all files are complete and durable. Reject incomplete archives, unsafe paths and nonempty output destinations. Checksums detect transfer damage; they are not proof of authenticity.

## Import: restore and prepare the database

| Backend | Export | Import |
|---|---|---|
| PostgreSQL | Selected-clone `pg_dump --format=directory --jobs=N` | New DB from `template0`, then `pg_restore --jobs=N --no-owner --no-privileges --exit-on-error` |
| YugabyteDB | Native snapshot, schema/metadata and all data/index tablet files | Restore schema, import mappings, copy files to required replicas, and wait for native restore completion |

Require supported schemas/tools and a fresh target. PostgreSQL major versions must match in the first release; preserve encoding/locale. Target live and hashing-clone names differ. Do not reuse unrelated `snapshot_<N>` databases.

For YugabyteDB, metadata alone is not a backup. Check catalog consistency and discover target tablet IDs/replicas rather than copying source node paths. Test 3→5 and 5→3 tablet-server restores at replication factor 3. Prepare native PITR/clone prerequisites before starting services.

Restore only supported committer objects from a trusted producer. Protect archive storage/transfer. Keep account passwords out of command arguments and logs; use `PGPASSFILE` or secure native-tool settings. Block/checkpoint verification does not make arbitrary SQL or extensions safe.

Before cleanup, confirm complete restore, ownership of created objects, `S>0`, representable `S+1`, and absent original `T` status. Private import work records support this check; a restored watermark alone is not proof of completion.

Apply all three cleanup changes together:

| Object | Action |
|---|---|
| `ns__snapshot` | Empty rows; retain schema, indexes and functions; no cascading deletion |
| Latest-snapshot metadata | Set SQL NULL value; retain row |
| Last-committed metadata | Set `S-1` using existing eight-byte big-endian encoding |
| Everything else | Preserve, including statuses, checkpoints and MVCC versions |

Snapshot bookkeeping and recovery metadata are excluded from the state hash. All changes commit or roll back together. Keep the pointer row: `SnapshotStateManager.ReadLatest` expects it.

Validate cleanup before reporting success or starting services. Retry only on the same owned, fully restored target before processing begins. Never reset a started target. Older local records remain in archive bytes, not the cleaned target.

## Startup: verify the block

Read historical config from restored `ns__config`. Its config transaction status identifies config block `K`; require `K<S` and matching channel/last-config metadata. Current orderer connection settings remain separate.

Verify `S`'s number, channel, config, data hash and orderer signatures under that historical policy. Apply normal transaction checks to `T`. Use [RFC 0003][block-hash-rfc]'s channel-selected block-data hash; absent/empty scheme name means `SHA256_FLAT`, and unknown schemes fail. Setting `SessionInfo.LastBlock` alone is not verification: `newBlockProcessingState` does not authenticate that seed.

Only the predecessor link from `S-1` to `S` cannot be checked, because no authenticated `S-1` is supplied. Data-hash and signature checks remain mandatory. Changing transaction bytes/TX ID breaks the data hash; changing the header to match breaks its signatures.

For the next block, require the expected number and:

```text
(S+1).Header.PreviousHash == BlockHeaderHash(verified_S.Header)
```

Normal hash-chain checks continue thereafter. Signed `S` does not prove all earlier history or authenticate an arbitrary archive's own policy.

## Startup: save the bootstrap boundary

The **saved bootstrap boundary** tells the local blockstore where this ledger started. It records block `S`, its header hash, its previous-hash field, and the historical verification config. The blockstore calls this its snapshot origin; it is separate from the database snapshot record and is not proof that `T` committed.

After verifying `S` and recovering regular statuses, save this boundary durably **before processing missing `T`**. Reuse existing `SnapshotInfo`/`BootstrappingSnapshotInfo` fields:

```text
SnapshotInfo.LastBlockNum      = S
SnapshotInfo.LastBlockHash     = BlockHeaderHash(verified_S.Header)
SnapshotInfo.PreviousBlockHash = verified_S.Header.PreviousHash
```

The copied previous-hash field does not prove that the `S-1` link was checked. Saved boundary config lets restart verify later blocks without needing historical block files or substituting newer live config.

Two blockstore values describe local history: `F` is the first available block; `H` is the next block to append (logical height). Initially both are `S+1`. Add metadata-only initialization with index savepoint at `S`, while preserving existing snapshot-import APIs. SQL statuses supply duplicate-ID checks, so historical TX-ID files are unnecessary. Inspect new storage before provider `Open` to avoid initializing it as a genesis ledger.

## Startup: process `T` and finish `S`

Reuse `appendMissingBlock`'s mapping/status lookup and `fillStatuses`, not its final append. Keep existing duplicate and malformed-transaction handling. Missing required regular status or an impossible future status fails startup. Do not trust unsigned validation filters or rerun saved writes/MVCC decisions.

Normal snapshot processing completes the target clone before committing `T`, its `PENDING` record and latest pointer. Scheduler hashes that clone using RFC 0004's state-hash contract.

Keep recovered outcomes in block tracking. Intercept completed `S` before ordinary append/client publication, finish internal accounting, and synchronously set the saved last-committed block number (watermark) to `S`. Then start public traffic, ordinary intake and the periodic watermark updater.

The required order is: **save boundary → complete `T` → save watermark → fetch `S+1`**. Require one active sidecar and no concurrent administrative progress changes.

## Restart: resume without repeating writes

| Saved state | Action |
|---|---|
| New store, configured `S`, watermark `S-1`, `T` absent | Verify block/statuses, save boundary, process only `T` |
| Boundary matches `S`, watermark `S-1`, `T` absent | Wait for coordinator idle; resume verified clone/execution retry |
| Boundary matches `S`, watermark `S-1`, exact committed `T` and valid local record | Keep clone/hash progress; finish watermark without reexecution |
| Boundary matches `S`, watermark at least `S` | Preserve advanced height/watermark; use normal recovery |
| Missing/conflicting boundary after progress, wrong record or missing required status | Fail; do not reset or relabel the ledger |

`H=S+1` alone does not prove completion. Keep staged `S` through incomplete retries; after completion, saved boundary/config can support restart without it. Resolve lost watermark responses by reading status, record and watermark. Reuse pre-commit clones only after checking ownership/native completion. Repair interrupted PostgreSQL connection exclusion through maintenance recovery. Missing active clone is an error; deleted older terminal clones do not trigger rehash.

Below `F`, unary history requests return gRPC `OutOfRange`; delivery seeks immediately return `NOT_FOUND`, including blocking seeks. `Oldest` starts at `F`; `Newest` at `H=F` has no stored block. Cached/indexed entries cannot bypass this bound. Missing historical TX indexes do not prove a TX never existed.

Orderers must serve `S+1` onward, including `C`. Missing blocks cause visible retry/failure, never a height jump. Checkpoint match, HOLD, HALT and abort behavior follow RFC 0004.

# Drawbacks
[drawbacks]: #drawbacks

- Source backups and target live/hashing databases need storage.
- Native restore can be slow and has version/topology limits.
- Export references need recovery after interrupted operations. Startup also spans separate database and blockstore writes.
- Earlier block history is unavailable locally.
- Archive contains confidential state; import cleanup does not erase backup bytes.
- Public results can be served before local checkpoint comparison, so trusted archive/config inputs are required.

# Rationale and alternatives
[alternatives]: #rationale-and-alternatives

The following alternatives were considered but not selected.

## Hold a database lock throughout export

This would protect the clone from deletion, but keep a transaction open for the entire backup. Long exports could delay PostgreSQL vacuum, and connection loss would release protection. The design instead records a durable operation reference using short atomic updates.

## Clean the clone before export

Changing the original clone would alter the source snapshot. Cleaning a separate copy would require another database copy and, for physical backups, a new capture of that cleaned state. The design instead performs one cleanup transaction after target restore, leaving the source clone unchanged.

## Fetch block `S` from orderers during startup

Orderers may have removed `S` while still serving `S+1` onward. Requiring that fetch would make an otherwise usable database backup unavailable for bootstrap. The design packages backed-up `S`; later blocks are still fetched from orderers.

## Use a whole-server PostgreSQL physical backup

`pg_basebackup` copies all server databases, roles and WAL, not just the selected clone. It could bring newer live state, account passwords or unrelated clones into the target. The design uses selected-clone logical backup. A faster physical option would need a separate isolation design and measured justification.

# Prior art
[prior-art]: #prior-art

- [Hyperledger Fabric snapshot join][fabric-snapshot] starts from saved state without full replay.
- [RFC 0004 — Database Snapshot and Checkpoint][snapshot-rfc] supplies snapshot creation, hashing and ordered checkpoint verification.
- [RFC 0003 — Block Data Hash Scheme][block-hash-rfc] supplies channel-selected block data hashing.
- [PostgreSQL logical backup][pg-dump] and [YugabyteDB YSQL snapshots][yb-snapshot] supply native transport.

# Testing
[testing]: #testing

Compare state, versions, statuses and later transaction outcomes between full replay and snapshot bootstrap. Cover:

1. **Export:** valid/deleted/corrupt selection; competing export/delete claims through both selectors; lost claim/release responses; stopped versus still-running abandoned operations; complete files and manifest-last publication. Export works after ordinary history loses `S`.
2. **Block backup:** incomplete backups are rejected; complete backups remain available through authorized reads during export. Cleanup and pruning cannot remove a backup still in use.
3. **Import:** DB-only setup, native completion/ownership, rollback of all cleanup actions, retained NULL pointer row and unchanged hashed data. Retry cannot reset a started target.
4. **Verification:** changed TX ID/data/header/signatures, wrong config/channel/height, supported/unknown hash schemes and unverified `LastBlock` seed. Check links from `S+1`.
5. **Recovery:** saved statuses, duplicate/malformed positions, only missing `T` execution, crash windows around boundary/clone/watermark writes, native retry and preserved progress. Never append/publish `S`.
6. **Checkpoint/config:** match, HOLD, mismatch HALT, abort and later config changes. Missing blocks never cause a height jump.
7. **Native transport:** PostgreSQL version/locale/ownership; YugabyteDB complete tablets/replicas, catalog consistency, clone prerequisites and 3→5/5→3 RF=3 restores.

Measure export, restore, cleanup, clone creation, hashing and catch-up separately. Each implementation PR includes tests/docs. Run focused tests locally and full suite in CI.

# Dependencies
[dependencies]: #dependencies

- **[RFC 0004][snapshot-rfc]:** snapshot transactions, clones, local records, hashing and checkpoint behavior.
- **[RFC 0003][block-hash-rfc]:** historical channel's block-data hash scheme.
- **`fabric-x-common`:** shared snapshot-record operation reference, saved bootstrap boundary/config and metadata-only initialization; preserve existing import APIs.
- **`fabric-x-committer`:** export/import commands, clone-reference enforcement, source block backup/read, cleanup and sidecar startup using existing recovery.
- **Deployment:** compatible native tools, target capacity/clone prerequisites, local signing/TLS credentials and orderers serving `S+1` onward.
- **[RFC 0002][sdk-rfc] clients:** honor oldest-available history and typed unavailable results.

Online pruning and general block-history archives do not gate bootstrap implementation.

# Unresolved questions
[unresolved]: #unresolved-questions

## Implementation details

- Operation-reference encoding, abandoned-operation recovery and enforcement by all clone-deletion tools.
- Backup-read API, access/size/time limits and cleanup protection.
- Narrow snapshot-record read if needed, plus existing recovery/no-append integration.
- Native ownership/retry procedures, supported version/topology matrix and archive compatibility fields.

## Separate future work

- **Pruning/history archives:** require durable database and full `S` backups for the selected checkpointed snapshot, locally verified checkpoint, durable progress and saved verification boundary. Archive affected block ranges before deletion; protect readers/backups and make cleanup crash-safe. This applies to genesis- and snapshot-started stores. Keep a snapshot-started store's saved bootstrap boundary unchanged when retention advances.
- **DB-backed `S`/stateless sidecar:** persist signed `S` in excluded metadata before cloning if clones should carry it. Full statelessness also needs durable boundary/config/progress and history storage; current proposal uses files/YAML.
- **Other restore methods:** whole-server PostgreSQL transport, engine/version migration and automatic repair of running deployments need separate designs.

[snapshot-rfc]: https://github.com/hyperledger/fabric-x-rfcs/blob/9422cd137f3366d1510936a34b4ca6285bf54adc/0004-database-snapshot-and-checkpoint.md
[block-hash-rfc]: https://github.com/hyperledger/fabric-x-rfcs/blob/e431fc81a1a4aa742fb7bf29813a8f2259cb333d/0003-block-data-hash-scheme.md
[sdk-rfc]: https://github.com/hyperledger/fabric-x-rfcs/blob/e431fc81a1a4aa742fb7bf29813a8f2259cb333d/0002-client-sdk.md
[fabric-snapshot]: https://hyperledger-fabric.readthedocs.io/en/latest/peer_ledger_snapshot.html
[pg-dump]: https://www.postgresql.org/docs/current/app-pgdump.html
[yb-snapshot]: https://docs.yugabyte.com/stable/manage/backup-restore/snapshot-ysql/
