---
layout: default
title: Fabric to Fabric-X Migration
nav_order: 3
---

- Feature Name: fabric_to_fabricx_migration
- Start Date: 2026-03-31
- RFC PR: -
- Fabric-X Component: fabric-x-committer, fabric-x-migrate
- Fabric-X Issue: [hyperledger/fabric-x#21](https://github.com/hyperledger/fabric-x/issues/21)

# Summary
[summary]: #summary

Fabric-X is a significant architectural evolution from Hyperledger Fabric, but
for existing Fabric deployments the question "how do we get our data across?" has
had no good answer. Without a migration path, organizations are stuck running
both systems in parallel indefinitely, or they start fresh and lose years of
production state. Neither is acceptable.

This RFC defines a complete, end-to-end migration framework for transferring
world state from a Hyperledger Fabric network into Fabric-X. It addresses the
two fundamental differences that make a naive 1-to-1 import impossible: Fabric-X
has no concept of channels, and it has no private data collections. The framework
handles both by defining rigorous data mapping, filtering, and transformation
rules — not as a hand-wave, but as a precise specification that an implementation
can follow without ambiguity.

Concretely, this proposal introduces:

1. A **canonical data set** specification: exactly which data from a Fabric peer
   snapshot is included, excluded, or transformed, with a verifiable manifest
   tying the output back to the source snapshot.
2. A **channel-to-global-state mapping** strategy: a clear architectural decision
   on how Fabric's channel-scoped namespaces map into Fabric-X's flat state
   model, with a full comparison of the available options and explicit rationale
   for the chosen approach.
3. An **Exporter CLI** (`fabric-x-migrate`): a standalone tool that reads a
   Fabric peer snapshot, applies the mapping and filtering rules, and produces
   the canonical data set ready for ingestion.
4. A **bootstrap feature** in the Fabric-X committer
   (`committer --init-from-snapshot`): a one-time initialization procedure that
   reads the canonical data set and bulk-loads state into the committer's
   database using the PostgreSQL COPY protocol, with a guard to prevent
   accidental re-initialization.
5. A **verification process** (`fabric-x-migrate verify`): an independent
   integrity check covering row counts, block height consistency, endorsement
   policy registration, and optional deep sampling of individual key-value pairs.
6. **Security and operational guidance**: snapshot trust, multi-org coordination,
   rollback strategy, ordering service integration, and performance expectations.
7. A **future enhancements roadmap**: a substantive set of follow-on directions
   the community can pursue once the foundational framework is stable.

# Motivation
[motivation]: #motivation

Fabric-X is a re-architecture of Hyperledger Fabric — microservice decomposition,
a new transaction model, pre-order execution, and a flat state model in place of
the per-channel ledger. These are meaningful improvements. But they come at a
cost: the two platforms are not compatible at the data level, and there is
currently no supported way to bring Fabric state into Fabric-X.

The practical consequence is that organizations evaluating Fabric-X face a hard
choice: start from a blank ledger (losing years of production state), or run both
systems in parallel indefinitely (paying double the operational and infrastructure
costs while keeping two code paths alive). Neither option is realistic for
enterprises whose ledger data underpins active business processes.

What typically happens instead is that individual teams build their own ad-hoc
migration scripts. These scripts are fragile, untested against edge cases, and
not shared with the community. Every team solves the same problem independently,
and none of those solutions benefit from code review, community testing, or
long-term maintenance. The result is a fragmented migration story that makes
Fabric-X adoption harder than it needs to be.

The official Fabric peer ledger snapshot feature (documented
[here](https://hyperledger-fabric.readthedocs.io/en/release-2.5/peer_ledger_snapshot.html))
provides a verifiable, point-in-time copy of the world state and is the right
starting point for migration. But a direct 1-to-1 import is not possible. Two
fundamental architectural differences must be resolved:

1. **No Channels**: Fabric-X does not have channels. In Fabric, all state is
   scoped by channel — a channel named `trade-finance` with a chaincode named
   `letters-of-credit` has its own isolated namespace completely separate from
   any other channel. Fabric-X operates on a single, flat namespace model. We
   need a precise, documented strategy for mapping Fabric's channel-scoped
   namespaces into Fabric-X's flat model. Without this, any migration tool
   would be guessing.

2. **No Private Data**: Fabric-X does not have private data collections (PDCs).
   Fabric snapshots include hashed representations of private data — not the
   actual values, just the hashes — which serve no purpose in Fabric-X and must
   be cleanly filtered out. The same applies to collection configuration history.
   These must be excluded in a deliberate, logged, and auditable way so that
   operators know exactly what was left behind.

There are also subtler issues: Fabric encodes MVCC versions as `(BlockNum, TxNum)`
pairs, while Fabric-X uses scalar integers. System chaincodes like `_lifecycle`
and `lscc` must not be migrated. CouchDB-based peers embed internal index keys
alongside application state. All of these need to be handled correctly.

This RFC addresses every one of these problems. The goal is a standard,
community-maintained migration framework that any Fabric operator can use with
confidence — one that has been thought through carefully, tested thoroughly, and
documented precisely enough that the implementation is not left to interpretation.
With the Fabric-X SDK ([RFC 0002](0002-client-sdk.md)) and EVM compatibility
([RFC 0001](0001-fabric-evm.md)) progressing in parallel, a solid migration path
completes the picture for organizations considering Fabric-X adoption.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The migration from Fabric to Fabric-X is a multi-step process. At a high level,
you take a snapshot of your Fabric peer's world state, run it through an export
tool that produces a well-defined migration artifact, bootstrap a new Fabric-X
committer instance from that artifact, and then verify that everything came
through correctly. After verification, the committer starts normally and begins
processing new transactions.

## Step 1: Quiesce and Snapshot

Before migrating, the Fabric network should be quiesced — meaning no new
transactions are submitted — to ensure the snapshot represents a consistent,
final state. Fabric 2.x and 3.x peers support
[ledger snapshots](https://hyperledger-fabric.readthedocs.io/en/release-2.5/peer_ledger_snapshot.html).
You request a snapshot at a specific block height:

```bash
peer snapshot submitrequest -c mychannel -b 1000 --peerAddress peer0:7051
```

The peer produces a directory containing the world state, transaction IDs,
collection configuration history, and signed metadata. This directory is the
input to the migration process.

## Step 2: Export to the Canonical Data Set

Run the exporter, pointing it at the snapshot directory and a destination for
the output:

```bash
fabric-x-migrate export \
  --snapshot-dir /var/hyperledger/snapshots/completed/mychannel-1000 \
  --output-dir /tmp/migration-data
```

The tool starts by verifying that the snapshot files haven't been tampered with
or corrupted in transit — it computes SHA-256 hashes and checks them against
what's recorded in `_snapshot_signable_metadata.json`. If anything doesn't
match, it stops immediately.

Then it walks through the snapshot namespace by namespace. User-defined chaincode
state (the actual application data) goes into the output. Everything else gets
filtered out:

- Private data collection hashes sit under namespaces like
  `mycc$$h$$secretCollection` — they're hashes of data that Fabric-X doesn't
  have, so there's nothing useful to import. The exporter logs every one it
  skips so you have a clear record.
- `collection_config.data` holds the history of PDC configurations. Ignored
  completely.
- System chaincodes — `_lifecycle`, `lscc`, `qscc`, `cscc`, `escc`, `vscc` —
  are Fabric internals with no Fabric-X equivalent. They don't come across.
- Transaction ID history is intentionally left out. There's a dedicated section
  below explaining the reasoning.

MVCC versions get converted from Fabric's `(BlockNum, TxNum)` pairs to the
scalar integers Fabric-X uses, and any CouchDB internal index keys get stripped
if the source peer was running CouchDB.

The output is a directory of namespace state files and endorsement policy files,
tied together by a manifest that records SHA-256 checksums for everything. That
directory is the canonical data set.

## Step 3: Bootstrap the Committer

```bash
committer --init-from-snapshot /tmp/migration-data
```

This flag puts the committer into bootstrap mode before it starts any of its
normal gRPC services. It reads the canonical data set, creates the database
schema, and bulk-loads all the namespace state using PostgreSQL's COPY
protocol — which is orders of magnitude faster than row-at-a-time inserts for
the kind of data volumes we're dealing with. Endorsement policies get
registered, the block height gets set to the snapshot's block number, and then
the committer transitions to normal operation from that point.

One important constraint: this is a one-time operation on an empty database.
If the committer finds that the metadata table already has a committed block
number, it stops and tells you. It won't touch an existing network's state.

## Step 4: Verify the Migration

Before letting the committer process real transactions, verify that what ended
up in the database matches what was exported:

```bash
fabric-x-migrate verify \
  --canonical-dir /tmp/migration-data \
  --db-endpoint localhost:5433 \
  --db-name fabricx \
  --db-user admin
```

The verifier checks row counts per namespace against the manifest, confirms the
block height is correct, and validates that endorsement policies are registered.
With `--deep` it also samples individual key-value pairs and compares them
against the canonical data set files. Don't route production traffic until this
passes.

## Transaction ID History: Why We're Not Migrating It

Fabric snapshots include `txids.data` — a sorted list of every transaction ID
committed up to the snapshot block. Fabric uses this to block replays: if a
peer sees a transaction ID it's already processed, it rejects it.

We're not migrating this history, and the reason is straightforward: a Fabric
transaction physically cannot be replayed on Fabric-X. The two platforms use
completely different transaction envelope formats, different signature schemes,
different endorsement structures. An old Fabric transaction would fail
parsing before it got anywhere near the MVCC check. On top of that, Fabric
derives transaction IDs from `SHA256(nonce || creator)` while Fabric-X uses
a different scheme — the ID spaces don't overlap.

So the replay risk that `txids.data` guards against in Fabric simply doesn't
exist in a cross-platform context. This matches the conclusion from the
[community discussion on issue #21](https://github.com/hyperledger/fabric-x/issues/21).
That said, this should be validated by a security expert before the feature
is called stable — it's the kind of reasoning that benefits from a second
pair of eyes.

If that review turns up a scenario we haven't thought of, the canonical data
set format can accommodate a `txids` section without breaking anything — it's
just not populated by default.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Analysis of Fabric Snapshot Data

A Fabric peer snapshot directory contains the following components. For each
component, we define exactly how it is handled during migration:

| Snapshot Component | Contents | Migration Action |
|--------------------|----------|------------------|
| `public_state.data` | Stream of varint-length-prefixed protobuf `SnapshotRecord` messages, each containing `key`, `value`, and `version` (encoded bytes). Records are grouped by namespace. | **Include** — primary migration data. Extract public chaincode state, convert versions, write to canonical state files. |
| `public_state.metadata` | Maps namespace names to their record count ranges in the data file. Each entry contains the namespace identifier and the number of key-value pairs belonging to it. | **Include** — used by the exporter to iterate the data file namespace-by-namespace and to identify which namespaces to include or exclude. |
| `txids.data` | Sorted list of all committed transaction IDs up to the snapshot block height, varint-length-prefixed strings. | **Exclude** — not migrated. See the Transaction ID History section above for full justification. |
| `txids.metadata` | Index metadata for `txids.data`. | **Exclude** — not applicable. |
| `collection_config.data` | History of private data collection configurations for all chaincodes (endorsement policies, dissemination policies, collection definitions). Protobuf binary format. | **Exclude** — Fabric-X does not support PDCs. This file is ignored entirely. |
| `_snapshot_signable_metadata.json` | JSON object containing `channel_name`, `last_block_number`, `last_block_hash_in_hex`, `previous_block_hash_in_hex`, `state_db_type` (CouchDB or SimpleKeyValueDB), and `files_and_hashes` (SHA-256 hashes of all snapshot files). | **Include** — used for snapshot integrity verification before export begins, and to extract the channel name and block height for the manifest. |
| `_snapshot_additional_metadata.json` | Contains `snapshot_hash` (SHA-256 of the signable metadata) and optionally `last_block_commit_hash`. | **Include** — used for integrity verification. |

### Namespace Filtering Rules

The exporter classifies each namespace found in `public_state.metadata` using
the following rules, applied in order:

1. **System chaincodes** — If the namespace is one of `_lifecycle`, `lscc`,
   `qscc`, `cscc`, `escc`, or `vscc`, it is **excluded**. These serve
   Fabric-internal functions (chaincode lifecycle, ledger queries,
   endorsement/validation plugins) that have no equivalent in Fabric-X.

2. **Private data collection hashes** — If the namespace matches the pattern
   `<chaincode>$$h$$<collection>` (where `$$` is the Fabric namespace joiner
   and `h` is the hash data prefix), it is **excluded**. These contain hashed
   representations of private data that cannot be used in Fabric-X. The
   exporter logs each excluded collection at INFO level:
   `Skipping private data hash namespace: mycc$$h$$secretCollection`.

3. **Private data collection private state** — If the namespace matches the
   pattern `<chaincode>$$p$$<collection>` (where `p` is the private data
   prefix), it is **excluded**. These would contain actual private data if
   present, but in snapshots only hashes are included.

4. **Implicit org collections** — Namespaces matching
   `_implicit_org_<MSPID>` are **excluded** as they are Fabric-specific
   implicit private data collections.

5. **All remaining namespaces** — These are user-defined chaincode namespaces
   and are **included** in the canonical data set.

The exporter produces a complete audit trail of its filtering decisions in both
the manifest's `excluded` section and its log output, so operators have full
visibility into what was included and what was skipped.

### CouchDB vs LevelDB Considerations

The `state_db_type` field in `_snapshot_signable_metadata.json` indicates
whether the source peer used CouchDB or LevelDB (SimpleKeyValueDB). The
snapshot format normalizes the world state into key-value pairs regardless of
the backend, so the exporter does not need special handling for either database
type. However, CouchDB-based peers may store additional internal index
definitions that appear as keys with the prefix `\x00` in certain namespaces.
The exporter strips these internal CouchDB artifacts during export.

## Canonical Data Set Specification

The canonical data set is the complete, self-contained output of the export
process and the sole input to the committer bootstrap process. It is a
directory with the following structure:

```
migration-data/
├── manifest.json
├── namespaces/
│   ├── mycc.state
│   ├── othercc.state
│   └── ...
└── policies/
    ├── mycc.policy
    ├── othercc.policy
    └── ...
```

### manifest.json

The manifest is the central metadata file. It records everything needed to
understand the provenance, contents, and integrity of the canonical data set.

```json
{
  "format_version": 1,
  "source": {
    "type": "fabric-snapshot",
    "channel": "mychannel",
    "block_number": 1000,
    "last_block_hash": "a1b2c3...",
    "state_db_type": "SimpleKeyValueDB",
    "snapshot_signable_metadata_hash": "d4e5f6..."
  },
  "namespaces": [
    {
      "name": "mycc",
      "row_count": 50000,
      "state_file": "namespaces/mycc.state",
      "state_file_sha256": "abc123...",
      "policy_file": "policies/mycc.policy",
      "policy_file_sha256": "def456..."
    }
  ],
  "total_keys": 75000,
  "excluded": {
    "system_chaincodes": ["_lifecycle", "lscc", "qscc"],
    "private_data_namespaces": [
      "mycc$$h$$secretCollection",
      "mycc$$p$$secretCollection"
    ],
    "implicit_org_collections": ["_implicit_org_Org1MSP"]
  },
  "transaction_ids_migrated": false,
  "transaction_ids_migration_rationale": "Different transaction formats between Fabric and Fabric-X prevent cross-platform replay attacks. See RFC for full analysis.",
  "created_at": "2026-03-31T10:00:00Z"
}
```

Key design decisions in the manifest:
- `format_version` allows future evolution of the canonical data set format
  without breaking older tooling.
- `state_file_sha256` and `policy_file_sha256` enable the verification tool
  to confirm file integrity without re-reading the source snapshot.
- `snapshot_signable_metadata_hash` ties the canonical data set back to the
  original Fabric snapshot for full provenance tracking.
- `excluded` provides a complete, auditable record of all filtering decisions.
- `transaction_ids_migrated` explicitly documents whether TX-IDs were included,
  with a rationale field for transparency.

### State Files

Each `.state` file contains the key-value-version triples for one namespace.
The format is a sequence of length-prefixed protocol buffer records:

```protobuf
message MigrationRecord {
  bytes  key     = 1;
  bytes  value   = 2;
  uint64 version = 3;  // Converted from Fabric's (BlockNum, TxNum) encoding
}
```

Each record is preceded by a 4-byte big-endian length prefix indicating the
size of the serialized protobuf message. This format is streaming-friendly —
the importer reads records one at a time without loading the entire file into
memory. Without this, namespaces with millions of keys would blow up memory.

### Version Conversion

Fabric encodes MVCC versions as `(BlockNum, TxNum)` pairs using the
`OrderPreservingVarUint64` encoding scheme. This is a variable-length encoding
where the first byte indicates the number of following bytes, and the remaining
bytes store the value in big-endian format with leading zeros trimmed. The
encoding preserves byte-order comparison without decoding.

Fabric-X uses scalar `BIGINT` versions (int64, stored as `version >= 0` in the
database). The canonical data set converts Fabric versions to scalar uint64
values using the following scheme:

```
version = (blockNum * MAX_TX_PER_BLOCK) + txNum
```

Where `MAX_TX_PER_BLOCK` defaults to `1,048,576` (2^20). This preserves the
total ordering of Fabric versions: if version A comes before version B in
Fabric's `(BlockNum, TxNum)` ordering, then the converted scalar A is strictly
less than the converted scalar B.

**Why 2^20?** Fabric's default `MaxMessageCount` (maximum transactions per
block) is typically configured between 10 and 500. A multiplier of 2^20
(~1 million) provides orders of magnitude of headroom while keeping the
resulting values well within the int64 range. Even with 1 billion blocks and
1 million transactions per block, the maximum version value would be
~1.05 × 10^15, still far below int64's maximum of ~9.2 × 10^18.

The exporter validates that no version exceeds `int64` range during conversion
and aborts with an error if this condition is violated. The multiplier can be
overridden with `--max-tx-per-block` for networks with unusual configurations,
but the default should work for all practical deployments.

**Decoding the source version**: The exporter reads the raw version bytes from
each `SnapshotRecord` and decodes them as follows:

1. Read the first byte to determine the length `n` (1–8).
2. Read the next `n` bytes as a big-endian unsigned integer → `blockNum`.
3. Repeat for `txNum`.
4. Compute `version = blockNum * MAX_TX_PER_BLOCK + txNum`.

## Channel-to-Global State Mapping

This is the most critical architectural decision in the migration design.
Fabric organizes state by channel, where each channel is an independent
blockchain with its own ledger, namespaces, and membership. Fabric-X does not
have channels — each committer instance operates on a single, flat namespace
model.

Two strategies were evaluated:

### Option 1: Key Prefixing

Merge data from multiple Fabric channels into a single Fabric-X instance by
prefixing every key with its channel of origin.

**Mechanism**: For each key-value pair, the exporter prepends the channel name
to the key: `mychannel_mycc_originalKey`. The namespace in Fabric-X becomes
`mychannel_mycc` (channel + chaincode combined).

**Advantages**:
- All data from all channels exists in a single Fabric-X instance.
- Enables cross-channel queries that were not possible in Fabric.

**Disadvantages**:
- **Namespace collision risk**: If two channels happen to have chaincodes with
  names that, when combined with channel prefixes, produce the same Fabric-X
  namespace identifier, data would collide. While unlikely, it requires
  validation logic.
- **Application modification required**: Existing application code that
  references keys by their original names would need to be updated to include
  the channel prefix. This is a breaking change for all client applications.
- **Endorsement policy complexity**: Policies that reference namespaces by
  name would need to account for the prefixed names.
- **Semantic mismatch**: Channels in Fabric exist precisely to provide
  isolation. Merging them into a single instance removes that isolation,
  potentially violating the original network's governance assumptions.
- **Operational complexity**: Debugging, monitoring, and access control become
  more complex when data from multiple independent channels is co-located.

### Option 2: One-to-One Mapping (Recommended)

Each Fabric channel migrates to its own dedicated Fabric-X committer instance.
Within each channel, the original chaincode namespace names are preserved
exactly as-is.

**Mechanism**: The migration is scoped to a single channel at a time. The
exporter reads one channel's snapshot and produces one canonical data set.
That data set bootstraps one committer instance. If the Fabric network has
three channels, the operator runs the process three times, producing three
independent Fabric-X instances.

**Advantages**:
- **Natural architectural alignment**: Fabric channels are, in practice,
  independent blockchain instances that share infrastructure. Each channel has
  its own genesis block, its own membership, and its own ledger. Mapping one
  channel to one Fabric-X instance mirrors this reality. This observation was
  also noted in the
  [community discussion](https://github.com/hyperledger/fabric-x/issues/21):
  "channels effectively function as separate Fabric instances, so this mapping
  aligns naturally with the architecture."
- **No namespace collisions**: Since each Fabric-X instance contains data from
  only one channel, there is zero risk of namespace conflicts.
- **No application changes**: Client applications continue to reference keys
  and namespaces by their original names. No code modifications needed.
- **Clean isolation**: Data governance, access control, and endorsement
  policies carry over without transformation.
- **Simpler implementation**: The exporter, importer, and verification tool do
  not need to handle multi-channel merging logic, prefix management, or
  collision detection.
- **Independent operation**: Each migrated instance can be started, stopped,
  upgraded, and scaled independently.

**Disadvantages**:
- Organizations with many channels need to operate multiple Fabric-X instances.
  However, this is consistent with Fabric-X's architecture where each instance
  is lightweight and independently deployable.
- Cross-channel queries remain impossible (but they were also impossible in
  Fabric, so this is not a regression).

### Decision

**This proposal adopts Option 2 (One-to-One Mapping).** The advantages are
decisive: it preserves the isolation guarantees that channels were designed to
provide, requires no application modifications, eliminates collision risks,
and aligns with both Fabric-X's architecture and the community's feedback.
Option 1's theoretical advantage of consolidation does not justify the
complexity and risks it introduces.

## Exporter Design (`fabric-x-migrate export`)

The exporter is a standalone CLI tool. It reads a Fabric peer snapshot
directory and produces the canonical data set.

### CLI Interface

```
fabric-x-migrate export [flags]

Required flags:
  --snapshot-dir    Path to the Fabric snapshot directory
  --output-dir     Path where the canonical data set will be written

Optional flags:
  --max-tx-per-block    Multiplier for version conversion (default: 1048576)
  --log-level           Logging verbosity: debug, info, warn, error (default: info)
  --skip-integrity      Skip snapshot integrity verification (not recommended)
  --dry-run             Preview what would be exported without writing any files
```

When `--dry-run` is specified, the exporter performs Phase 1 (integrity
verification) and Phase 2 (namespace discovery and classification) but skips
the actual state export. It prints a detailed summary of what would be
included and excluded, along with estimated output size based on record
counts. This allows operators to review the migration scope before committing
to a full export.

### Exporter Algorithm

The export process follows this exact sequence:

**Phase 1: Integrity Verification**

1. Read `_snapshot_signable_metadata.json` and parse the JSON to extract
   `channel_name`, `last_block_number`, `state_db_type`, and the
   `files_and_hashes` map.
2. For each file listed in `files_and_hashes`, compute its SHA-256 hash and
   compare against the recorded hash. If any hash does not match, abort with
   an error identifying the corrupted file. This step can be skipped with
   `--skip-integrity` for development/testing, but is strongly recommended
   for production migrations.
3. If `_snapshot_additional_metadata.json` exists, verify that
   `snapshot_hash` matches the SHA-256 of
   `_snapshot_signable_metadata.json`.

**Phase 2: Namespace Discovery and Classification**

4. Read `public_state.metadata` to discover all namespaces and their record
   counts.
5. Classify each namespace using the filtering rules defined above (system
   chaincodes, PDC hashes, PDC private state, implicit org collections).
6. Log the classification of every namespace at INFO level:
   ```
   Including namespace: mycc (50000 records)
   Including namespace: othercc (25000 records)
   Excluding system chaincode: _lifecycle (120 records)
   Excluding PDC hash namespace: mycc$$h$$secretCol (3000 records)
   ```

**Phase 3: State Export**

7. Create the output directory structure (`namespaces/`, `policies/`).
8. Open `public_state.data` for sequential reading.
9. For each namespace in the metadata (in order):
   a. Read the namespace's record count from metadata.
   b. If the namespace is excluded, skip that many records in the data file.
   c. If the namespace is included:
      - Open the output `.state` file.
      - Initialize a SHA-256 hasher for the output file.
      - For each record in the namespace:
        - Read the varint-length-prefixed `SnapshotRecord` from the data file.
        - Decode the key, value, and version bytes.
        - Decode the Fabric version (two `OrderPreservingVarUint64` values:
          blockNum and txNum).
        - Convert to scalar: `version = blockNum * MAX_TX_PER_BLOCK + txNum`.
        - Validate that the version fits in int64.
        - If `state_db_type` is CouchDB, check for and skip internal index
          keys (keys starting with `\x00`).
        - Serialize a `MigrationRecord` protobuf message.
        - Write the 4-byte big-endian length prefix followed by the serialized
          message to the output file.
        - Update the SHA-256 hasher.
      - Close the output file and record its SHA-256 hash.
      - Record the row count.

**Phase 4: Policy Export**

10. For each included namespace, extract the endorsement policy. In Fabric,
    endorsement policies are part of the chaincode definition stored in the
    `_lifecycle` namespace. The exporter reads the relevant policy data and
    writes it as a raw bytes file (`.policy`). If no explicit policy is
    available (e.g., the chaincode used the channel default), the exporter
    records this in the manifest so the operator can configure the policy
    manually in Fabric-X.

**Phase 5: Manifest Generation**

11. Write `manifest.json` with all metadata, file references, SHA-256 hashes,
    row counts, excluded namespaces, and provenance information.
12. Log a summary:
    ```
    Export complete: 2 namespaces, 75000 total keys, block height 1000
    Output: /tmp/migration-data
    ```

### Error Handling

- If the snapshot directory is missing required files, the exporter aborts
  immediately with a clear error message listing the missing files.
- If integrity verification fails, the exporter aborts and reports which
  file's hash did not match.
- If version conversion overflows int64, the exporter aborts and suggests
  using `--max-tx-per-block` with a smaller value.
- If the output directory already exists and is non-empty, the exporter
  aborts to prevent accidental overwriting of a previous export.
- All errors are reported with sufficient context for the operator to
  diagnose and resolve the issue.

### Memory Efficiency

The exporter processes records in a streaming fashion. It never loads an
entire namespace's state into memory. Records are read one at a time from the
snapshot data file and written one at a time to the output state file. The
peak memory usage is proportional to the size of a single key-value pair,
not to the size of the namespace. For namespaces with millions of keys, anything else would be impractical.

## Committer Bootstrap (`--init-from-snapshot`)

The committer's `start` command gains a `--init-from-snapshot <path>` flag.
When set, the committer enters bootstrap mode before starting its normal gRPC
services.

### One-Time Operation Guard

Before beginning the import, the committer checks whether the database already
contains state:

1. Attempt to read the `last committed block number` key from the `metadata`
   table.
2. If the metadata table exists and contains a non-null value for this key,
   the database has already been initialized. The committer logs an error and
   exits:
   ```
   Error: database already contains committed state (last block: 500).
   The --init-from-snapshot flag can only be used on an empty database.
   ```
3. If the metadata table does not exist or the key's value is null, proceed
   with bootstrap.

This guard ensures that bootstrap cannot accidentally overwrite a running
network's state.

### Bootstrap Sequence

The bootstrap process uses the `StateImporter` component to execute the
following operations on the database:

**Step 1: Schema Initialization**

Call `StateImporter.InitSchema()`, which executes the database initialization
SQL:

```sql
CREATE TABLE IF NOT EXISTS metadata (
    key   BYTEA NOT NULL PRIMARY KEY,
    value BYTEA
);
INSERT INTO metadata VALUES ('last committed block number', NULL)
  ON CONFLICT DO NOTHING;

CREATE TABLE IF NOT EXISTS tx_status (
    tx_id  BYTEA NOT NULL PRIMARY KEY,
    status INTEGER,
    height BYTEA NOT NULL
);
```

Then creates tables and stored functions for the system namespaces `__meta`
and `__config` using the namespace table template:

```sql
CREATE TABLE IF NOT EXISTS ns_<namespace_id> (
    key     BYTEA                    NOT NULL PRIMARY KEY,
    value   BYTEA  DEFAULT NULL,
    version BIGINT DEFAULT 0::BIGINT NOT NULL CHECK (version >= 0)
);
```

Along with the corresponding `insert_ns_<id>`, `update_ns_<id>`, and
`validate_reads_ns_<id>` stored functions.

**Step 2: Namespace Table Creation and State Import**

For each namespace listed in the manifest:

1. Call `StateImporter.CreateNamespace(nsID)` to create the namespace-specific
   table (`ns_<nsID>`) and its stored functions.
2. Open the corresponding `.state` file.
3. Wrap it in a `StateIterator` that reads `MigrationRecord` messages one at
   a time.
4. Call `StateImporter.ImportNamespaceState(nsID, iterator)`, which uses
   PostgreSQL's COPY protocol via `pgx.CopyFrom()` to bulk-insert rows
   directly into the `ns_<nsID>` table. The COPY protocol streams rows to
   the database in a binary format that bypasses per-row SQL parsing,
   achieving throughput orders of magnitude higher than individual INSERT
   statements.
5. Log progress: `Imported 50000 rows into namespace [mycc]`.

**Step 3: Policy Registration**

For each namespace, call `StateImporter.ImportPolicy(nsID, policyBytes)` to
insert the endorsement policy into the `__meta` system namespace. This uses
the `insert_ns___meta` stored function, which inserts the namespace ID as the
key and the serialized policy as the value.

**Note on the `__config` system namespace**: The `__config` namespace is
created during schema initialization but is not populated during snapshot
bootstrap. In normal Fabric-X operation, `__config` stores the channel
configuration transaction envelope. Since Fabric-X does not use Fabric-style
channel configuration, this namespace remains empty after migration. The
coordinator and other components that depend on configuration should be
initialized separately as part of the Fabric-X network setup (see
Operational Considerations below).

**Step 4: Block Height Setting**

Call `StateImporter.SetBlockHeight(blockNumber)` to record the snapshot's
block height in the metadata table:

```sql
UPDATE metadata SET value = $2 WHERE key = 'last committed block number';
```

The block number is encoded as an 8-byte big-endian uint64. After this,
`getNextBlockNumberToCommit()` will return `blockNumber + 1`, so the
committer resumes processing from the block after the snapshot.

**Step 5: Integrity Verification**

Call `StateImporter.VerifyImport(summary)`, which:

1. For each namespace, executes `SELECT count(*) FROM ns_<id>` and compares
   against the expected row count from the manifest.
2. Calls `getNextBlockNumberToCommit()` and verifies it returns
   `blockNumber + 1`.
3. If any check fails, the committer logs the mismatch and exits with an
   error.

**Step 6: Transition to Normal Operation**

If all checks pass, the committer logs a success message and starts its
normal gRPC services (preparer → validator → committer pipeline). From this
point, it processes new transactions normally.

### Error Handling and Recovery

If bootstrap fails at any point (database connection error, disk full, COPY
failure, verification mismatch), the database is left in a partially
initialized state. Recovery is straightforward:

1. Drop the database: `DROP DATABASE fabricx;`
2. Create a fresh database: `CREATE DATABASE fabricx;`
3. Fix the underlying issue (disk space, connectivity, etc.).
4. Re-run the committer with `--init-from-snapshot`.

This is safe because bootstrap operates exclusively on an empty database.
There is no risk of corrupting existing state.

### Performance Characteristics

The bottleneck during bootstrap is the COPY-based state import into the
database. Expected throughput depends on hardware, network latency to the
database, and the average size of key-value pairs:

| Data Size | Estimated Keys | Estimated Import Time | Disk (DB) |
|-----------|---------------|----------------------|-----------|
| Small     | < 100K        | < 30 seconds         | < 1 GB    |
| Medium    | 100K – 1M     | 1 – 5 minutes        | 1 – 10 GB |
| Large     | 1M – 10M      | 5 – 30 minutes       | 10 – 100 GB |
| Very large| > 10M         | 30+ minutes          | 100+ GB   |

These are rough estimates assuming average key-value sizes of ~500 bytes and
a PostgreSQL instance on local SSD storage. YugabyteDB with tablet
pre-splitting may show different characteristics. The export phase is
typically I/O-bound and completes faster than import since it only involves
sequential file reads and writes.

Operators should ensure sufficient disk space for both the canonical data set
(approximately equal to the uncompressed world state size) and the database
(which adds indexing overhead of roughly 2–3x the raw data size).

## Verification (`fabric-x-migrate verify`)

The verification tool provides an independent check that the migration was
successful. It reads the canonical data set manifest and queries the
committer's database to confirm consistency.

### CLI Interface

```
fabric-x-migrate verify [flags]

Required flags:
  --canonical-dir   Path to the canonical data set directory
  --db-endpoint     Database host:port
  --db-name         Database name
  --db-user         Database username

Optional flags:
  --db-password     Database password (or set via FABRIC_X_DB_PASSWORD env var)
  --db-tls          Enable TLS for database connection
  --deep            Enable deep verification (sample-based value comparison)
  --sample-rate     Fraction of keys to verify in deep mode (default: 0.01)
```

### Verification Algorithm

**Level 1: Structural Verification (always performed)**

1. Read `manifest.json` from the canonical data set.
2. Connect to the committer's database.
3. For each namespace in the manifest:
   a. Verify the table `ns_<nsID>` exists.
   b. Execute `SELECT count(*) FROM ns_<nsID>` and compare against the
      manifest's `row_count`.
   c. Report any mismatch.
4. Read the `last committed block number` from the metadata table, decode
   the 8-byte big-endian value, and verify it equals the manifest's
   `block_number`.
5. Verify that endorsement policies exist in `ns___meta` for each namespace.

**Level 2: Deep Verification (with `--deep` flag)**

6. For each namespace, open the corresponding `.state` file.
7. Sample keys at the configured rate (default 1%).
8. For each sampled key, query the database:
   `SELECT value, version FROM ns_<nsID> WHERE key = $1`
9. Compare the returned value and version against the canonical data set
   record. Report any mismatch with full details (key, expected value/version,
   actual value/version).

**Output**

The verification tool produces a structured report:

```
Verification Report
===================
Namespace: mycc
  Row count: 50000 (expected) / 50000 (actual) .. OK
  Deep check: 500/500 sampled keys match .. OK

Namespace: othercc
  Row count: 25000 (expected) / 25000 (actual) .. OK
  Deep check: 250/250 sampled keys match .. OK

Block height: 1000 (expected) / 1000 (actual) .. OK
Policies: 2/2 namespaces registered .. OK

Result: PASS
```

If any check fails, the tool exits with a non-zero exit code and the report
clearly identifies the discrepancy.

## Complete Example Walkthrough

```bash
# 1. Quiesce the Fabric network (stop submitting new transactions)

# 2. Take a snapshot on your Fabric peer at block 1000
peer snapshot submitrequest -c mychannel -b 1000 --peerAddress peer0:7051

# 3. Wait for the snapshot to complete, then locate it
ls /var/hyperledger/production/snapshots/completed/mychannel/1000/

# 4. Export the snapshot to the canonical data set
fabric-x-migrate export \
  --snapshot-dir /var/hyperledger/production/snapshots/completed/mychannel/1000 \
  --output-dir /tmp/migration-data

# 5. Inspect the manifest to review what was exported
cat /tmp/migration-data/manifest.json | jq .

# 6. Prepare the database for the Fabric-X committer
createdb fabricx

# 7. Bootstrap the committer from the canonical data set
committer --init-from-snapshot /tmp/migration-data

# 8. Verify the migration
fabric-x-migrate verify \
  --canonical-dir /tmp/migration-data \
  --db-endpoint localhost:5433 \
  --db-name fabricx \
  --db-user admin \
  --deep

# 9. If verification passes, start the committer normally for ongoing operation
committer start --config committer-config.yaml
```

# Drawbacks
[drawbacks]: #drawbacks

**Private data is not migrated.**

Fabric snapshots contain only hashes of private data, not the actual values.
Since Fabric-X does not support private data collections at all, there is
nothing meaningful to migrate even if we wanted to. Organizations that rely
heavily on PDCs will need to handle that data through a separate,
application-level process — whether that means exporting it from the peers'
side databases, archiving it, or accepting that it stays on the Fabric side.
The exporter logs every excluded PDC namespace clearly so operators have a
complete picture of what was left behind. This is a real limitation, and it
should be communicated transparently to organizations evaluating migration.
The Future Enhancements section describes how this could be addressed if
Fabric-X gains private data support down the line.

**Version semantics change.**

Fabric's `(BlockNum, TxNum)` MVCC version encoding is converted to a scalar
integer during migration. The ordering of versions is preserved — a key that
was written later will always have a higher version number in Fabric-X — but
the internal structure is lost. Any application code that tries to parse or
interpret version values directly will break. In practice this should be rare,
since the Fabric SDKs deliberately treat versions as opaque, but it is worth
auditing application code before migrating. The version conversion algorithm
is fully specified and deterministic, so it can be tested against known values.

**One-time cutover, not continuous sync.**

This proposal is a snapshot-based cutover. The Fabric network gets quiesced,
a snapshot is taken, and Fabric-X takes over from that point. There is no
mechanism to keep both systems in sync during migration. For most networks
this is fine — the maintenance window is short and predictable. But for
networks where even a few minutes of downtime is genuinely unacceptable, this
approach is not sufficient on its own. The incremental migration path described
in Future Enhancements would address this, but it is a separate, more complex
piece of work.

**Transaction history is not migrated.**

This is a deliberate design decision (see the Transaction ID History section
for the full reasoning), but it is still a loss. Historical transaction IDs
are not imported, meaning you cannot query the Fabric-X committer about whether
a specific historical Fabric transaction was committed. If audit or regulatory
requirements demand access to historical transaction records, those queries
will need to go to the preserved Fabric ledger rather than Fabric-X.

**New tooling to maintain.**

Adding `fabric-x-migrate` means adding surface area to maintain as both Fabric's
snapshot format and Fabric-X's internals evolve. Fabric's snapshot format has
been stable across 2.x and 3.x, which gives some confidence, but there are no
guarantees. The mitigation is keeping the exporter as a standalone tool that
depends on Fabric protobuf definitions through a versioned dependency, rather
than coupling it tightly to either platform's internals.

**Endorsement policy mapping may require manual intervention.**

Fabric endorsement policies are serialized as protobuf and reference MSP IDs
and organizational roles. The initial implementation imports them as opaque
bytes. If the Fabric-X policy evaluation engine interprets these differently
from Fabric's, or if MSP IDs change during the migration, policies may not
evaluate correctly after import. Operators should verify that endorsement
policies are correctly applied to at least one test transaction during the
post-migration verification step before routing production traffic.

# Security considerations
[security]: #security-considerations

You're moving the entire world state of a production blockchain. The security
bar here should be at least as high as the security of the network itself.

## Snapshot Trust and Provenance

The Fabric snapshot is where everything starts. If it's been tampered with,
every downstream step produces a corrupted result — and you won't necessarily
know until something breaks in production.

The exporter verifies file hashes against `_snapshot_signable_metadata.json`
before doing anything else. This catches corruption in transit but it only
proves the snapshot is internally consistent — it doesn't prove it came from
a legitimate peer. In a multi-organization network, each organization should
take a snapshot from their own peer independently and compare the
`_snapshot_signable_metadata.json` hashes out of band. Matching hashes mean
consistent ledger state. A mismatch means there's a divergence that needs
investigating before migration proceeds.

The integrity chain extends through the whole pipeline: the exporter records
the snapshot's metadata hash in the manifest, and the verification tool
confirms the database matches the canonical data set. Snapshot → canonical
data set → database — each step is verifiable against the previous one.

## Data Confidentiality

The `.state` files in the canonical data set contain the full, plaintext
world state of your Fabric network. Treat them accordingly: encrypted volumes,
restricted file permissions, and careful handling if you're moving them between
machines.

For database credentials, use the `FABRIC_X_DB_PASSWORD` environment variable
rather than the `--db-password` flag. Flags show up in process listings and
shell history; environment variables don't.

## Bootstrap Access Control

`--init-from-snapshot` is a powerful operation. It sets the entire state of
the committer's database. Restrict access to it: the committer binary should
only be executable by authorized system accounts, and the canonical data set
directory should have tight file permissions.

The one-time operation guard is a safety check, not a security control. Someone
with database access could drop and recreate the database to bypass it. The
real protection comes from securing the database itself through standard
PostgreSQL/YugabyteDB access controls, not from the guard.

## Transaction Replay Analysis

The case for not migrating transaction IDs is covered in the guide-level
section, but to be precise about the security reasoning: a Fabric transaction
cannot be replayed on Fabric-X because the two platforms use different envelope
formats, different signature schemes, and different endorsement structures. The
transaction would fail parsing or signature verification before getting anywhere
near the committer.

This should be formally reviewed by a security expert before the feature is
called stable. The review should specifically confirm that (1) no historical
Fabric transaction can be turned into a valid Fabric-X transaction, (2) the
transaction ID spaces are disjoint, and (3) nothing in the migrated state
(key-value pairs, versions, policies) can be exploited to forge valid
Fabric-X transactions.

# Operational considerations
[operations]: #operational-considerations

This section provides practical guidance for operators planning a migration.

## Pre-Migration Checklist

In a multi-organization Fabric network, migration is not something one org can
do unilaterally. All participating organizations need to agree on the block
height for the snapshot, the timeline for quiescing the network, and how
snapshot hashes will be compared out-of-band to confirm ledger consistency.
The cutover procedure — when Fabric stops and Fabric-X takes over — needs to
be communicated and agreed upon across all parties well before it happens.

The Fabric-X infrastructure also needs to be ready before bootstrap begins.
The ordering service should be deployed and configured. The committer's
database (PostgreSQL or YugabyteDB) should be provisioned with enough storage.
The sidecar, coordinator, and any other required components should be wired up
and ready — they just shouldn't be started until after the committer finishes
its bootstrap.

Before committing to a production migration window, run through the whole thing
once against a test database. Take a real snapshot, run `--dry-run` to review
what will be exported, do the full export and bootstrap, and time it. That
test run will tell you how much storage the canonical data set needs, how long
the import takes with your actual data volume, and whether anything in your
environment breaks the process. The 2–3x database storage overhead relative to
the raw state size is also worth accounting for in your provisioning plan.

## Ordering Service Coordination

After bootstrap, the committer's metadata table records the last committed
block number as the snapshot's block height. When the committer starts
normally, it calls `getNextBlockNumberToCommit()` which returns
`snapshotBlockNumber + 1`. The committer then requests blocks starting from
this height from the ordering service via the sidecar.

This means the ordering service must be initialized to begin sequencing new
transactions into blocks starting at the correct height. The Fabric-X network's
initial configuration must reference the same logical starting point as the
snapshot, and the sidecar delivery endpoint must be reachable by the committer.

The exact procedure for setting this up depends on the ordering service
implementation and is outside the scope of this RFC. But it is a critical
operational dependency — the committer cannot process new transactions until
this is in place, and it needs to be worked out before the production cutover.

## Rollback Strategy

Migration is a one-way operation by design, but things can go wrong —
configuration issues, unexpected behavior in the Fabric-X network, problems
discovered during burn-in. The most important rule is: do not decommission
the Fabric peers or ordering service until Fabric-X has been verified and
running in production long enough to be confident. Keep the original Fabric
snapshot and canonical data set on durable storage as well. If something needs
to be redone, the export doesn't need to be repeated — just drop the database,
create a fresh one, and re-run the bootstrap with the existing canonical data
set.

This migration does not provide dual-write between Fabric and Fabric-X. The
cutover is atomic: at a defined moment, applications switch from Fabric to
Fabric-X. There is no gradual traffic splitting or parallel write mode.

## Post-Migration Checklist

Once bootstrap and verification pass, start the committer in normal mode and
confirm it connects to the ordering service and sidecar. Don't route production
traffic yet — submit a test transaction first and confirm it commits, then
confirm the resulting state change shows up in the database. Watch the
committer's Prometheus metrics on the monitoring endpoint for any errors or
anomalies during this warmup period.

Client applications can be migrated gradually from the Fabric Gateway/SDK to
the Fabric-X SDK or direct gRPC clients — there's no reason to do it all at
once. After the Fabric-X network has been running stably for a sufficient
burn-in period, the Fabric infrastructure can be decommissioned. Don't do it
earlier — having the original Fabric network intact is the safest rollback
option you have.

## Multi-Channel Migration

For Fabric networks with multiple channels, each channel is migrated
independently:

1. Take a snapshot of each channel.
2. Run `fabric-x-migrate export` for each channel.
3. Provision a separate database for each Fabric-X committer instance.
4. Run `committer --init-from-snapshot` for each instance.
5. Verify each instance independently.

The channels can be migrated in parallel or sequentially depending on
operational preference. There is no dependency between channel migrations.

# Rationale and alternatives
[alternatives]: #alternatives

Replaying the entire blockchain from genesis would produce the same world
state, but for a large ledger it would take hours or days. Snapshots give us
the world state directly — which is all we actually need to bootstrap
Fabric-X — and the peer's snapshot feature has been stable and well-tested
since Fabric 2.x. The tradeoff is that transaction history doesn't come across,
but that history stays preserved in the original Fabric ledger and Fabric-X
doesn't need it to operate.

The exporter is a standalone CLI rather than logic embedded in the committer
because these two things have very different dependency profiles. The exporter
needs Fabric's protobuf definitions (`fabric-protos-go-apiv2`, `SnapshotRecord`)
to read snapshot files. The committer should not carry those. Keeping them
separate also means the exporter can run on the same machine as the Fabric
peer — avoiding the need to move large snapshot directories across the network
— while the committer stays on its own machine. A coupled design would force
one or the other to live in the wrong place.

The one-to-one channel mapping (Option 2 over Option 1) is covered thoroughly
in the Channel-to-Global State Mapping section, but the short version is this:
key prefixing adds namespace collision risks, forces application code changes,
destroys the isolation guarantees that channels were designed to provide, and
adds complexity for a benefit — consolidation — that most deployments don't
actually need. One-to-one mapping requires no application changes, carries
over isolation naturally, and aligns with how both platforms model state.

The Fabric Smart Client was considered and ruled out for migration use.
It's built for interactive transaction flows — multi-party protocols,
peer-to-peer negotiation — not bulk state transfer. Adapting it would bring
in significant dependencies and complexity. A purpose-built tool is simpler
to test, simpler to audit, and simpler to maintain over time.

Transaction ID migration is addressed in the dedicated section in the
guide-level explanation and in the Security section. The reasoning is that
Fabric and Fabric-X use incompatible transaction formats, so a Fabric
transaction cannot survive the parsing stage in Fabric-X regardless of what
the TX-ID history looks like. That reasoning warrants a formal security review
before the feature is promoted to stable.

Without a standard migration path, the community will end up with a patchwork
of one-off scripts — fragile, untested, and different for every deployment.
That's not a foundation Fabric-X adoption can stand on.

# Prior art
[prior-art]: #prior-art

- **Fabric peer snapshot and restore**: Fabric 2.x introduced peer snapshots
  specifically to enable joining a channel from a point-in-time state without
  replaying the full blockchain. The feature is documented
  [here](https://hyperledger-fabric.readthedocs.io/en/release-2.5/peer_ledger_snapshot.html)
  and the snapshot format has been stable across Fabric 2.x and 3.x releases.
  Our exporter builds directly on this mechanism, using the same protobuf
  definitions and file format. The key innovation here is not the snapshot
  itself, but the transformation layer that makes Fabric snapshots consumable
  by a structurally different system.

- **Fabric Smart Client**
  ([github.com/hyperledger-labs/fabric-smart-client](https://github.com/hyperledger-labs/fabric-smart-client)):
  While not a migration tool, the Smart Client demonstrates how to interact
  with Fabric state programmatically outside of chaincode and already supports
  Fabric-X as a backend. Its approach to state abstraction informed some of
  the interface design decisions in this proposal, particularly the streaming
  iterator pattern for reading state.

- **Cosmos SDK genesis export**: The Cosmos ecosystem handles chain upgrades
  and migrations through genesis files — a full export of state that is used
  to initialize a new chain version. The pattern of export → canonical format
  → new chain bootstrap is directly analogous to what this RFC proposes. Cosmos
  has dealt with similar challenges around version semantics and module
  filtering, and their tooling is worth studying as implementation begins.

- **PostgreSQL logical replication and pg_upgrade**: PostgreSQL's own tooling
  for migrating between major versions follows a similar export-transform-verify
  pattern. The use of the COPY protocol for high-throughput bulk loading is
  directly borrowed from PostgreSQL best practices for data ingestion.

- **Database migration tooling**: Tools like `pg_dump`/`pg_restore`, Flyway,
  and Liquibase follow a similar pattern of export → transform → import →
  verify. The principle of keeping the canonical data set as an intermediate,
  inspectable artifact (rather than migrating directly from source to
  destination) comes from this tradition — it allows verification and replay
  without touching the source again.

- **Ethereum state migration**: Ethereum clients have dealt with state
  migration during hard forks and client switches (e.g., Geth to Nethermind).
  These typically involve streaming state tries, which is analogous to our
  streaming snapshot records. The principle of verifying the migrated state
  against the source via checksums rather than re-running the migration is
  standard practice in this domain and is adopted here.

- **Apache Kafka MirrorMaker**: While a different domain, Kafka's approach to
  cross-cluster migration — replicate, verify, cut over — is a useful model
  for thinking about the incremental migration path described in Future
  Enhancements. The idea of keeping the source system active while gradually
  building up the destination is well-tested in distributed systems.

# Testing
[testing]: #testing

Testing a migration framework is different from testing a running service.
The correctness bar is higher — data that enters incorrectly cannot be easily
fixed after the fact — and the failure modes are more varied. The strategy
below covers unit, integration, performance, chaos, and security dimensions.

**Unit tests**

Each component must have thorough unit test coverage before integration:

- Snapshot parser: synthetic snapshot directories with known state, covering
  empty namespaces, single-record namespaces, large namespaces, namespaces
  with CouchDB internal index keys (`\x00`-prefixed), and malformed records.
- Namespace classifier: all five filtering rules tested with representative
  inputs — system chaincodes, PDC hash namespaces (`$$h$$`), PDC private
  namespaces (`$$p$$`), implicit org collections, and valid user namespaces.
  Verify that the classifier is exhaustive and that no namespace falls through
  unclassified.
- Version converter: zero version, maximum block number (approaching int64
  limit), maximum `txNum`, values that exceed the conversion range, and
  round-trip consistency (convert then verify ordering is preserved).
- State file format: write a set of known records, read them back, assert
  byte-for-byte equality. Also test with an empty file, a single-record file,
  and a file with a truncated final record.
- Manifest: serialize and deserialize a manifest with every field populated,
  verify SHA-256 checksums match, verify `format_version` is validated on
  read.
- `StateImporter`: covered by
  [PR #516](https://github.com/hyperledger/fabric-x-committer/pull/516),
  which includes InitSchema, CreateNamespace, ImportNamespaceState,
  ImportPolicy, SetBlockHeight, VerifyImport, and multi-namespace end-to-end
  scenarios.

**Integration tests**

- **Full migration pipeline**: A Fabric peer with deterministic, known
  chaincode state (multiple namespaces, specific key-value pairs) → snapshot
  → export → bootstrap → verify → submit a Fabric-X transaction that reads
  migrated state and writes new state → confirm the transaction commits. This
  is the most important test: it proves the whole pipeline works end-to-end
  and that migrated state is immediately usable. Run against both PostgreSQL
  and YugabyteDB.
- **Large-scale import**: A synthetic canonical data set with 1M+ keys across
  multiple namespaces. Validates COPY protocol behavior under load, confirms
  memory usage stays bounded, and establishes baseline import throughput
  numbers that can be tracked over time.
- **Multi-namespace isolation**: Multiple chaincodes with overlapping key
  names. Confirms that namespace tables are fully isolated and a key in
  namespace A cannot be read from namespace B after migration.
- **PDC filtering**: A Fabric snapshot containing PDC hash namespaces,
  collection config history, and public state. Confirms PDC namespaces are
  excluded, public state is present, and the manifest accurately records what
  was excluded.
- **System chaincode filtering**: A snapshot from a network with active
  `_lifecycle` state. Confirms system chaincode namespaces do not appear in
  the canonical data set.
- **CouchDB snapshot**: Export from a CouchDB-backed peer, verify that
  internal index keys are stripped and do not appear in the committer database.
- **One-time guard**: Run `--init-from-snapshot` against a non-empty database.
  Confirm the committer exits with a clear error message and makes no changes
  to the existing data.

**Performance benchmarks**

- Measure export throughput (records/second, MB/second) for namespaces of
  varying sizes.
- Measure import throughput for the COPY-based bootstrap against PostgreSQL
  and YugabyteDB.
- Measure verification time (Level 1 and Level 2) for data sets of varying
  sizes.
- Track these numbers across releases to catch regressions.

**Chaos and failure tests**

- Corrupt a snapshot file (modify one byte after the SHA-256 is computed)
  and confirm the exporter detects and rejects it.
- Kill the bootstrap process midway through importing a namespace. Confirm
  that re-running after dropping and recreating the database completes
  successfully.
- Simulate a disk-full condition during export. Confirm the exporter fails
  cleanly with a useful error message and does not leave a partially written
  output directory that could be mistaken for a valid canonical data set.
- Submit a Fabric transaction using a historical Fabric transaction ID after
  migration. Confirm Fabric-X does not accept it (validates the TX format
  incompatibility reasoning).

**Compatibility tests**

- Snapshots from Fabric 2.5.x and Fabric 3.x peers — confirm both work.
- LevelDB and CouchDB state database types — confirm both produce correct output.
- Multi-organization snapshots — confirm that snapshots from different peers in
  the same channel produce consistent canonical data sets.

**Security tests**

- Confirm that the verification tool correctly detects a tampered canonical
  data set (single bit flip in a `.state` file).
- Confirm that the bootstrap refuses to run if the manifest's SHA-256
  checksums do not match the actual state files.
- Confirm that database credentials are not logged at any log level.

# Dependencies
[dependencies]: #dependencies

- `fabric-protos-go-apiv2` — for parsing Fabric snapshot `SnapshotRecord`
  protobuf messages and endorsement policy definitions.
- `fabric-x-committer` — the `StateImporter` component
  ([PR #516](https://github.com/hyperledger/fabric-x-committer/pull/516))
  provides the database bootstrap operations (schema initialization, COPY-based
  bulk loading, block height setting, verification).
- `fabric-x-common` — for shared protobuf definitions (`committerpb`,
  `MetaNamespaceID`, `ConfigNamespaceID`).
- `google.golang.org/protobuf` — for protobuf serialization/deserialization of
  `MigrationRecord` messages.
- `github.com/yugabyte/pgx/v5` — PostgreSQL/YugabyteDB driver, used by the
  verification tool to connect to the committer database.
- [RFC 0002 (Fabric-X SDK)](0002-client-sdk.md) — the SDK's Synchronizer and
  Parser components may be useful for post-migration transaction processing
  but are not a hard dependency for migration itself.

# Future enhancements
[future]: #future-enhancements

These ideas go beyond the initial scope of this RFC. None of them are blockers
for the first release, but they represent directions that could make the
migration story significantly stronger over time. Some are natural follow-ups;
others came up during design discussions and felt worth capturing while the
context is fresh.

## Incremental / Near-Zero-Downtime Migration

The biggest limitation of the current design is the maintenance window. For
networks where even a few minutes of downtime is unacceptable, we could build
a two-phase migration:

1. **Catch-up phase**: A background process tails new Fabric blocks as they
   get committed and streams the resulting state changes into the Fabric-X
   committer. The Fabric-X database stays a few blocks behind but is
   continuously converging.
2. **Cutover phase**: When the operator is ready, the Fabric network is
   briefly paused, the last remaining blocks are flushed, and Fabric-X takes
   over.

This needs a block-level export mechanism (not snapshot-based) and a
translation layer that can convert Fabric read-write sets into Fabric-X state
updates. It is a substantial piece of work, but it would bring the migration
window down from minutes to seconds — which matters a lot for high-availability
production networks.

## Private Data Migration

Right now we skip all private data because Fabric-X doesn't have PDCs. But if
that changes down the line, the migration framework should be ready. The
canonical data set format already supports extension — we could add a
`private_data/` directory alongside `namespaces/` and `policies/`. The harder
part is getting access to the actual private data: Fabric snapshots only
contain hashes, so the exporter would need to pull the real data from
organization-specific side databases or the peers' transient stores. Each org
would need to independently export and verify their private data, which adds
coordination complexity.

## Selective Namespace Migration

Not every chaincode on a Fabric channel may be relevant to the Fabric-X
deployment. An `--include-namespaces` or `--exclude-namespaces` flag on the
exporter would let operators cherry-pick which chaincodes to migrate. This is
useful when some chaincodes are being deprecated, or when the organization
wants to start fresh with certain parts of the state while preserving others.

## State Pruning During Migration

Migration is a natural opportunity to clean up stale data. A pruning mode
could allow operators to define rules (e.g., skip keys matching a pattern,
exclude keys with versions older than block N, drop keys with empty values)
that are applied during export. This would reduce the size of the canonical
data set and the bootstrapped database, which is especially helpful for
networks that have accumulated a lot of obsolete state over the years.

## Parallel Namespace Import

For very large migrations with tens of millions of keys spread across many
namespaces, importing one namespace at a time can be slow. Since each
namespace maps to an independent database table, there's no reason they can't
be imported in parallel — multiple goroutines each running their own COPY
stream against different tables. The main thing to watch is database
connection pool sizing and I/O contention, but for networks with dozens of
namespaces this could cut import time significantly.

## Migration Health Dashboard

A simple web UI or CLI dashboard that shows real-time progress during
migration would be valuable for operators managing production cutovers. It
could show:

- Per-namespace import progress (rows imported / total expected).
- Elapsed time and estimated time remaining.
- Database resource utilization (connections, disk I/O).
- Verification status as checks complete.

This could be as simple as a Prometheus endpoint on the committer that exposes
migration-specific metrics, with a Grafana dashboard template shipped alongside
the tool.

## Pre-Migration Compatibility Checker

Before operators invest time in a full export, a lightweight
`fabric-x-migrate check` command could analyze a Fabric snapshot and report
potential issues:

- Namespaces that use features not supported in Fabric-X.
- Keys or values that exceed Fabric-X's size limits (if any).
- Private data collections that will be lost.
- Version values that would overflow during conversion.
- Estimated export size and import time.

Think of it as a pre-flight checklist that gives operators confidence before
they start the actual migration.

## Namespace Renaming During Migration

Some organizations might want to rename chaincodes as part of the transition
(e.g., `legacy_supply_chain` becomes `supply_chain_v2`). A namespace mapping
file (JSON or YAML) that the exporter reads could define old-name to new-name
mappings. The canonical data set would use the new names, and the manifest
would record the mapping for auditability.

## Migration Audit Log for Compliance

Regulated industries (finance, healthcare, government) often need a detailed
audit trail for any data transformation. The exporter could produce a
comprehensive audit log that records:

- Every namespace included or excluded, with the rule that triggered the
  decision.
- Every key processed, with the original and converted version values.
- Timestamps for each phase of the export.
- Hash values at each stage for tamper detection.

This log would be a separate file from the manifest, potentially large, and
intended for compliance review rather than operational use.

## Canonical Data Set Archival

For disaster recovery, the canonical data set should be preserved long-term.
A built-in `fabric-x-migrate archive` command could package the data set into
a compressed, signed archive and upload it to object storage (S3, GCS, Azure
Blob). This gives organizations a point-in-time snapshot of their migrated
state that they can re-bootstrap from if the database is ever lost.

## Automated Migration Orchestration

For enterprise deployments with many channels and organizations, a
higher-level orchestration tool could coordinate the entire migration:

- Trigger and validate snapshots across peers from different organizations.
- Run exports and bootstraps in parallel for multiple channels.
- Aggregate verification results into a single migration report.
- Provide a rollback mechanism that restores Fabric operations if any
  channel's migration fails.

This is probably best implemented as a separate tool that calls
`fabric-x-migrate` and the committer under the hood, rather than building
all this logic into the core tools.

## Chaincode Logic Migration Companion

State migration is only half the story. Applications also need their
chaincode logic ported to Fabric-X. Once the
[Fabric-X SDK](0002-client-sdk.md) is mature enough, a companion guide or
tool could help developers:

- Translate Go/Java/Node.js Fabric chaincode into Fabric-X endorser views.
- Adapt existing contracts for the
  [EVM compatibility layer](0001-fabric-evm.md) where applicable.
- Map Fabric's shim API (GetState, PutState, GetHistoryForKey) to Fabric-X
  equivalents.

This is a separate effort, but it naturally pairs with state migration — you
need both your data and your logic on the new platform for the transition to
be complete.

## Reverse Migration (Fabric-X to Fabric)

While hopefully not needed often, the ability to export state from Fabric-X
back into a format that a Fabric peer can consume would provide a safety net
for organizations that need to roll back after migration. This would involve
generating a synthetic Fabric snapshot from the Fabric-X database, converting
versions back to `(BlockNum, TxNum)` format, and producing the required
metadata files. It's complex but would give organizations the confidence to
migrate knowing they have a way back.

## Multi-Peer Snapshot Agreement

The current design verifies the integrity of a single peer's snapshot against
its own `_snapshot_signable_metadata.json`. But in a multi-organization
network, each org runs their own peer, and those peers should all have the same
world state at a given block height. A `fabric-x-migrate validate-snapshot`
command could collect snapshot metadata from multiple peers across organizations
and confirm that the file hashes all match before anyone starts exporting. If
they don't match, there's a ledger divergence that needs to be investigated
before migration. This adds a meaningful layer of cross-org trust to the
process and is especially important in networks where organizations don't
fully trust each other.

## Delta Migration

For organizations that want to first migrate at block N and then later bring
their Fabric-X state up to date with block M (perhaps after a partial test
migration), a delta export would be useful. Rather than re-exporting everything
from scratch, the tool would export only the keys whose versions fall between
block N and block M, producing a smaller canonical data set that gets applied
on top of an already-bootstrapped database. This is also the building block
for the incremental near-zero-downtime migration described above.

## Canonical Data Set Compression and Encrypted Transfer

For large networks, the canonical data set can easily reach tens or hundreds
of gigabytes. A `fabric-x-migrate pack` command that compresses the data set
(using gzip or zstd), computes an HMAC over the archive, and optionally
encrypts it before transfer would make it much more practical to move the
data set between machines securely. A matching `fabric-x-migrate unpack`
command would verify the HMAC and decompress before use. The manifest's
SHA-256 checksums remain the source of truth for content integrity regardless
of how the archive is transferred.

## Chaincode API Compatibility Report

State migration moves data, but it doesn't tell developers which parts of
their chaincode will break on Fabric-X. A static analysis tool
(`fabric-x-migrate analyze-chaincode`) could scan chaincode source code and
flag Fabric shim API calls that have no Fabric-X equivalent — things like
`GetHistoryForKey`, `GetPrivateData`, `GetCreator`, `GetTransient`, and
channel event APIs. The output would be a structured report listing every
problematic call site with a suggested Fabric-X alternative where one exists.
Developers could run this before committing to a migration timeline and get a
realistic picture of the code changes required alongside the state migration.

## In-Progress Verification

For very large migrations, waiting until all namespaces are imported before
running verification can mean waiting a long time before knowing whether
something went wrong. An in-progress verification mode would verify each
namespace immediately after it finishes importing, before moving on to the
next one. If a namespace fails verification, the bootstrap stops early with
a clear report rather than wasting time importing the rest. This shortens
the feedback loop for large data sets and makes it easier to identify which
part of the data had a problem.

## Committer State Restore (Fabric-X to Fabric-X)

Separate from reverse migration, organizations need a way to restore a
Fabric-X committer from backup if the database is lost or corrupted. Since
the canonical data set is a complete representation of the world state at a
specific block height, a `fabric-x-migrate restore` command could wipe and
re-bootstrap a committer from its own canonical data set — entirely within the
Fabric-X ecosystem. Unlike the initial bootstrap which reads from a Fabric
snapshot, this restore would read from a previously exported Fabric-X
canonical data set. Combined with the archival enhancement above, this gives
operators a complete backup and recovery story for their Fabric-X deployment.

## Resumable Migration

The current bootstrap design is all-or-nothing: if it fails halfway, the
operator drops the database and starts over. For very large migrations this
is painful, especially if the failure happened while importing the last
namespace after hours of successful work. A resumable bootstrap would track
which namespaces have been successfully imported (using a progress file or a
dedicated database table), and on restart would skip those and continue from
where it left off. This requires handling partial imports carefully — a
namespace that was mid-import when the failure occurred would need to be
detected, its table dropped, and the import retried from the beginning for
that namespace. The one-time operation guard would need to be relaxed to
allow restart after a failed bootstrap (while still preventing restart after
a successful one).

## Signed Migration Attestation

In regulated industries — finance, healthcare, government — any transformation
of ledger data may need to be formally attested. A signed attestation document
would record the exact canonical data set used, the database it was imported
into, the block height, the verification results, and the identities of the
operators who performed each step. Each step would be signed using the
operator's Fabric-X identity (MSP certificate and private key). The result
is a legally defensible record that the migration was performed correctly by
authorized parties — the kind of artifact that satisfies auditors without
having to explain distributed ledgers from scratch.

## Custom State Validation Rules

Not all migrated state is necessarily clean. A Fabric network that has been
running for years might have accumulated malformed values, keys that violate
business rules, or data that was valid under old chaincode logic but is now
considered stale or invalid. A validation rule engine in the exporter would
let operators define predicates — in a YAML or JSON configuration file — that
are applied to each key-value pair during export:

```yaml
rules:
  - namespace: mycc
    key_pattern: "^order:.*"
    require_valid_json: true
  - namespace: mycc
    value_max_bytes: 1048576
    action: warn    # or: skip, fail
```

Rules can emit warnings, skip the offending record, or abort the export.
This turns migration into an opportunity to audit and clean up state before
it enters Fabric-X — something that is much harder to do after the fact.

## Key Analytics and Capacity Planning Report

Before committing to a full migration, operators need to understand what
they're dealing with. A `fabric-x-migrate analyze` command would scan the
snapshot and produce a detailed report without writing any output files:

- Total key count per namespace.
- Key size distribution (min, max, p50, p90, p99).
- Value size distribution.
- Version spread (how many distinct block heights are represented in the data).
- Estimated canonical data set size on disk.
- Estimated database footprint after import (with indexing overhead).
- Estimated import time based on benchmark data.
- A list of large namespaces that may benefit from parallel import.

This report gives operators the information they need to plan storage, allocate
time for the maintenance window, and decide whether any namespaces should be
excluded to reduce migration scope.

## Migration SDK (Go Library)

The `fabric-x-migrate` CLI is built for the common case, but every Fabric
deployment is different. Organizations with custom chaincode, non-standard
key structures, or specialized storage needs may need to build their own
migration tooling. A well-structured Go library exposing the core primitives
would make this much easier:

```go
// Open and verify a Fabric snapshot
snapshot, err := fabricmigrate.OpenSnapshot(dir)

// Iterate namespaces
for ns := range snapshot.Namespaces() {
    // Read records with automatic filtering and version conversion
    iter := ns.Records(fabricmigrate.DefaultFilter())
    for iter.Next() {
        key, value, version := iter.Key(), iter.Value(), iter.Version()
        // do something custom
    }
}
```

This is how the CLI itself would be built internally, so the library would
be a natural byproduct of the implementation rather than extra work. Exposing
it as a public API gives the community a foundation to build custom migration
tools, data validators, analytics pipelines, and anything else they need.

## Post-Migration Business Logic Simulation

Getting the data across is only half the problem. You also need confidence
that your chaincode logic behaves identically against the migrated state.
Before cutting over to Fabric-X in production, a simulation mode would let
operators replay a recorded set of test transactions against the migrated
Fabric-X database in read-only mode and compare the results against what
Fabric produced for the same inputs. Differences would surface business logic
incompatibilities before they affect real users. This is especially valuable
for complex chaincode that does non-trivial state reads as part of its logic.
The simulation doesn't need to actually commit anything — it just needs to
confirm that the read-write sets produced by the Fabric-X endorser match what
would have been produced on Fabric.

# Unresolved questions
[unresolved]: #unresolved-questions

These are the questions that should be resolved through the RFC process and
early implementation, before this feature is considered stable.

**Where should the exporter tool live?**

It could be a standalone `fabric-x-migrate` repository, a subcommand of
`fabric-x-tools`, or part of the committer repository. The standalone approach
is cleanest from a dependency perspective — the exporter needs Fabric protobuf
definitions that the committer should not carry — but it means one more
repository to maintain. Community feedback on this would be welcome before
implementation begins.

**How exactly are endorsement policies extracted from the snapshot?**

Fabric endorsement policies are stored inside the `_lifecycle` namespace as
part of the serialized chaincode definition. The exporter filters out
`_lifecycle` as a system namespace, but it needs to read policy data from it
before discarding it. The precise protobuf path through the chaincode
definition structure — and how to handle chaincodes that use the channel-level
default policy rather than an explicit one — needs to be worked out during
implementation. This is currently hand-waved in the Phase 4 section and needs
to be fully specified.

**Security review of the TX-ID decision.**

The case for not migrating transaction IDs rests on the assertion that Fabric
and Fabric-X transaction formats are incompatible enough that replay attacks
across the two platforms are not possible. This is a reasonable engineering
judgement, but it should be formally reviewed by a security expert before the
migration feature is promoted to stable status. The review should specifically
address whether any information from migrated state (key-value pairs, versions,
policies) could be exploited to craft a valid Fabric-X transaction.

**Should the bootstrap be resumable?**

The current design requires dropping and recreating the database on failure.
For large migrations this is wasteful. The resumable migration enhancement
describes how this could work, but it adds complexity to the one-time
operation guard logic. Should resumability be built into the initial
implementation, or deferred to a follow-up?

**What is the policy for the `__config` system namespace after migration?**

The `__config` namespace is created during schema initialization but left
empty. In a normal Fabric-X deployment this namespace holds the channel
configuration transaction. For a migrated network, is it expected to be
populated before the committer starts processing transactions? If so, what
does that configuration look like, and who is responsible for producing it?

**Should we enforce cross-org snapshot agreement before export?**

The multi-peer snapshot agreement enhancement describes a `validate-snapshot`
command that compares snapshot hashes across organizations. Should this be a
required step before export is permitted, or an optional pre-flight check?
Making it required adds safety but also adds coordination overhead. For
single-organization networks it would be an unnecessary requirement.

**How is the ordering service initialized for a migrated network?**

After bootstrap, the committer expects blocks starting from
`snapshotBlockNumber + 1`. The ordering service must be configured to produce
blocks starting from that height. The exact procedure depends on the ordering
service implementation and is outside the scope of this RFC, but it is a
critical operational gap that the broader Fabric-X documentation must address.
This RFC should cross-reference that documentation once it exists.

**Is there a maximum supported world state size?**

The design is streaming throughout, so there is no hard limit from a memory
perspective. But very large databases (hundreds of millions of keys) may
expose bottlenecks in the COPY protocol, the database's ability to build
indexes during import, or the verification tool's ability to scan counts
quickly. We need benchmark data at scale before declaring any size limits or
SLA expectations.
