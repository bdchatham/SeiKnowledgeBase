---
topic: "Networking design rationale: Gateway listeners, protocol routing, and hostname conventions"
sources:
  - repo: sei-k8s-controller
    version: HEAD
    files:
      - api/v1alpha1/networking_types.go
      - internal/controller/nodedeployment/networking.go
  - repo: slanders (sei-protocol/slanders)
    version: main branch
    files:
      - seienv/README.md
  - external:
      - sei-infra Terraform (ALB/target group patterns)
      - Alchemy, Infura, QuickNode public API patterns
      - Cosmos ecosystem validator endpoint conventions
verified: 2026-04-09
confidence: high
---

## Overview

Design decisions for how Sei node services are exposed publicly via the Kubernetes Gateway API. Documents why certain patterns were chosen, what was rejected, and the planned evolution from single-node to multi-deployment support.

## Key Design Decisions

### 1. Gateway Owns Protocol Topology, Deployments Just Attach

The Gateway (platform-managed) defines listeners per protocol with wildcard hostname patterns. Deployments select which listeners to attach to. The deployment name is prepended to the listener's hostname pattern for uniqueness.

```
Gateway listeners (platform-level):
  evm   → *.evm.prod.platform.sei.io    :443 HTTPS
  rpc   → *.rpc.prod.platform.sei.io    :443 HTTPS
  rest  → *.rest.prod.platform.sei.io   :443 HTTPS
  grpc  → *.grpc.prod.platform.sei.io   :443 HTTPS

Deployment "pacific-1" attaches to [evm, rpc]:
  pacific-1.evm.prod.platform.sei.io  → Service :8545
  pacific-1.rpc.prod.platform.sei.io  → Service :26657

Deployment "arctic-1" attaches to [evm, rpc, rest, grpc]:
  arctic-1.evm.prod.platform.sei.io   → Service :8545
  arctic-1.rpc.prod.platform.sei.io   → Service :26657
  arctic-1.rest.prod.platform.sei.io  → Service :1317
  arctic-1.grpc.prod.platform.sei.io  → Service :9090
```

**Why:** Deployment names like `rpc` in `rpc.pacific-1.sei.io` conflate the deployment identity with the protocol. Multiple RPC deployments would collide. Separating protocol from deployment name allows unlimited deployments per chain.

### 2. EVM HTTP and WebSocket Share One Listener

EVM JSON-RPC (8545) and EVM WebSocket (8546) are merged into a single `evm` listener. Envoy handles the WebSocket upgrade natively on the same HTTPRoute.

**Why:** Every major EVM provider (Alchemy, Infura, QuickNode) uses one URL for both HTTP and WebSocket. Wallets/dApps use `https://` for requests and `wss://` for subscriptions on the same hostname. Separate subdomains for HTTP vs WS is non-standard and confusing.

**Result:** 4 listeners (evm, rpc, rest, grpc) instead of 5 (evm-rpc, evm-ws, rpc, rest, grpc).

### 3. Path-Based Routing Rejected

Considered and rejected: serving all protocols on one hostname with path prefixes (`/rpc`, `/rest`, `/evm`).

**Why rejected:**
- gRPC clients construct paths from protobuf definitions (`/cosmos.bank.v1beta1.Query/Balance`). Adding a path prefix breaks every gRPC client.
- CometBFT RPC has its own paths (`/status`, `/block`). Nesting under `/rpc/status` breaks client expectations.
- No standard tooling supports custom base paths for these protocols.

### 4. Listeners Derived from Node Mode, Not CRD Config

The controller derives which listeners a deployment needs from the node mode via `seiconfig.NodePortsForMode()`. No manual listener/protocol selection in the CRD.

| Mode | Listeners |
|------|-----------|
| Validator | (none — validators don't serve public traffic) |
| Full / Archive | evm, rpc, rest, grpc |

**Why:** The node mode already determines which services are enabled. Exposing protocol selection in the CRD is unnecessary complexity — the controller knows which ports are active.

**Current state:** Single-node deployments only. The listener selection is implicit. When multi-deployment support is added, an optional `listeners` field on the CRD will allow overriding the defaults (e.g., an EVM-only fleet that only needs the `evm` listener).

### 5. TLS is the Gateway's Responsibility

The controller does not configure TLS. The Gateway's HTTPS listeners handle TLS termination with wildcard certs. The controller targets a specific listener via `SEI_GATEWAY_SECTION_NAME` (optional platform env var).

**Why:** TLS certs, protocols, and cipher suites are infrastructure concerns that vary per environment. Putting them in the CRD would couple deployment config to platform security policy.

### 6. WebSocket Connection Lifecycle

WebSocket connections (EVM `eth_subscribe`) are long-lived and pinned to a specific backend pod. During deployments:
- The Service selector flip (SwitchTraffic) breaks existing WebSocket connections
- Clients must reconnect — there is no way to "move" a WebSocket connection
- `terminationGracePeriodSeconds` on the StatefulSet gives clients time to reconnect
- Every EVM WebSocket library (ethers.js, web3.js) implements automatic reconnection

**This is an inherent WebSocket limitation**, not specific to this architecture. The same behavior occurs with AWS ALB, Cloudflare, and every other reverse proxy.

## Legacy Pattern (sei-infra) and Why It Changed

The legacy EC2 infrastructure used 5 subdomains per chain on a single ALB:
```
rpc.pacific-1.sei.io     → ALB :443 → target group :26657
rest.pacific-1.sei.io    → ALB :443 → target group :1317
grpc.pacific-1.sei.io    → ALB :443 → target group :9090
evm-rpc.pacific-1.sei.io → ALB :443 → target group :8545
evm-ws.pacific-1.sei.io  → ALB :443 → target group :8546
```

**Problems with this pattern:**
- `rpc` is a deployment name, not a protocol — second RPC fleet would collide
- `evm-rpc` and `evm-ws` as separate subdomains is non-standard for EVM ecosystem
- Protocol names leak into the public API surface
- Tightly couples deployment identity to protocol routing

## Platform Configuration (Environment Variables)

| Env Var | Purpose | Required |
|---------|---------|----------|
| `SEI_GATEWAY_NAME` | Gateway resource name | Yes |
| `SEI_GATEWAY_NAMESPACE` | Gateway resource namespace | Yes |
| `SEI_GATEWAY_SECTION_NAME` | Target a specific listener (e.g., "https") | No (attaches to all compatible listeners when empty) |

## Future Work

- **Multi-deployment support:** Add optional `listeners` field to CRD for overriding mode-derived defaults
- **GRPCRoute:** When Gateway API GRPCRoute support matures, generate GRPCRoute instead of HTTPRoute for the grpc listener
- **Traffic splitting:** Weighted routing between deployments for canary upgrades (Gateway API supports this natively via HTTPRoute backendRef weights)

## Key Takeaways

- Protocol routing is a Gateway concern, not a deployment concern
- Deployment names provide hostname uniqueness, not protocol names
- EVM HTTP + WebSocket = one listener (industry standard)
- 4 protocol listeners: evm, rpc, rest, grpc
- No path-based routing (gRPC incompatibility)
- TLS handled by Gateway, not controller
- WebSocket connections break on deployment — clients reconnect (inherent limitation)
- Listeners derived from node mode — no CRD config needed for single deployments
