---
topic: "sei-k8s-controller: Kubernetes operator for Sei node management"
sources:
  - repo: sei-k8s-controller
    version: HEAD (local workspace)
    files:
      - CLAUDE.md
      - cmd/main.go
      - api/v1alpha1/groupversion_info.go
      - api/v1alpha1/seinode_types.go
      - api/v1alpha1/seinodegroup_types.go
      - api/v1alpha1/common_types.go
      - api/v1alpha1/networking_types.go
      - api/v1alpha1/monitoring_types.go
      - api/v1alpha1/validator_types.go
      - api/v1alpha1/archive_types.go
      - api/v1alpha1/full_node_types.go
      - api/v1alpha1/replayer_types.go
      - internal/controller/nodegroup/controller.go
      - internal/controller/nodegroup/nodes.go
      - internal/controller/nodegroup/genesis.go
      - internal/controller/nodegroup/labels.go
      - internal/controller/nodegroup/metrics.go
      - internal/controller/nodegroup/plan.go
      - internal/controller/nodegroup/predicates.go
      - internal/controller/nodegroup/status.go
      - internal/controller/nodegroup/networking.go
      - internal/controller/nodegroup/monitoring.go
      - internal/controller/node/controller.go
      - internal/controller/node/resources.go
      - internal/controller/node/labels.go
      - internal/controller/node/metrics.go
      - internal/controller/node/monitor.go
      - internal/controller/node/peers.go
      - internal/controller/node/plan_execution.go
      - internal/controller/node/sizing.go
      - internal/controller/observability/metrics.go
      - internal/planner/planner.go
      - internal/planner/bootstrap.go
      - internal/planner/full.go
      - internal/planner/archive.go
      - internal/planner/validator.go
      - internal/planner/replay.go
      - internal/planner/group.go
      - internal/planner/deployment.go
      - internal/planner/executor.go
      - internal/planner/constants.go
      - internal/planner/metrics.go
      - internal/task/task.go
      - internal/task/sidecar.go
      - internal/task/bootstrap.go
      - internal/task/config.go
      - internal/task/bootstrap_resources.go
      - internal/task/bootstrap_job.go
      - internal/task/bootstrap_service.go
      - internal/task/bootstrap_await.go
      - internal/task/bootstrap_teardown.go
      - internal/task/genesis.go
      - internal/task/genesis_peers.go
      - internal/task/deployment.go
      - internal/task/deployment_create.go
      - internal/task/deployment_await.go
      - internal/task/deployment_signal.go
      - internal/task/deployment_switch.go
      - internal/task/await_nodes_running.go
      - internal/task/fork.go
      - internal/task/fork_export.go
      - internal/platform/platform.go
      - internal/platform/objectstore.go
      - internal/platform/platformtest/config.go
      - manifests/role.yaml
      - config/manager/manager.yaml
      - config/monitoring/prometheus-rule.yaml
      - manifests/samples/seinode/pacific-1-full-node.yaml
      - manifests/samples/seinode/pacific-1-state-syncer.yaml
      - manifests/samples/seinode/pacific-1-snapshotter.yaml
      - manifests/samples/seinode/pacific-1-shadow-replayer.yaml
      - manifests/samples/seinodegroup/pacific-1-rpc-group.yaml
      - manifests/samples/seinodegroup/genesis-ceremony.yaml
      - manifests/samples/seinodegroup/fork-genesis-ceremony.yaml
verified: 2026-04-07
confidence: high
---

# sei-k8s-controller: Complete Architecture Reference

## 1. Overview

The sei-k8s-controller is a Kubernetes operator for managing Sei blockchain nodes. It is a single binary that runs two controllers:

- **SeiNodeGroup controller** -- fleet orchestration, genesis ceremonies, deployments, networking, monitoring
- **SeiNode controller** -- individual node lifecycle, StatefulSets, Services, PVCs, bootstrap progression

**API group:** `sei.io/v1alpha1`
**Framework:** controller-runtime v0.23.1 / kubebuilder v4.12.0
**Entry point:** `cmd/main.go`

The controller relies on a sidecar (`seictl serve`) that runs alongside every `seid` process. The controller drives the sidecar via HTTP task submission and polling. All config is environment-driven via `platform.Config` -- no defaults, no config files.

---

## 2. CRD Definitions

### 2.1 SeiNode

**Short name:** `snode`
**Print columns:** Phase, Age

#### SeiNodeSpec

Exactly one mode sub-spec must be set (enforced by CEL validation):

| Field | Type | Description |
|-------|------|-------------|
| `chainId` | string (required) | Chain ID this node belongs to |
| `image` | string (required) | seid container image |
| `peers` | `[]PeerSource` | How to discover network peers (EC2Tags, Static, or Label) |
| `entrypoint` | `*EntrypointConfig` | Override command/args for seid |
| `overrides` | `map[string]string` | Dotted TOML key paths for config (sei-config schema) |
| `sidecar` | `*SidecarConfig` | Image, port, resources for seictl sidecar |
| `podLabels` | `map[string]string` | Additional labels on StatefulSet pod template |
| `fullNode` | `*FullNodeSpec` | Chain-following full node |
| `archive` | `*ArchiveSpec` | Archive node (no pruning, full history) |
| `replayer` | `*ReplayerSpec` | Ephemeral replay workload |
| `validator` | `*ValidatorSpec` | Consensus-participating validator |

**Mode sub-specs:**

- **FullNodeSpec:** Optional `Snapshot` (S3 or StateSync), optional `SnapshotGeneration`
- **ArchiveSpec:** Optional `SnapshotGeneration` (always bootstraps via state sync internally)
- **ReplayerSpec:** Required `Snapshot` (must be S3), optional `ResultExport` for shadow comparison. Peers are required (CEL validation).
- **ValidatorSpec:** Optional `Snapshot`, optional `GenesisCeremony` (set by SeiNodeGroup controller)

**SnapshotSource** (union, exactly one of s3 or stateSync):
- `s3.targetHeight` -- block height to sync to
- `stateSync` -- Tendermint state sync from peers
- `trustPeriod` -- window for validator trust
- `backfillBlocks` -- historical blocks to fetch after restore
- `bootstrapImage` -- triggers bootstrap Job workflow (separate image for snapshot restore phase)

**PeerSource** (union, exactly one of ec2Tags, static, or label):
- `ec2Tags` -- AWS EC2 tag-based discovery (legacy)
- `static` -- fixed `nodeId@host:port` addresses
- `label` -- Kubernetes label selector on SeiNode resources, resolved to DNS hostnames by controller

#### SeiNodeStatus

| Field | Type | Description |
|-------|------|-------------|
| `phase` | SeiNodePhase | Pending, Initializing, Running, Failed, Terminating |
| `conditions` | `[]metav1.Condition` | Standard conditions |
| `plan` | `*TaskPlan` | Active initialization task sequence |
| `monitorTasks` | `map[string]MonitorTask` | Long-running sidecar tasks (snapshot upload, result export) |
| `resolvedPeers` | `[]string` | DNS hostnames from label-based peer discovery |
| `configStatus` | `*ConfigStatus` | Config version, mode, diagnostics, drift detection |

**SeiNodePhase state machine:**
```
Pending -> Initializing -> Running
                       \-> Failed
                       
Running (terminal unless deleted)
Failed (terminal -- must delete/recreate)
Any -> Terminating (on deletion)
```

**TaskPlan structure:**
- `ID` -- UUID v4 unique per plan instance
- `Phase` -- Active, Complete, Failed
- `Tasks[]` -- ordered PlannedTask list
- `FailedTaskIndex` / `FailedTaskDetail` -- diagnostics on failure

**PlannedTask:**
- `Type` -- task type string (e.g. "snapshot-restore", "config-apply")
- `ID` -- deterministic UUID v5 from planID/taskType/planIndex
- `Status` -- Pending, Complete, Failed
- `Params` -- opaque JSON payload
- `SubmittedAt`, `Error`, `MaxRetries`, `RetryCount`

### 2.2 SeiNodeGroup

**Short name:** `sng`
**Print columns:** Ready, Replicas, Phase, Revision (priority=1), Host (priority=1), Age

#### SeiNodeGroupSpec

| Field | Type | Description |
|-------|------|-------------|
| `replicas` | int32 (1-100, default 1) | Number of SeiNode instances |
| `template` | SeiNodeTemplate | SeiNodeSpec applied to each replica |
| `deletionPolicy` | DeletionPolicy | Delete (cascade) or Retain (orphan) |
| `genesis` | `*GenesisCeremonyConfig` | Genesis ceremony orchestration |
| `networking` | `*NetworkingConfig` | Traffic exposure (Service, Gateway, Isolation) |
| `monitoring` | `*MonitoringConfig` | ServiceMonitor for Prometheus |
| `updateStrategy` | `*UpdateStrategy` | BlueGreen or HardFork deployment |

**GenesisCeremonyConfig:**
- `chainId`, `stakingAmount`, `accountBalance` -- per-validator genesis params
- `accounts` -- non-validator genesis accounts
- `overrides` -- genesis parameter overrides (dotted key paths)
- `maxCeremonyDuration` -- timeout (default 15m)
- `fork` -- ForkConfig for forking from existing chain (sourceChainId, sourceImage, exportHeight)

**UpdateStrategy:**
- `type: BlueGreen` -- waits for entrant nodes to catch up (catching_up == false), then switches
- `type: HardFork` -- requires `haltHeight > 0`; stops old binary via SIGTERM at halt height, new binary continues

**NetworkingConfig:**
- `service` -- shared non-headless Service (ClusterIP, LoadBalancer, NodePort) with port selection
- `gateway` -- Kubernetes Gateway API HTTPRoute with parentRef, hostnames, annotations
- `isolation` -- Istio AuthorizationPolicy with allowed sources (principals, namespaces)

**MonitoringConfig:**
- `serviceMonitor` -- Prometheus Operator ServiceMonitor with interval and labels

#### SeiNodeGroupStatus

| Field | Type | Description |
|-------|------|-------------|
| `observedGeneration` | int64 | Last reconciled generation |
| `templateHash` | string | SHA-256 hash of deployment-worthy spec fields (first 16 hex chars) |
| `phase` | SeiNodeGroupPhase | Pending, Initializing, Ready, Upgrading, Degraded, Failed, Terminating |
| `replicas` / `readyReplicas` | int32 | Desired and ready counts |
| `nodes` | `[]GroupNodeStatus` | Per-child node name and phase |
| `plan` | `*TaskPlan` | Active group-level plan |
| `genesisHash` / `genesisS3URI` | string | Genesis ceremony output |
| `incumbentNodes` | `[]string` | Currently active SeiNode names |
| `deployment` | `*DeploymentStatus` | In-progress deployment metadata |
| `networkingStatus` | `*NetworkingStatus` | External service name, LB ingress |
| `conditions` | `[]metav1.Condition` | NodesReady, ExternalServiceReady, RouteReady, IsolationReady, ServiceMonitorReady, GenesisCeremonyComplete, PlanInProgress, GenesisCeremonyNeeded, ForkGenesisCeremonyNeeded |

**SeiNodeGroupPhase derivation:**
- PlanInProgress + Deployment != nil -> Upgrading
- PlanInProgress -> Initializing
- No nodes -> Pending
- All ready -> Ready
- All failed -> Failed
- Some failed, some ready -> Degraded
- Otherwise -> Initializing

---

## 3. Ownership Model

```
SeiNodeGroup (parent)
  |-- owns --> SeiNode (child, via controllerReference)
  |-- owns --> Service (external, shared)
  |-- owns --> HTTPRoute (unstructured, gateway.networking.k8s.io/v1)
  |-- owns --> AuthorizationPolicy (unstructured, security.istio.io/v1)
  |-- owns --> ServiceMonitor (unstructured, monitoring.coreos.com/v1)

SeiNode (parent)
  |-- owns --> StatefulSet (replicas=1)
  |-- owns --> Service (headless, per-node)
  |-- owns --> PersistentVolumeClaim (data volume)
  |-- owns --> Job (bootstrap, temporary)
```

**Naming conventions:**
- Child SeiNode: `{group-name}-{ordinal}` (0-indexed)
- External Service: `{group-name}-external`
- Headless Service: `{node-name}` (same name as SeiNode)
- Data PVC: `data-{node-name}`
- Bootstrap Job: `{node-name}-bootstrap`
- Entrant nodes during deployment: `{group-name}-g{generation}-{ordinal}`
- Exporter node (fork): `{group-name}-exporter`

**Labels:**
- `sei.io/nodegroup` -- group membership
- `sei.io/nodegroup-ordinal` -- ordinal within group
- `sei.io/revision` -- revision string (generation number)
- `sei.io/node` -- node name (on StatefulSet pods)
- `sei.io/role` -- "exporter", "rpc", "validator" etc.
- `sei.io/component` -- "bootstrap" (on Job pods)

**Deletion policies:**
- `Delete` (default) -- cascades deletion of child SeiNodes and networking resources
- `Retain` -- orphans children and networking resources by removing owner references

---

## 4. SeiNodeGroup Controller Reconciliation

**File:** `internal/controller/nodegroup/controller.go`
**Field owner:** `seinodegroup-controller`
**Finalizer:** `sei.io/seinodegroup-finalizer`
**Status poll interval:** 30s

**Reconcile loop:**

1. **Get SeiNodeGroup** -- exit if NotFound
2. **Handle deletion** -- if DeletionTimestamp set, set Terminating phase, apply deletion policy (Delete or Retain), cleanup metrics, remove finalizer
3. **Ensure finalizer** -- add if not present
4. **Snapshot status base** -- for optimistic locking on status patches
5. **reconcileSeiNodes** -- ensure child SeiNodes exist, scale down excess, populate incumbentNodes, detect deployment-needed, detect genesis-ceremony-needed
6. **reconcilePlan** -- drive active plan or build new plan (genesis or deployment)
7. **reconcileNetworking** -- external Service, HTTPRoute, AuthorizationPolicy
8. **reconcileMonitoring** -- ServiceMonitor
9. **updateStatus** -- count ready replicas, compute phase, set conditions
10. **Emit metrics** -- phase gauge, replica counts, condition gauges

**Watch setup:**
- `For: SeiNodeGroup` with GenerationChangedPredicate
- `Owns: SeiNode` with childPhaseChangedPredicate (only triggers on phase transitions, not task status patches)
- `Owns: Service`

### 4.1 SeiNode Reconciliation (within group)

**ensureSeiNode:** Creates or updates child SeiNodes. Compares labels, annotations, image, entrypoint, sidecar, podLabels. Does NOT use server-side apply for SeiNode management -- uses Get/Create/Update pattern.

**scaleDown:** Lists children by group label, deletes any with ordinal >= desired replicas.

**detectDeploymentNeeded:** Compares templateHash (covers chainId, image, entrypoint, sidecar image) against stored hash. When changed and updateStrategy is set, populates DeploymentStatus with incumbent/entrant revisions and node names.

**detectGenesisCeremonyNeeded:** Sets GenesisCeremonyNeeded or ForkGenesisCeremonyNeeded condition based on genesis config and fork config presence.

### 4.2 Networking Reconciliation

**External Service:** Uses server-side apply (SSA) with `seinodegroup-controller` field owner. Selector targets group label + revision label during deployments. Ports come from `seiconfig.NodePorts()`.

**HTTPRoute:** Unstructured SSA. References a shared Gateway parentRef. Backends route to the external Service on RPC port. Gracefully handles missing Gateway API CRDs (sets condition, emits warning event).

**AuthorizationPolicy:** Unstructured SSA. ALLOW policy matching group selector. Auto-injects controller SA principal (`SEI_CONTROLLER_SA_PRINCIPAL` env var) so controller-to-sidecar communication is never blocked. Handles missing Istio CRDs gracefully.

### 4.3 Monitoring Reconciliation

**ServiceMonitor:** Unstructured SSA. Matches group selector. Scrapes `metrics` port at configured interval. Handles missing Prometheus Operator CRDs gracefully.

---

## 5. SeiNode Controller Reconciliation

**File:** `internal/controller/node/controller.go`
**Field owner:** `seinode-controller`
**Finalizer:** `sei.io/seinode-finalizer`
**Status poll interval:** 30s

**Reconcile loop:**

1. **Get SeiNode** -- exit if NotFound
2. **Emit phase metric**
3. **Handle deletion** -- set Terminating phase, delete data PVC, cleanup metrics, remove finalizer
4. **Ensure finalizer**
5. **Resolve planner** -- ForNode() selects fullNodePlanner, archiveNodePlanner, replayerPlanner, or validatorPlanner
6. **Validate spec** -- mode-specific validation
7. **reconcilePeers** -- resolve label-based peers to DNS hostnames, write to status.resolvedPeers
8. **Ensure data PVC** -- Create-if-not-exists pattern (not SSA)
9. **Phase-based dispatch:**
   - **Pending** -> build plan, transition to Initializing
   - **Initializing** -> drive plan to completion, create StatefulSet/Service when appropriate, transition to Running or Failed
   - **Running** -> reconcile runtime tasks (monitor tasks)
   - **Failed** -> emit warning event, do nothing (terminal)

**Watch setup:**
- `For: SeiNode` with GenerationChangedPredicate
- `Owns: StatefulSet, Job, Service, PersistentVolumeClaim`

### 5.1 Resource Generation

**StatefulSet:**
- Replicas: 1 (always)
- ServiceName: node name (for headless Service)
- Pod annotation: `karpenter.sh/do-not-disrupt: "true"`
- ShareProcessNamespace: true (needed for sidecar signal handling)
- Init containers: `seid-init` (runs `seid init` with genesis.json guard), `sei-sidecar` (restartable init container with `RestartPolicy: Always`)
- Main container: `seid` with sidecar wait command wrapper
- Toleration: platform-configured key/value
- Node affinity: Karpenter nodepool selector
- Volume: data PVC

**Sidecar wait command:** The seid main container wraps its entrypoint in a bash polling loop that checks the sidecar's `/v0/healthz` endpoint via `/dev/tcp`. It blocks until HTTP 200 is returned (meaning all sidecar init tasks are complete and mark-ready has been called), then exec's seid. This is necessary because Kubernetes starts main containers as soon as restartable init containers are running, but the sidecar's /healthz returns 503 until initialization is complete.

**Headless Service:** ClusterIP: None, PublishNotReadyAddresses: true, all sei-config ports.

**Data PVC:** Storage class and size determined by node mode via platform config. AccessMode: ReadWriteOnce.

### 5.2 Sizing

Node mode determines resource sizing:
- **Archive:** platform.ResourceCPUArchive/ResourceMemArchive, platform.StorageClassPerf/StorageSizeArchive
- **Full/Validator:** platform.ResourceCPUDefault/ResourceMemDefault, platform.StorageClassPerf/StorageSizeDefault
- **Default:** platform.StorageClassDefault/StorageSizeDefault

### 5.3 Peer Resolution

The controller resolves label-based peer sources on every reconcile:
1. Lists SeiNode resources matching the label selector
2. Excludes self
3. Constructs stable DNS hostnames: `{name}-0.{name}.{namespace}.svc.cluster.local`
4. Sorts, deduplicates, writes to `status.resolvedPeers`
5. The planner reads resolvedPeers when building discover-peers task params

EC2Tags and Static sources bypass the controller -- they are passed directly to the sidecar's discover-peers task.

---

## 6. Plan System (Task Orchestration)

### 6.1 Architecture

The plan system is a generic, stateless task executor that drives ordered sequences of tasks to completion. It is parameterized by resource type `T` (SeiNode or SeiNodeGroup).

**Key types:**
- `NodePlanner` / `GroupPlanner` -- build task plans from resource state
- `PlanExecutor[T]` -- drives plans to completion via reconcile loop
- `TaskExecution` -- interface with Execute (submit), Status (poll), Err
- `ExecutionConfig` -- dependency injection bundle (sidecar client factory, kube client, scheme, platform config, object store)

**Execution flow per reconcile:**
1. Find first non-Complete task in plan
2. If task status is Pending, call Execute (submit to sidecar or perform controller action)
3. Call Status to poll current state
4. On Complete: patch task status, requeue immediately
5. On Failed: check MaxRetries; if retriable, reset to Pending with backoff; otherwise mark plan Failed
6. On Running: requeue after 5s poll interval

**Task ID generation:** Deterministic UUID v5 from `planID/taskType/planIndex`. This ensures the same task gets the same sidecar ID across controller restarts, enabling idempotent resubmission.

**Error semantics:**
- Plain error from Execute -> transient, will retry
- `Terminal(err)` -> permanent failure, fails the plan
- `UnknownTaskTypeError` -> permanent failure

**Retry backoff:** Exponential from 5s up to 30s max, formula: `5s * 2^min(attempt, 5)`

### 6.2 Task Registry

All task types are registered in `internal/task/task.go`:

**Sidecar tasks** (submitted to seictl HTTP API):
| Task Type | Fire-and-Forget | Description |
|-----------|----------------|-------------|
| `snapshot-restore` | No | Download and restore S3 snapshot |
| `configure-state-sync` | No | Set up Tendermint state sync config |
| `await-condition` | No | Wait for height/sync condition, optionally send SIGTERM |
| `config-apply` | No | Apply sei-config mode and overrides |
| `config-validate` | Yes | Validate config consistency |
| `configure-genesis` | No | Download genesis.json (from sei-config or S3) |
| `discover-peers` | No | Discover and set persistent_peers |
| `mark-ready` | Yes | Mark sidecar healthy (/healthz returns 200) |
| `generate-identity` | No | Generate validator identity (priv_validator_key, node_key) |
| `generate-gentx` | No | Generate genesis transaction |
| `upload-genesis-artifacts` | No | Upload gentx/node_key to S3 |
| `assemble-genesis` | No | Assemble final genesis.json from all node artifacts |
| `set-genesis-peers` | No | Set persistent_peers from genesis artifacts in S3 |
| `assemble-genesis-fork` | No | Fork genesis: rewrite chain ID, strip validators, collect-gentxs |

**Controller-side node tasks:**
| Task Type | Description |
|-----------|-------------|
| `deploy-bootstrap-service` | Create headless Service for bootstrap Job DNS |
| `deploy-bootstrap-job` | Create bootstrap Job with seid --halt-height |
| `await-bootstrap-complete` | Poll Job status until Complete or Failed |
| `teardown-bootstrap` | Delete bootstrap Job and Service |

**Controller-side group tasks:**
| Task Type | Description |
|-----------|-------------|
| `await-nodes-running` | Poll child SeiNodes until all reach Running |
| `collect-and-set-peers` | Collect node IDs via sidecar, patch peers on all nodes |
| `create-entrant-nodes` | Create entrant SeiNode resources for deployment |
| `submit-halt-signal` | Submit await-condition(SIGTERM) to incumbent sidecars |
| `await-nodes-at-height` | Poll entrant sidecars for block height |
| `await-nodes-caught-up` | Poll entrant sidecars until catching_up == false |
| `switch-traffic` | Update Service selector to entrant revision |
| `teardown-nodes` | Delete incumbent SeiNode resources |
| `create-exporter` | Create temporary SeiNode for state export (fork) |
| `await-exporter-running` | Wait for exporter to reach Running |
| `submit-export-state` | Submit export-state task to exporter sidecar |
| `teardown-exporter` | Delete exporter SeiNode |

### 6.3 Node Plan Progressions

**Base progressions** (defined in `planner.go`):

```
snapshot:   snapshot-restore -> config-apply -> config-validate -> mark-ready
state-sync: config-apply -> config-validate -> mark-ready
genesis:    config-apply -> config-validate -> mark-ready
```

These are then augmented with optional tasks:
- `configure-genesis` inserted before `config-apply` (always)
- `discover-peers` inserted before `config-validate` (when peers configured)
- `configure-state-sync` inserted before `config-validate` (when snapshot configured)

**Full node plan (S3 snapshot, with peers):**
```
snapshot-restore -> configure-genesis -> config-apply -> discover-peers -> configure-state-sync -> config-validate -> mark-ready
```

**Archive node plan (always state sync):**
```
configure-genesis -> config-apply -> discover-peers -> configure-state-sync -> config-validate -> mark-ready
```

**Replayer plan (S3 snapshot, requires peers):**
Same as full node plan. Replayer uses `ModeFull` for config-apply but adds custom pruning overrides and SC config.

**Validator plan (no snapshot, no peers):**
```
configure-genesis -> config-apply -> config-validate -> mark-ready
```

**Genesis ceremony validator plan:**
```
generate-identity -> generate-gentx -> upload-genesis-artifacts -> configure-genesis (with 180 retries) -> config-apply -> set-genesis-peers -> config-validate -> mark-ready
```

### 6.4 Bootstrap Plan (when BootstrapImage is set)

When a node has `snapshot.bootstrapImage` set, the plan uses a two-phase approach:

**Phase 1 -- Bootstrap infrastructure:**
```
deploy-bootstrap-service -> deploy-bootstrap-job
```

**Phase 2 -- Sidecar tasks on bootstrap Job pod:**
```
snapshot-restore -> configure-genesis -> config-apply -> discover-peers -> configure-state-sync -> config-validate
```
(No mark-ready -- the bootstrap Job runs seid with --halt-height instead)

**Phase 3 -- Wait and teardown:**
```
await-bootstrap-complete -> teardown-bootstrap
```

**Phase 4 -- Post-bootstrap config on StatefulSet pod:**
```
configure-genesis -> config-apply -> discover-peers -> config-validate -> mark-ready
```

The bootstrap Job uses the bootstrapImage for seid and the same sidecar image. It sets `hostname: {name}-0` and `subdomain: {name}` to get the same DNS name as the future StatefulSet pod. The seid wrapper waits for sidecar /healthz 200 before starting, then runs `seid start --halt-height N`. Exit code 130 (SIGINT from halt-height) is treated as success.

The StatefulSet and headless Service are only created AFTER teardown-bootstrap completes (to avoid RWO PVC conflicts).

### 6.5 Config Apply Parameters

Each planner builds mode-specific `ConfigApplyParams`:

**Full node:** Mode "full", controller overrides for concurrency_workers=500, pruning=custom, plus optional snapshot-generation overrides (snapshot_interval=2000, snapshot_keep_recent). User overrides take precedence.

**Archive:** Mode "archive", controller overrides for concurrency_workers=500, optional snapshot-generation overrides.

**Validator:** Mode "validator", user overrides only.

**Replayer:** Mode "full", controller overrides for concurrency_workers=500, custom pruning, and state-commit tuning (async_commit_buffer=100, memiavl snapshot settings).

---

## 7. Group-Level Plans

### 7.1 Genesis Ceremony Plan

Triggered when `GenesisCeremonyNeeded` condition is True and all replicas are created.

**Standard genesis:**
```
assemble-genesis (180 retries) -> collect-and-set-peers -> await-nodes-running
```

**Fork genesis (prepends exporter lifecycle):**
```
create-exporter -> await-exporter-running -> submit-export-state -> teardown-exporter -> assemble-genesis-fork (180 retries) -> collect-and-set-peers -> await-nodes-running
```

**How it works:**
1. Each child SeiNode independently runs its genesis plan (generate-identity, generate-gentx, upload-artifacts)
2. The child's configure-genesis task retries up to 180 times (30 min) waiting for the group to assemble genesis
3. The group plan runs assemble-genesis on one child's sidecar, which collects all artifacts from S3 and produces final genesis.json
4. collect-and-set-peers queries each node's sidecar for its Tendermint node ID, then patches static peers onto all nodes
5. await-nodes-running polls until all children reach Running phase

### 7.2 Blue-Green Deployment Plan

Triggered when templateHash changes and `updateStrategy.type: BlueGreen`.

```
create-entrant-nodes -> await-nodes-running -> await-nodes-caught-up -> switch-traffic -> teardown-nodes
```

1. Create new SeiNodes named `{group}-g{generation}-{ordinal}` with entrant revision label
2. Wait for all entrant nodes to reach Running
3. Poll sidecar `/status` until all report Ready (catching_up == false)
4. Patch DeploymentStatus.IncumbentRevision to entrant revision (Service selector follows)
5. Delete old incumbent SeiNodes

### 7.3 Hard Fork Deployment Plan

Triggered when templateHash changes and `updateStrategy.type: HardFork`.

```
create-entrant-nodes -> await-nodes-running -> submit-halt-signal -> await-nodes-at-height -> switch-traffic -> teardown-nodes
```

1. Create entrant nodes (new binary)
2. Wait for entrant nodes to reach Running
3. Submit `await-condition(height=H, action=SIGTERM)` to incumbent sidecars (fire-and-forget)
4. Poll entrant sidecars until they reach the halt height
5. Switch traffic
6. Delete incumbents

---

## 8. Sidecar Interaction Model

### 8.1 Communication

The controller communicates with the sidecar via HTTP:
- **URL:** `http://{name}-0.{name}.{namespace}.svc.cluster.local:{port}`
- **Default port:** 7777 (from `seiconfig.PortSidecar`)
- **Client:** `seictl/sidecar/client.SidecarClient`

**Key endpoints:**
- `POST /v0/tasks` -- Submit a task (with optional pre-assigned UUID)
- `GET /v0/tasks/{id}` -- Get task status and result
- `GET /v0/livez` -- Liveness probe
- `GET /v0/healthz` -- Readiness probe (returns 503 until mark-ready)
- `GET /v0/status` -- Node status including catching_up flag
- `GET /v0/node-id` -- Tendermint node ID

### 8.2 Task Submission

Tasks are submitted with a `TaskRequest`:
```go
type TaskRequest struct {
    Id     *uuid.UUID       // Pre-assigned deterministic ID
    Type   string           // Task type
    Params *map[string]any  // Task-specific parameters
}
```

The controller pre-assigns deterministic UUIDs (UUID v5) so that:
- Resubmitting the same task is idempotent
- The sidecar's cloud-API model transparently re-executes failed tasks when resubmitted with the same ID
- Task IDs survive controller restarts

### 8.3 Fire-and-Forget vs Polled Tasks

- **Fire-and-forget:** `config-validate`, `mark-ready` -- Execute succeeds, Status immediately returns Complete
- **Polled:** All other sidecar tasks -- Execute submits, Status polls via GetTask until Completed/Failed

### 8.4 Monitor Tasks (Runtime)

After a node reaches Running phase, the controller submits long-running monitor tasks:
- **snapshot-upload:** For nodes with SnapshotGeneration config. Sidecar handles upload on its own schedule.
- **result-export:** For replayer nodes with ResultExport config. Sidecar compares local block results against canonical RPC; completes when divergence detected.

Monitor tasks are tracked in `status.monitorTasks` (keyed by task type). They are submitted exactly once and polled on each reconcile. Terminal states trigger Conditions and Events.

---

## 9. Platform Config

All configuration is environment-driven. The `platform.Config` struct has **no defaults** -- every field must be set via environment variable.

| Env Var | Field | Purpose |
|---------|-------|---------|
| `SEI_NODEPOOL_NAME` | NodepoolName | Karpenter nodepool for node affinity |
| `SEI_TOLERATION_KEY` | TolerationKey | Toleration key for node pods |
| `SEI_TOLERATION_VALUE` | TolerationVal | Toleration value |
| `SEI_SERVICE_ACCOUNT` | ServiceAccount | Pod service account name |
| `SEI_STORAGE_CLASS_PERF` | StorageClassPerf | High-performance storage class (full/archive/validator) |
| `SEI_STORAGE_CLASS_DEFAULT` | StorageClassDefault | Default storage class |
| `SEI_STORAGE_SIZE_DEFAULT` | StorageSizeDefault | PVC size for full/validator nodes |
| `SEI_STORAGE_SIZE_ARCHIVE` | StorageSizeArchive | PVC size for archive nodes |
| `SEI_RESOURCE_CPU_ARCHIVE` | ResourceCPUArchive | CPU request for archive mode |
| `SEI_RESOURCE_MEM_ARCHIVE` | ResourceMemArchive | Memory request for archive mode |
| `SEI_RESOURCE_CPU_DEFAULT` | ResourceCPUDefault | CPU request for full/validator mode |
| `SEI_RESOURCE_MEM_DEFAULT` | ResourceMemDefault | Memory request for full/validator mode |
| `SEI_SNAPSHOT_BUCKET` | SnapshotBucket | S3 bucket for snapshots |
| `SEI_SNAPSHOT_REGION` | SnapshotRegion | S3 region for snapshots |
| `SEI_RESULT_EXPORT_BUCKET` | ResultExportBucket | S3 bucket for result exports |
| `SEI_RESULT_EXPORT_REGION` | ResultExportRegion | S3 region for result exports |
| `SEI_RESULT_EXPORT_PREFIX` | ResultExportPrefix | S3 key prefix for result exports |
| `SEI_GENESIS_BUCKET` | GenesisBucket | S3 bucket for genesis artifacts |
| `SEI_GENESIS_REGION` | GenesisRegion | S3 region for genesis artifacts |
| `SEI_CONTROLLER_SA_PRINCIPAL` | (on group reconciler) | SPIFFE principal for AuthorizationPolicy |

**Production values (from manager.yaml):**
- Nodepool: `sei-node`, Toleration: `sei.io/workload=sei-node`
- SA: `seid-node`
- Storage: `gp3-10k-750` (perf), `gp3` (default), 2000Gi (default), 4000Gi (archive)
- Resources: 4 CPU/32Gi (default), 16 CPU/256Gi (archive)
- Sidecar default image: pinned sha256 digest of seictl

---

## 10. RBAC Permissions

Generated from `+kubebuilder:rbac:` markers. The ClusterRole `manager-role` requires:

| API Group | Resources | Verbs |
|-----------|-----------|-------|
| `""` | events | create, patch |
| `""` | persistentvolumeclaims, services | full CRUD |
| `""` | pods | get, list, watch |
| `apps` | statefulsets | full CRUD |
| `batch` | jobs | full CRUD |
| `gateway.networking.k8s.io` | httproutes | full CRUD |
| `monitoring.coreos.com` | servicemonitors | full CRUD |
| `security.istio.io` | authorizationpolicies | full CRUD |
| `sei.io` | seinodegroups, seinodes | full CRUD |
| `sei.io` | seinodegroups/finalizers, seinodes/finalizers | update |
| `sei.io` | seinodegroups/status, seinodes/status | get, patch, update |

Additional RBAC in `config/rbac/`:
- Leader election Role + RoleBinding
- ClusterRoleBinding to service account

---

## 11. Observability

### 11.1 Prometheus Metrics

**SeiNodeGroup controller:**
- `sei_controller_seinodegroup_phase` -- gauge per phase (1=active)
- `sei_controller_seinodegroup_replicas` -- gauge for desired/ready counts
- `sei_controller_seinodegroup_condition` -- gauge per condition type and status
- `sei_controller_seinodegroup_reconcile_substep_duration_seconds` -- histogram per substep

**SeiNode controller:**
- `sei_controller_seinode_phase` -- gauge per phase
- `sei_controller_seinode_phase_transitions_total` -- counter from/to
- `sei_controller_seinode_init_duration_seconds` -- histogram (Pending to Running)
- `sei_controller_seinode_last_init_duration_seconds` -- gauge per node
- `sei_controller_sidecar_unreachable_total` -- counter
- `sei_controller_monitor_task_completed_total` -- counter per task type and reason
- `sei_controller_monitor_task_status` -- gauge per task type and status

**Plan executor:**
- `sei_controller_task_retries_total` -- counter per task type
- `sei_controller_task_failures_total` -- counter per task type
- `sei_controller_plan_active` -- gauge (1=plan in progress)

**Shared:**
- `sei_controller_reconcile_errors_total` -- counter per controller

### 11.2 PrometheusRule Alerts

| Alert | Condition | Severity |
|-------|-----------|----------|
| SeiNodeFailed | phase=="Failed" for 1m | critical |
| SeiNodeGroupDegraded | phase=="Degraded" for 10m | warning |
| SeiNodeGroupFailed | phase=="Failed" for 5m | critical |
| SeiNodeStuckInitializing | Initializing for 30m with no active plan | warning |
| SeiNodeStuckPending | Pending for 15m | warning |
| SidecarUnreachableHigh | rate > 0.1 for 10m | warning |
| ControllerReconcileErrors | >5 errors in 15m for 5m | warning |
| ControllerHighReconcileLatency | p99 substep > 10s for 10m | warning |
| TaskFailureRateHigh | >3 failures in 15m for 5m | warning |
| ControllerMetricsDown | metrics endpoint down for 5m | critical |

### 11.3 Kubernetes Events

Events are emitted for:
- SeiNode creation/deletion
- Phase transitions
- Plan start/complete/fail
- Monitor task complete/fail
- Networking resource reconciliation (HTTPRoute, AuthorizationPolicy, ServiceMonitor)
- CRD not installed warnings
- Deletion policy actions (retain/delete)

---

## 12. Server-Side Apply (SSA)

Used for resources that need drift correction:
- **SeiNode controller:** StatefulSet, headless Service -- field owner `seinode-controller`
- **SeiNodeGroup controller:** External Service, HTTPRoute, AuthorizationPolicy, ServiceMonitor -- field owner `seinodegroup-controller`

Both use `client.Apply` with `client.ForceOwnership` to handle conflicts.

Data PVC and SeiNode CRs use traditional Get/Create/Update patterns because PVC specs are immutable after creation and SeiNode spec changes need explicit field-by-field comparison.

---

## 13. Key Implementation Details

### Genesis Resolution
The sidecar resolves genesis autonomously. For well-known chains (pacific-1, atlantic-2, etc.), genesis is embedded in sei-config. For custom chains, the sidecar falls back to `{SEI_GENESIS_BUCKET}/{chainID}/genesis.json` in S3.

### Config Schema
Config keys use the sei-config unified schema (dotted paths like `evm.http_port`, `storage.pruning`). The sidecar's config-apply task translates these to seid's native `config.toml` format (which uses hyphens, e.g. `persistent-peers`, `trust-height`).

### Process Namespace Sharing
`shareProcessNamespace: true` is set on all node pods. This allows the sidecar to send signals (SIGTERM, SIGINT) to the seid process during halt-height scenarios.

### Bootstrap Guard
The seid-init container checks for existing `genesis.json` before running `seid init`. This prevents re-initialization of pre-populated volumes (e.g., from genesis ceremonies or previous bootstrap Jobs).

### Karpenter Integration
All node pods have `karpenter.sh/do-not-disrupt: "true"` annotation and node affinity targeting the configured Karpenter nodepool.

### Sidecar as Restartable Init Container
The sidecar runs as an init container with `RestartPolicy: Always` (Kubernetes sidecar container pattern). This ensures it starts before and survives longer than the main seid container, while Kubernetes manages its lifecycle.

---

## 14. Sample Manifests

### Standalone SeiNode Examples

**Full node (S3 snapshot bootstrap):**
```yaml
spec:
  chainId: pacific-1
  image: "ghcr.io/sei-protocol/sei:v6.3.0"
  peers:
    - ec2Tags: {region: eu-central-1, tags: {ChainIdentifier: pacific-1, Component: state-syncer}}
  fullNode:
    snapshot:
      s3: {targetHeight: 198740000}
      trustPeriod: "9999h0m0s"
```

**Archive snapshotter (state sync bootstrap, produces snapshots):**
```yaml
spec:
  chainId: pacific-1
  image: "ghcr.io/sei-protocol/sei:v6.3.0"
  peers:
    - ec2Tags: {region: eu-central-1, tags: {ChainIdentifier: pacific-1, Component: state-syncer}}
  archive:
    snapshotGeneration: {keepRecent: 5}
```

**Shadow replayer (bootstrap Job, result export with divergence detection):**
```yaml
spec:
  chainId: pacific-1
  image: ghcr.io/bdchatham/sei-shadow@sha256:...
  entrypoint: {command: ["seid"], args: ["start", "--home", "/sei", "--skip-app-hash-validation"]}
  overrides: {giga_executor.enabled: "true", giga_executor.occ_enabled: "true", mempool.size: "0"}
  peers:
    - ec2Tags: {region: eu-central-1, tags: {ChainIdentifier: pacific-1, Component: state-syncer}}
  replayer:
    snapshot:
      s3: {targetHeight: 200940000}
      bootstrapImage: "ghcr.io/sei-protocol/sei:v6.3.0"
      trustPeriod: "9999h0m0s"
    resultExport:
      canonicalRpc: "http://pacific-1-archive-0.pacific-1-archive.default.svc.cluster.local:26657"
```

### SeiNodeGroup Examples

**RPC fleet (3 replicas, networking, monitoring):**
```yaml
spec:
  replicas: 3
  template:
    spec:
      chainId: pacific-1
      image: "ghcr.io/sei-protocol/sei:v6.3.0"
      fullNode:
        snapshot: {s3: {targetHeight: 198740000}, trustPeriod: "9999h0m0s"}
  networking:
    service: {type: ClusterIP}
    gateway:
      parentRef: {name: sei-gateway, namespace: istio-system}
      hostnames: ["rpc.pacific-1.sei.io"]
    isolation:
      authorizationPolicy:
        allowedSources:
          - principals: ["cluster.local/ns/istio-system/sa/sei-gateway-istio"]
  monitoring:
    serviceMonitor: {interval: "30s", labels: {release: prometheus}}
```

**Genesis ceremony (4 validators):**
```yaml
spec:
  replicas: 4
  genesis:
    chainId: genesis-test-1
    stakingAmount: "10000000usei"
    accountBalance: "1000000000000000000000usei,..."
  template:
    spec:
      chainId: genesis-test-1
      image: "ghcr.io/sei-protocol/sei:v6.3.0"
      validator: {}
```

**Fork genesis ceremony:**
```yaml
spec:
  replicas: 3
  genesis:
    chainId: fork-test-1
    stakingAmount: "10000000usei"
    accountBalance: "..."
    fork:
      sourceChainId: pacific-1
      sourceImage: "ghcr.io/sei-protocol/sei:v6.3.0"
      exportHeight: 200950000
  template:
    spec:
      chainId: fork-test-1
      image: "ghcr.io/sei-protocol/sei:v6.3.0"
      validator: {}
```

---

## 15. Manager Deployment

Runs as a single-replica Deployment with:
- Leader election (`bc1f5b0a.sei.io`)
- Health/ready probes on :8081
- Metrics on :8080
- Security context: non-root, read-only root filesystem, no privilege escalation
- Resources: 50m CPU, 128Mi memory
- All platform config via env vars
