# Declarative System Definition Framework — Manifest Schema Draft v0.1

**Date:** 2026-03-08  
**Purpose:** First concrete draft of the manifest shape implied by the reworked PRD.

This draft is intentionally biased toward a small, stable Layer 1 core and an optional Layer 2 dispatch extension.

## Design intent

The manifest should do three things well:

1. Define the externally meaningful shape of a system.
2. Define its interfaces, contracts, and integrations in a reviewable form.
3. Optionally define canonical dispatch lifecycle/state semantics for event dispatch systems.

It should *not* try to encode implementation detail, deployment topology, storage tables, or worker internals.

## Top-level shape

```yaml
manifestVersion: v1alpha1
kind: SystemDefinition
metadata:
  name: string
  title: string
  summary: string
  tags: []
system:
  domain: string
  scope: string
  style: integration | dispatch | application
components: []
interfaces: []
contracts: []
integrations: []
dispatch: {}   # optional Layer 2 extension
realization: {} # optional future layer, intentionally reserved
```

## Layer 1 core sections

### metadata
Simple document identity and human review metadata.

### system
Defines the bounded thing being described.

Recommended fields:
- `domain`
- `scope`
- `style`
- `owners` (optional)
- `notes` (optional)

### components
Logical units whose boundary matters externally or architecturally.

Recommended minimum fields per component:
- `name`
- `type`
- `summary`
- `boundary` (`internal` or `external`)

### interfaces
Named boundaries where information enters or leaves a component.

Recommended minimum fields per interface:
- `name`
- `component`
- `direction` (`inbound`, `outbound`, or `bidirectional`)
- `mechanism`
- `summary`
- `contracts`

### contracts
Reusable payload and message definitions.

Recommended minimum fields per contract:
- `name`
- `kind`
- `schema`

The schema should be JSON-Schema-like, but the manifest can remain YAML-authored.

### integrations
Connections between interfaces or between a component interface and an external boundary.

Recommended minimum fields per integration:
- `name`
- `from`
- `to`
- `mechanism`
- `contracts`

## Layer 2 dispatch extension

The `dispatch` block should remain absent unless the system is explicitly a dispatch/stateful work-tracking system.

Recommended shape:

```yaml
dispatch:
  workTypes: []
  lifecycleModels: []
  dispatchAttempts: {}
  statusUpdates: {}
  correlationRules: []
  surfaces: {}
```

### workTypes
Defines the kinds of work accepted by the system.

Recommended minimum fields:
- `name`
- `submissionContract`
- `lifecycleModel`
- `dispatchTargets`

### lifecycleModels
Defines canonical lifecycle semantics.

Recommended minimum fields:
- `name`
- `initialStatus`
- `statuses`
- `transitions`

Each status should support:
- `name`
- `category` (`accepted`, `in_progress`, `succeeded`, `failed`, `cancelled`, `unknown`)
- `terminal` (boolean)
- `requiredDetails` (optional)

### dispatchAttempts
Defines the canonical shape of outbound attempts.

Recommended fields:
- `enabled`
- `contract`
- `identityFields`
- `statusFields`

### statusUpdates
Defines the canonical shape of observed status changes.

Recommended fields:
- `contract`
- `sources`
- `requiredCommonFields`

### correlationRules
Defines how downstream observations map back to canonical work.

Recommended fields:
- `name`
- `appliesTo`
- `strategy`
- `fields`

### surfaces
Defines the canonical retrieval/reporting surfaces that should exist.

Recommended fields:
- `submit`
- `queryById`
- `history`
- `attempts`

## Opinionated choices in this draft

This draft intentionally chooses the following:

- Interfaces are first-class.
- Contracts are reusable named objects rather than inline blobs everywhere.
- Integrations describe relationships, not runtime behavior.
- Dispatch state is canonical and internal-first, not merely copied from downstream systems.
- Realization remains reserved and non-authoritative for now.

## Questions to pressure-test next

1. Should `interfaces` and `integrations` both remain top-level, or should one collapse into the other?
2. Should contracts be fully JSON Schema, or should the manifest allow a smaller constrained schema subset?
3. Should lifecycle transitions support guard expressions now, or only declarative `from` -> `to` constraints?
4. Should `surfaces` live under dispatch, or should query/history surfaces be modeled as normal interfaces?
5. Should dispatch targets be declared on work types, integrations, or both?

## Files accompanying this draft

- `system_definition_manifest.schema.json` — first machine-validatable schema draft
- `example_layer1_system.yaml` — pure Layer 1 example
- `example_layer2_dispatch_system.yaml` — Layer 1 + Layer 2 dispatch example

