# Investigation: Multicluster Global Hub and OCM Fleet

> **Status: Exploration / proposal.** This document records an architectural
> direction to investigate. It is not an approved design, an implementation
> plan, or a claim that the proposed integration exists today.

## Purpose

Explore how Multicluster Global Hub (Global Hub) and OCM Fleet could complement
one another to provide a unified fleet-management experience while continuing
to use existing Open Cluster Management (OCM) hubs. This is not a proposal to
replace OCM or to require each OCM hub to become a Fleet instance.

The investigation focuses on four questions:

1. How can inventoried managed clusters become Fleet delivery targets while
   remaining managed by their existing OCM hubs?
2. Can a target be reached through its OCM hub, directly through Fleet, or
   through another delivery mechanism without changing its identity?
3. Can Fleet identity, authentication, authorization, and RBAC guarantees be
   maintained across those delivery paths?
4. Can the same logical capabilities support both a production single-pod
   topology and a distributed, multi-service topology?

## Executive summary

The projects have complementary strengths:

| System | Potential responsibility in a combined architecture |
| --- | --- |
| OCM | Regional hub and managed-cluster relationships, placement and established hub-to-cluster delivery mechanisms such as ManifestWork |
| Global Hub | Cross-hub discovery and inventory, compliance and security reporting, historical data, hub health, and existing hub-level operations such as managed-cluster migration |
| OCM Fleet | Fleet-wide target identity, resource APIs, user experience, authorization, desired delivery intent, and orchestration across delivery mechanisms |

The key architectural distinction is between a **target** and its **delivery
route**. An inventoried cluster can be represented as a Fleet target without
being a Fleet platform instance. Its current OCM hub is one possible route to
that target, not an intrinsic part of its identity. A Fleet-native agent or a
future delivery technology may provide another route.

This model is a hypothesis. It requires explicit contracts for target identity,
route selection and handoff, resource ownership, authorization, and status. It
must not silently fall back to a route with weaker security or permit two
independent reconcilers to manage the same resources.

The deployment topology is a separate choice. Logical component boundaries
should support a single-pod production installation as well as a distributed
multi-service installation. Modularity should not make additional services,
databases, or message brokers mandatory when the single-pod topology can meet
the stated workload and durability requirements.

## Goals and non-goals

### Goals

- Keep existing OCM hubs as OCM hubs; do not require them to run Fleet.
- Make inventoried managed clusters eligible to become Fleet delivery targets.
- Allow target selection to be independent of the mechanism used to deliver
  work.
- Preserve a coherent Fleet authentication and authorization boundary for
  both hub-mediated and Fleet-native delivery.
- Keep Global Hub's reporting and historical-data capabilities available
  without making its current service topology a mandatory Fleet dependency.
- Support production use in both single-pod and distributed deployments.
- State clearly what is implemented, what is design intent, and what remains
  unproven.

### Non-goals for this investigation

- Replacing OCM or changing OCM hubs into Fleet instances.
- Selecting a final delivery transport or implementing a production connector.
- Replacing Global Hub's compliance history with Fleet's latest-state
  inventory.
- Combining the repositories as a prerequisite for architectural convergence.
- Claiming that the security properties of one delivery path automatically
  apply to another.

## Conceptual model

```text
                    Fleet API / CLI / Console
                               |
              Fleet identity, authorization, and intent
                               |
                 Stable Fleet cluster-target identity
                               |
                 Delivery policy and route selection
                    /                 |              \
             Existing OCM hub    Fleet-native      Future route
                    |              delivery        implementation
              ManifestWork             |                |
                    +------------------+----------------+
                                       |
                                Managed cluster

       Global Hub: cross-hub discovery, inventory, history,
       compliance, security observations, and hub operations
```

The diagram separates four concepts that must not be conflated:

1. **Target identity:** the cluster Fleet intends to address.
2. **Discovery and association:** observations that a cluster exists and is
   currently connected to a particular OCM hub.
3. **Delivery mechanism:** the path that carries and applies a particular
   fulfillment.
4. **Authorization:** who may request the operation, what evidence is required,
   and which identity actually applies it.

Global Hub discovery can inform target creation and hub association. Discovery
alone does not enroll a target into Fleet trust, assign it to a tenant or
workspace, or grant a user permission to deliver to it.

## Responsibility and state ownership

The following ownership split is a starting point for investigation, not a
final decision. Each state transition should have one authoritative owner.

| State or capability | Candidate owner | Boundary to validate |
| --- | --- | --- |
| OCM hub's regional cluster registration and hub-local resources | OCM | Fleet must not duplicate OCM registration semantics. |
| Cross-hub observations and hub association | Global Hub or an explicitly defined discovery adapter | Decide which identity fields are durable and which are source-specific observations. |
| Fleet target identity, tenant/workspace assignment, and trust enrollment | Fleet | Importing an observation must not silently grant authority. |
| Desired deployment/fulfillment, placement, rollout, and route choice | Fleet | Keep one orchestration owner for Fleet-created intent. |
| ManifestWork reconciliation through a hub | OCM | Fleet should use the supported OCM contract rather than compete with its controllers. |
| Target-local application and Kubernetes authorization | Delivery mechanism and target policy | Specify what each route verifies and which apply identity it uses. |
| Cross-hub compliance and historical reporting | Global Hub or a dedicated analytics service | Do not assume Fleet's current inventory store supplies historical analytics. |
| Fleet API access and workspace authorization | Fleet | Addons and connectors must not bypass Fleet's permission decision. |
| Cluster migration workflow | Existing Global Hub/OCM owner unless explicitly changed | Fleet may present and orchestrate a request, but must not introduce a second migration controller by default. |

Avoid shared ownership of database tables. Prefer a versioned API or event
contract with explicit freshness, identity, deletion, and replay semantics.

## Target identity and discovery

### Identity should outlive a route

A Fleet target should keep its identity when its delivery route changes from one
OCM hub to another, or from a hub-mediated path to a Fleet-native path. The
identity should not be derived only from a mutable hub association, cluster
name, network endpoint, or transport-specific ID.

The canonical identifier and its trust establishment are open questions. OCM
object UIDs, imported-cluster identifiers, infrastructure IDs, and
cryptographically asserted target identities have different lifecycle and
security properties. The investigation should establish which identifiers
survive hub migration and cluster re-import, how recreation is distinguished
from continuity, and how collisions or conflicting claims are handled.

### Discovery is not enrollment

Global Hub may report that a cluster exists, its observed properties, and its
current hub association. Fleet should treat those as discovery inputs. A
separate policy or explicit workflow should decide:

- whether the cluster is admitted as a Fleet target;
- which Fleet tenant and workspace own or can see it;
- which delivery mechanisms it supports;
- which trust anchors and authorization profiles are accepted; and
- who may request delivery to it.

Stale or disconnected observations must remain visibly stale rather than being
treated as proof of current reachability or current authorization.

### Migration and target continuity

When Global Hub migrates a cluster between regional hubs, the design should
preserve Fleet target identity while updating route association. Define the
behavior for in-flight work, old-hub disconnection, new-hub readiness,
duplicate status reports, and rollback. A hub migration must not implicitly
transfer tenant ownership, trust, or delivery authority.

## Delivery mechanisms and route independence

Candidate routes include:

1. **Hub-mediated:** Fleet sends a delivery request to the associated OCM hub,
   which delivers work to the managed cluster using an OCM mechanism such as
   ManifestWork.
2. **Fleet-native:** a Fleet-connected delivery agent or other Fleet transport
   reaches the cluster without using the OCM hub as the application path.
3. **Future mechanisms:** another implementation that satisfies the same
   target, authorization, delivery, observation, and recovery contracts.

These are possible complementary paths, not an assumption that every cluster
must support all of them. The architecture should model capabilities and
authorization profiles explicitly and select an eligible path for each
fulfillment.

### Route selection and handoff

The contract should define:

- how supported routes and their capabilities are registered and kept fresh;
- whether route choice is per target, fulfillment, resource type, or operation;
- how ordering and generation are preserved across transports;
- how duplicate, delayed, and replayed deliveries are handled;
- how progress, apply, availability, and errors are reported consistently;
- how ownership transfers when a route changes; and
- how removal, pruning, and cleanup work after route loss or migration.

Multiple mechanisms may be available for one target, but that must not result
in two active controllers applying or pruning the same resources
unintentionally. Route failover should use explicit fencing, ownership, or
handoff semantics. It must not be treated as a transparent retry if the new
route has different trust or apply authority.

### Route-independent guarantees

Fleet should define the authorization properties a delivery must satisfy, then
document which routes can provide them. If a route cannot satisfy a policy,
Fleet should reject that route or require an explicit weaker policy; it should
not silently weaken authorization during fallback.

## Fleet identity, authentication, and authorization

### Separate the identities

For each operation, record and reason about at least:

| Identity | Meaning |
| --- | --- |
| Fleet caller | The authenticated user or workload that requested the operation. |
| Authorized intent | The exact target, tenant/workspace scope, payload or fulfillment generation, and constraints approved by Fleet policy. |
| Delivery identity | The principal used by a connector, hub, or agent to transport or submit the work. |
| Apply identity | The user, delegated workload, or target-local service identity that performs the Kubernetes API operation. |

These identities may be different. Audit metadata that names the original
caller is useful, but it does not by itself enforce authorization. Likewise,
authenticating a connector to a hub does not establish that the requested work
was allowed by Fleet's tenant and workspace policy.

### Hub-mediated path

The hub route must preserve Fleet's authorization decision across the Fleet to
hub boundary and define what the target independently verifies. Candidate
mechanisms include:

- mapping Fleet principals to hub identities and target permissions;
- scoped delegation to a constrained service account or workload identity;
- an authenticated, target-bound delivery envelope carrying Fleet-signed
  intent or provenance, verified before apply; or
- a composition of these mechanisms.

These are alternatives for investigation, not selected designs. Any identity
mapping must specify issuer, subject, tenant mapping, audience, lifetime,
revocation, and audit correlation. A delegation flow must constrain the
authority that can be exercised and address permission drift. A signed intent
must bind the relevant payload, target identity, tenant, generation, and
constraints, and must define replay and trust-update behavior.

OCM has relevant existing mechanisms: hub-side admission checks whether a
caller may set a ManifestWork executor (`execute-as`), and spoke-side executor
validation checks the configured service account's resource permissions and
RBAC escalation. These are useful building blocks, but they do not by
themselves establish Fleet tenant/workspace authorization or verify Fleet
provenance. The complete end-to-end path needs validation.

Do not assume that putting the Fleet user's name or groups in a ManifestWork
annotation makes the target act as that user or enforces the user's Fleet
permissions. Direct end-user impersonation by a privileged connector also
requires explicit review; a narrowly delegated target identity may provide a
safer apply model, but changes native attribution and permission-lifecycle
behavior.

### Fleet-native path

A direct Fleet path can use Fleet's defined credential-presentation and
provenance choices, subject to the target's capabilities. Depending on policy,
the apply identity could be the user's credential, a scoped delegated identity,
or a target-local delivery-agent identity. The path must still bind the
operation to the stable Fleet target and enforce target-local constraints.

### Security requirements to compare

For each route, document:

- Fleet-side authentication and tenant/workspace authorization;
- connector or hub authentication and least-privilege permissions;
- target-side verification of credentials, provenance, target scope, and
  ordering;
- credential audience, storage, expiration, refresh, and revocation;
- behavior when the user loses access while work is queued or a target is
  disconnected;
- behavior when trust configuration or proof material is missing or stale;
- protection against route switching, payload alteration, and replay; and
- audit correlation from Fleet caller through delivery identity to apply
  identity and resulting resources.

Also state the compromise assumptions. A design may trust Fleet as the primary
stateful authorizer while asking the target to verify selected signed facts. It
must not claim the target is independent of Fleet authorization unless the
necessary policy and trust state are actually available at the target.

## Production deployment topologies

The logical architecture should not require a particular packaging topology.
Investigate the same user-visible capabilities in both modes:

| Topology | Design intent to validate |
| --- | --- |
| **Single-pod production** | Core APIs, orchestration, integration, and local persistence can operate without mandatory distributed infrastructure when the workload allows it. Define persistent storage, backup/restore, upgrade, and recovery requirements. |
| **Distributed multi-service** | Services can scale, isolate failures, and use external persistence or messaging where needed. Define service ownership, API/event contracts, coordination, and operational requirements. |

Single-pod production does not automatically mean high availability. The
availability, recovery, scale limits, and failure domains of each topology
must be explicit. Conversely, distributed services should add operational
flexibility or scale rather than being required merely to preserve code
modularity.

Global Hub currently documents manager, agent, Kafka, PostgreSQL, and related
services as part of its architecture. Fleet's design also includes a single-pod
viability invariant with embedded persistence as a first-class option. A
combined design must determine which Global Hub capabilities can be embedded
or run locally, which require durable external services at scale, and which
should remain optional integrations. Similar-looking storage or messaging
interfaces do not prove equivalent durability, ordering, or recovery semantics.

Compare at least:

1. Integrating existing services through stable contracts.
2. Extracting reusable Global Hub modules while retaining independent service
   deployments.
3. A shared runtime with optional service separation for larger deployments.

Repository consolidation is a separate decision and should be justified by
build, release, ownership, and operational benefits—not assumed as a condition
of capability convergence.

## Inventory, history, and analytics

Global Hub's cross-hub reporting and historical compliance are distinct from
the current-state inventory needed to identify Fleet targets. Fleet should not
be assumed to replace Global Hub's analytics store merely because it can accept
inventory reports.

Evaluate three patterns:

1. **Reference/query:** Fleet queries Global Hub or a reporting API on demand.
2. **Current-state projection:** Fleet stores selected latest observations
   needed for target identity, placement, and UI; Global Hub remains the owner
   of compliance history and analytics.
3. **Dedicated analytics boundary:** Global Hub or a separate analytics
   service exposes aggregated and historical views, while Fleet owns
   orchestration state.

The design should define freshness, pagination, partial failure, deletion,
retention, and tenant filtering for any view. It should avoid shared database
ownership and make the source of truth visible to users.

Fleet's current inventory-report service is oriented toward latest reported
state; it should not be treated as a historical compliance store without
additional design and implementation. Global Hub's PostgreSQL history and
compliance summarization remain relevant capabilities to preserve or expose.

## Risks and mitigations

| Risk | Mitigation or investigation |
| --- | --- |
| Cluster identity changes during hub migration or re-import | Define a stable Fleet target identity and an authenticated continuity/claim process. |
| Discovery is mistaken for enrollment or authorization | Separate observed inventory from Fleet trust, tenant/workspace assignment, and delivery grants. |
| Hub mediation loses Fleet security context | Specify and test end-to-end identity, intent, delegation, provenance, and enforcement at each hop. |
| Direct and hub routes apply the same resources concurrently | Define route ownership, fencing, handoff, generation ordering, and cleanup. |
| Failover weakens authorization | Require route eligibility under the same policy; reject or explicitly pause when guarantees cannot be met. |
| Two systems own the same mutable state | Assign authoritative owners and use explicit contracts rather than shared tables or duplicate controllers. |
| Current-state inventory is confused with historical compliance | Keep history and analytics ownership explicit and expose source/freshness information. |
| Single-pod and distributed modes behave differently | Test semantic parity, persistence guarantees, recovery, and failure behavior in both topologies. |
| Proposed capabilities are mistaken for existing functionality | Label current implementation, design intent, and hypotheses distinctly; validate claims against code. |

## Open questions

1. Which cluster identifier is stable across OCM hub migration, re-import,
   recreation, and loss of the original hub?
2. What explicit enrollment and trust step converts a discovered cluster into a
   Fleet target?
3. Which delivery contract can be implemented by both an OCM hub connector and
   a Fleet-native agent without making their security guarantees appear
   identical when they are not?
4. Where should Fleet authorization be enforced for ManifestWork delivery, and
   what signed evidence or delegation is necessary at the target?
5. How are target-local permissions provisioned, revoked, and reconciled when
   Fleet grants or user permissions change?
6. What fencing and handoff protocol prevents concurrent application or
   pruning during route changes?
7. Which Global Hub data belongs in a Fleet current-state projection, and which
   data remains query-time or historical analytics?
8. What are the minimum persistence, backup, and recovery guarantees for
   single-pod production? Which workloads require distributed services?
9. Which services and transports are truly optional, and what failure behavior
   is acceptable when each is unavailable?
10. What first proof of concept would validate the most consequential
    assumptions without prematurely committing to a deployment topology?

## Suggested investigation sequence

1. Build a code-backed capability and state-ownership map for Fleet and Global
   Hub; separate implemented behavior from design documents.
2. Trace one managed cluster from Global Hub observation through Fleet target
   enrollment and a hub-mediated delivery request to target-side apply and
   status.
3. Trace an equivalent request through a Fleet-native path and compare identity,
   authorization, provenance, ordering, audit, and recovery guarantees.
4. Test identity continuity and safe delivery handoff when a managed cluster
   moves between hubs or changes routes.
5. Compare single-pod and distributed deployment designs against the same
   functional and recovery scenarios.
6. Record a recommended first increment and the remaining decisions before
   implementation or repository consolidation.

## References

### Multicluster Global Hub

- [Global Hub architecture](ARCHITECTURE.md)
- `manager/pkg/status/handlers/handlers.go` — registered status domains.
- `manager/pkg/migration/` — managed-cluster migration workflow.
- `pkg/database/models/` — Global Hub persistence models.

### OCM Fleet

- [Management plane architecture](https://github.com/stolostron/ocm-fleet/blob/main/docs/design/architecture.md)
- [Authentication and credential presentation](https://github.com/stolostron/ocm-fleet/blob/main/docs/design/authentication.md)
- [Delivery security model](https://github.com/stolostron/ocm-fleet/blob/main/docs/design/security.md)
- [Addon integration](https://github.com/stolostron/ocm-fleet/blob/main/docs/design/architecture/addon_integration.md)
- [Tenancy and permissions](https://github.com/stolostron/ocm-fleet/blob/main/docs/design/architecture/tenancy_and_permissions.md)
- [Platform hierarchy and query federation](https://github.com/stolostron/ocm-fleet/blob/main/docs/design/architecture/platform_hierarchy.md)
- `server/internal/application/inventory_report_service.go` — current inventory reporting semantics.

### Open Cluster Management

- [`ManifestWork` executor admission validation](https://github.com/stolostron/ocm/blob/main/pkg/work/webhook/v1/manifestwork_validating.go)
- [Spoke executor permission and escalation checks](https://github.com/stolostron/ocm/blob/main/pkg/work/spoke/auth/basic/auth.go)
