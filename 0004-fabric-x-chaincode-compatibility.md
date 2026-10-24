---
layout: default
title: Fabric-X Chaincode Compatibility
nav_order: 3
---

- Feature Name: Fabric-X Chaincode Compatibility
- Start Date: 2026-10-14
- RFC PR:
- Fabric-X Component: fabric-x-chaincode-compatibility
- Fabric-X Issue:

# Summary
[summary]: #summary

This proposal introduces a compatibility layer that enables existing Hyperledger Fabric chaincodes using supported shim APIs to run on Fabric-X without modifying their business logic. It provides an execution environment that translates Fabric chaincode operations into Fabric-X transactions, while supporting multi-organization endorsement, chaincode lifecycle management, and pluggable orchestrator discovery. The goal is to simplify migration from Fabric to Fabric-X while preserving the existing chaincode programming model.

# Motivation
[motivation]: #motivation

Fabric-X introduces a modular transaction processing architecture designed for improved performance and scalability. However, unlike traditional Hyperledger Fabric, it does not provide a native chaincode execution environment. Organizations with existing Fabric applications therefore face significant migration challenges, as their chaincodes cannot be directly executed on Fabric-X without additional infrastructure or modifications.

The proposed compatibility layer addresses this challenge by enabling existing Fabric chaincodes to execute on Fabric-X with minimal modifications. This allows organizations to preserve their chaincode business logic while adopting Fabric-X's transaction processing model, reducing migration complexity and development effort.The Orchestrator may also expose a Fabric Gateway compatible client interface to minimize changes to existing applications, supporting query only evaluation and transaction submission workflows. This remains a design goal alongside support for custom client APIs.

The main objectives are:

- Run existing Go chaincodes using Chaincode-as-a-Service (CCaaS).
- Support multi-organization execution and endorsement.
- Provide Fabric compatible chaincode lifecycle operations.
- Allow pluggable discovery and integration with Fabric-X components.
- Support transaction coordination, duplicate detection, and finality tracking.

Together, these capabilities aim to simplify migration while preserving familiar Fabric workflows and leveraging Fabric-X's transaction processing model.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The compatibility layer allows developers to reuse existing Fabric chaincodes on Fabric-X while retaining familiar deployment and invocation workflows.

The architecture consists of the following components:

| Component | Responsibility |
|---|---|
| **Orchestrator** | Coordinates execution across organizations, collects endorsements, and submits transactions |
| **Compatibility Helper** | Handles Fabric shim communication and translates chaincode state requests into Fabric-X-compatible operations |
| **Chaincode Service** | Hosts and executes existing Fabric chaincodes as independent CCaaS services |
| **Lifecycle Manager** | Manages chaincode installation, organizational approvals, and committed definitions |
| **Discovery Resolver** | Locates and selects Orchestrator instances through configurable discovery mechanisms |

## Chaincode execution

Consider an existing Fabric application that uses chaincode to manage assets. With the compatibility layer, developers can deploy the same chaincode as an external service and continue invoking its existing functions.

For example, an asset-transfer operation can continue using familiar shim APIs such as `GetState`, `PutState`, and `DelState` to retrieve an asset, change its ownership, and update its state. Developers do not need to rewrite this business logic using Fabric-X-specific transaction APIs.

The compatibility layer handles the underlying interaction with Fabric-X, allowing supported chaincode operations to follow its transaction processing model.

## Multi-organization endorsement

Fabric applications may require multiple organizations to endorse a transaction. The compatibility layer preserves this approach without requiring developers to implement cross-organization execution themselves.

For example, an asset transfer requiring endorsements from Org0 and Org1 can be executed through their respective Orchestrators. The transaction proceeds only when both organizations produce consistent execution results and the required endorsements are obtained. If the results differ, the transaction is rejected.

From the application's perspective, the existing organizational endorsement requirements remain part of the transaction workflow.

## Chaincode lifecycle management

Developers can use familiar lifecycle operations such as `package`, `install`, `approveformyorg`, `checkcommitreadiness`, `commit`, and `querycommitted` to manage their chaincodes.

Once a chaincode definition is committed, newly started Orchestrators within the same organization can retrieve the committed chaincode definition without repeating the commitment process. This simplifies chaincode management across multiple instances.


## Lifecycle state and commitment

The Lifecycle Manager maintains organizational approvals in each Orchestrator's instance-local, in-memory store. During `checkcommitreadiness`, the initiating Orchestrator verifies approvals from participating organizations through RPC requests. These approvals are not persisted on the Fabric-X ledger or shared automatically across instances.

Once the required approvals are obtained, the initiating Orchestrator submits the committed chaincode definition as a Fabric-X transaction into a configurable lifecycle namespace (`lifecycle-namespace`, default `0`). Committed definitions are stored as ledger-backed metadata and reconstructed by Orchestrators during startup.

Application namespace creation and endorsement policy configuration remain separate Fabric-X operations. Approval checks are performed by the Lifecycle Manager, while Fabric-X validates the resulting lifecycle transaction according to the lifecycle namespace's policy.

## Orchestrator discovery and availability

Organizations are not restricted to a single Orchestrator instance. Multiple instances can be deployed to distribute workloads and improve service availability.

The compatibility layer provides a pluggable discovery interface that resolves these instances through static configuration or dynamic resolution, allowing multiple instances to be associated with the same organization.

In case of multi-Org Endorsement The initiating Orchestrator selects an instance from the resolved endpoints and forwards the transaction proposal for execution and endorsement. This enables flexible deployment and workload distribution without modifying existing chaincodes or endorsement requirements.

Each Orchestrator also maintains a transaction store that tracks transaction IDs and their execution states. This helps identify repeated requests and prevent unnecessary execution or submission.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The compatibility layer operates as an external service and integrates with Fabric-X through its existing interfaces and SDK. The initial implementation supports direct key-based state operations without modifying Fabric-X core components. Extending compatibility to additional Fabric chaincode APIs may require general purpose enhancements to Fabric-X services and transaction validation, developed in coordination with the Fabric-X community.

## Architectural overview

The architecture separates chaincode execution from Fabric-X transaction processing. Each Orchestrator contains an embedded compatibility helper that implements the Fabric chaincode shim communication protocol and communicates with external Chaincode-as-a-Service (CCaaS) runtimes.

![Fabric Chaincode Compatibility Architecture](./images/chaincode-compatibility-overview.png)

The design separates three main responsibilities:

- **Execution and endorsement:** The compatibility helper handles chaincode execution and state operations, producing results for endorsement.
- **Coordination:** Orchestrators coordinate execution across organizations, collect endorsements, and manage transaction submission and finality.
- **Discovery and transaction coordination:** Orchestrators resolve instances through configurable discovery mechanisms and track transaction progress locally to identify duplicate requests and manage execution states.

This separation allows the compatibility layer to reuse existing Fabric chaincode execution mechanisms without introducing the traditional Fabric peer architecture into Fabric-X core components.

## Execution and endorsement

When an Orchestrator receives a transaction proposal, it resolves the corresponding chaincode runtime and invokes it through the embedded compatibility helper.

The Orchestrator handles client requests through its client facing interface, with Fabric Gateway compatibility as a design goal. `Evaluate` executes chaincode without submitting a transaction, while `Endorse` collects the required endorsements, `Submit` forwards the signed transaction for ordering, and `CommitStatus` retrieves its final outcome through the Notification Service.

The helper processes Fabric shim messages and translates state requests into operations supported by Fabric-X. Direct key-based reads are retrieved from committed state through the Committer Query Service, recording each key's version or absence in the transaction's read set for MVCC validation. Writes and deletions are collected in the transaction's write set without modifying committed ledger state. Subsequent reads continue to retrieve committed values rather than the transaction's pending writes, preserving Fabric's execution semantics.

For multi-organization endorsement, the initiating Orchestrator resolves the required organizations and requests independent execution through their respective Orchestrators. The resulting execution responses and read-write sets are compared for consistency before endorsements are combined. Any disagreement prevents the transaction from proceeding to submission.

Each organization signs the Fabric-X namespace transaction contents, including the transaction ID, namespace information, read-write set, and transaction metadata. The chaincode response payload is included in the signed metadata, while response status and message are compared separately before endorsement aggregation.

The Fabric-X SDK provides reusable functionality for transaction construction, identity handling, and interaction with Fabric-X services. The resulting transaction is submitted to the Ordering Service, while its commit status is observed through the Notification Service.

Chaincode events generated through **SetEvent** are included in the endorsed transaction metadata using the Fabric-X SDK. The event name and payload are stored in **applicationpb.Tx.Metadata[0]** and carried into the ordered block. After receiving **COMMITTED** finality, the Orchestrator returns the event in the invocation response.

## Coordination

The initiating Orchestrator coordinates transaction execution and endorsement across participating organizations. It identifies the required organizations and resolves their Orchestrator instances through the discovery interface.

Inter-Orchestrator communication supports configurable mutually authenticated TLS (mTLS). Each participating Orchestrator independently verifies the original client's proposal signature, MSP identity, and invocation permissions against configured trust and authorization policies before execution and endorsement. These checks operate independently of endpoint discovery.

Each participating Orchestrator independently executes the chaincode against committed state obtained through its Query Service and returns its execution response, read-write set, and endorsement. The initiating Orchestrator compares these results and combines endorsements only when they agree; otherwise, submission is aborted. The compatibility layer does not enforce shared ledger snapshots across organizations, leaving final transaction validation and commitment to Fabric-X.

Once the endorsements are collected, the initiating Orchestrator constructs the Fabric-X transaction using the SDK and submits it to the Ordering Service. The transaction's final outcome is then observed through the Notification Service.

Endorsement requirements are configured as Fabric-X namespace policies, using Fabric's `SignaturePolicyEnvelope` representation within `NamespacePolicy.msp_rule`. This supports organization-level `AND`, `OR`, and `OutOf` policies. The initiating Orchestrator interprets these policies to coordinate execution and collect endorsements, while Fabric-X verifies endorsement signatures and enforces the namespace policy during transaction validation. Namespace policies are configured separately from chaincode lifecycle definitions.

Throughout this process, the Orchestrator maintains transaction progress in a local store to identify duplicate requests and track execution, submission, and finality.

## Discovery and transaction coordination

The discovery interface supports both statically configured endpoints and dynamic resolution of Orchestrator instances. Each organization can be associated with multiple endpoints, and resolved connections can be cached for subsequent requests.

This allows an Orchestrator to select among available instances when coordinating execution with other organizations. Endpoint selection, load balancing, and failover strategies can evolve independently of the chaincode execution logic.

To reduce redundant processing, the Orchestrator maintains a transaction store with the following states:

- `IN_PROGRESS`: Transaction execution has started.
- `EXECUTED`: Execution results have been collected.
- `SUBMITTED`: Transaction has been submitted for ordering.
- `COMPLETED`: A final transaction outcome has been observed.

These states help identify repeated requests and track execution progress. Additional recovery and synchronization mechanisms may be required when multiple Orchestrators process transactions concurrently or restart during execution.

If an Orchestrator or Query Service times out, requests may be redirected to another instance, subject to transaction recovery and consistency requirements. If required endorsements cannot be obtained, submission is aborted. Transaction IDs identify client retries to prevent duplicate processing. Submitted transactions without finality notifications remain SUBMITTED until their outcome is confirmed, without automatic re-submission.

## Compatibility limitations

- **State operations:** The initial implementation supports direct key-based `GetState`, `PutState`, and `DelState` operations using exact ledger keys.
- **Range queries and iterators:** Ordered traversal, partial composite-key queries, and pagination require Query Service extensions and Validator-Committer support for phantom read validation.
- **Advanced queries:** Rich queries and history queries remain outside the initial scope.
- **Private data:** Private data collections and their associated chaincode APIs are not supported.
- **Chaincode runtime:** The initial implementation supports Go chaincodes deployed as external Chaincode-as-a-Service (CCaaS) runtimes.

### Fabric shim API support

The following table summarizes the Fabric shim APIs supported by the current Go CCaaS implementation.

| Shim API | Current status |
|---|---|
| `GetState`, `PutState`, `DelState` | Supported |
| `GetArgs`, `GetStringArgs`, `GetFunctionAndParameters`, `GetArgsSlice` | Supported |
| `GetTxID`, `GetChannelID`, `GetTxTimestamp` | Supported |
| `GetCreator`, `GetTransient`, `GetSignedProposal`, `GetBinding`, `GetDecorations` | Supported |
| `CreateCompositeKey`, `SplitCompositeKey` | Supported for direct key operations |
| `SetEvent` | Supported |
| Range queries, partial composite-key queries, and pagination | Not supported |
| Rich queries and history queries | Not supported |
| Private data APIs | Outside initial scope  |
| `GetStateValidationParameter`, `SetStateValidationParameter` | Outside initial scope |
| `InvokeChaincode` (chaincode-to-chaincode invocation) | Outside initial scope |

Unsupported operations and additional compatibility features may be addressed in future iterations.


For further architectural details and implementation considerations, see the [Compatibility Layer Design Document](https://docs.google.com/document/d/1eseyFreUHmNmviQoq8qYGqatzMXgYZb79CFnlk6kBqg/edit?tab=t.0) (working draft).

# Drawbacks
[drawbacks]: #drawbacks

- **Communication overhead:** Additional chaincode execution and cross-organization communication are expected to increase transaction latency compared with native Fabric-X applications. The extent of this overhead will be evaluated through comparative latency and throughput benchmarks under equivalent workloads.
- **Runtime dependency:** The initial implementation supports only Chaincode-as-a-Service (CCaaS), requiring chaincodes to run as external services.
- **Maintenance overhead:** Maintaining compatibility with Fabric chaincode protocols introduces ongoing maintenance requirements.
- **Feature limitations:** Private data collections, advanced queries, and other unsupported Fabric operations require additional functionality or architectural changes.

# Rationale and alternatives
[alternatives]: #alternatives

- An alternative is to rewrite existing chaincodes as native Fabric-X applications. While this offers flexibility, it imposes significant migration costs.

- Another approach would be to integrate chaincode execution directly into Fabric-X core components. However, this would introduce a dependency on the traditional Fabric execution model.
  
- A third approach would be to adapt existing Fabric peers to execute chaincodes and interface with Fabric-X services. However, this would require maintaining additional peer infrastructure and adapting its transaction processing model.

The proposed approach keeps chaincode execution separate from Fabric-X, allowing existing applications to migrate while preserving the modular architecture of Fabric-X.

# Prior art
[prior-art]: #prior-art

- [Fabric Chaincode-as-a-Service](https://hyperledger-fabric.readthedocs.io/en/latest/cc_service.html) demonstrates how chaincode execution can be separated from the Fabric peer process. This proposal builds on that model while adapting chaincode execution to Fabric-X's transaction processing architecture.
- [Fabric Smart Client](https://github.com/hyperledger-labs/fabric-smart-client) provides a model for coordinating distributed transaction execution. The compatibility layer follows a different approach, focusing specifically on preserving Fabric chaincode execution and endorsement workflows.
- The [Fabric-X SDK](https://github.com/hyperledger/fabric-x-sdk) proposal introduces reusable building blocks for custom endorsers and identifies chaincode compatibility as an intended application. The compatibility layer builds on these components to provide a dedicated execution and coordination environment for existing Fabric chaincodes.

# Testing
[testing]: #testing

Beyond standard unit tests for individual components such as the Orchestrator, compatibility helper, and lifecycle manager, validating this proposal requires integration and end-to-end testing to ensure correctness and compatibility.

The primary goal is to demonstrate that existing Fabric chaincodes can execute on Fabric-X without modifying their business logic, while preserving consistent execution results, endorsement, and commit semantics. To achieve this, we will adopt two complementary strategies:

1. **Chaincode Replay and Differential Testing:** Execute representative chaincodes from the Fabric Samples repository on both traditional Hyperledger Fabric and Fabric-X through the compatibility layer, using identical inputs. Compare chaincode responses, final ledger states, and supported events to verify consistent execution behavior. Additionally, test multi-organization endorsement and MVCC validation.Initial testing will cover supported operations from Fabric Samples. Operations such as GetAllAssets, which depend on GetStateByRange, will be tested once range-query support becomes available.

2. **Lifecycle and Orchestrator Integration:** Test chaincode installation, organizational approvals, and ledger-backed definitions across multiple Orchestrators. Verify that newly started instances can reconstruct lifecycle state, discover other Orchestrators, and handle duplicate requests and transaction finality correctly.

3. **Performance Evaluation:** Compare transaction latency and throughput between native Fabric-X applications and the compatibility layer under equivalent workloads. Measure end-to-end transaction latency, including commit finality, and sustained throughput for single-organization and multi-organization execution to quantify the overhead introduced by the compatibility layer.

The success criteria are that existing Fabric chaincodes produce consistent execution results across participating organizations and that the resulting transactions are correctly validated and committed by Fabric-X.

These tests will run against traditional Hyperledger Fabric and Fabric-X networks, covering both normal execution and failure scenarios. The goal is to demonstrate that the compatibility layer provides a reliable migration path for existing Fabric applications without requiring changes to their chaincode business logic.

# Dependencies
[dependencies]: #dependencies

The compatibility layer depends on:

- Fabric chaincode shim and protobuf definitions.
- Fabric-X SDK for transaction construction, identity handling, and interaction with Fabric-X services.
- Fabric-X Committer Query Service.
- Fabric-X Ordering and Notification Services.
- gRPC for orchestrator and chaincode communication.

Additional Query Service and Validator-Committer capabilities will be required for complete range-query support and phantom-read validation.

# Unresolved questions
[unresolved]: #unresolved-questions

- Should Fabric's package and install lifecycle steps be simplified for CCaaS deployments?
- How should range queries and phantom-read validation be supported in Fabric-X?
- What additional coordination is required for reliable transaction recovery across multiple orchestrators?
- Which additional Fabric chaincode APIs should be prioritized for future compatibility?
