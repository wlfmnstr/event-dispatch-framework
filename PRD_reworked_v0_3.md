# Declarative System Definition Framework — Product Requirements Document

**Status:** Draft v0.3  
**Author:** Engineering Architecture  
**Last Updated:** 2026-03-08

---

## 1. Executive Summary

This product defines systems declaratively.

Its purpose is to let engineers describe the externally meaningful shape of a system, its interfaces, its integration points, and—when applicable—the lifecycle and state model of dispatched work, using structured manifests and schemas instead of repeatedly redesigning the same patterns.

The framework is organized into three conceptual layers:

1. **Layer 1 — Declarative System Universals**  
   A bounded declarative model for describing system boundaries, interfaces, contracts, and integrations.

2. **Layer 2 — Declarative Event Dispatch State Model**  
   A specialized extension for systems that accept work, dispatch it to downstream systems, and track lifecycle state over time.

3. **Layer 3 — Realization / Generation**  
   A downstream layer that consumes the outputs of Layers 1 and 2 to generate implementation-adjacent artifacts.

The product center of gravity is Layers 1 and 2. Layer 3 matters, but it is downstream.

This PRD intentionally narrows the framework around a clear thesis:

**Start with a small universal system-definition core, then add a high-value event dispatch state extension that solves the recurring architecture pain.**

---

## 2. Problem Statement

Teams repeatedly build systems that share the same external shape.

At a broad level, teams repeatedly redefine:
- what the main boxes in a system are,
- what interfaces those boxes expose,
- how those boxes communicate,
- what messages cross those boundaries,
- what contracts govern those messages,
- and which integration mechanisms are used.

For event dispatch systems specifically, teams also repeatedly redesign:
- how submitted work is represented,
- how accepted work is persisted canonically,
- how dispatch attempts are tracked,
- how statuses are modeled,
- how state transitions are constrained,
- how asynchronous downstream progress is correlated,
- and how APIs, storage, and reporting align around lifecycle state.

The repeated pain is not mainly API description. The repeated pain is repeated structural and lifecycle architecture.

This leads to:

1. **Repeated design effort**  
   Engineers solve the same structural problems many times.

2. **Inconsistent system shape**  
   Similar systems diverge in contracts, lifecycle semantics, naming, and integration conventions.

3. **Slow architecture-to-build flow**  
   Teams spend too much time rediscovering patterns before implementation starts.

4. **Weak reuse of best practices**  
   Architecture stays trapped in people and teams rather than encoded in durable definitions.

---

## 3. Product Vision

A developer should be able to define a system by authoring a structured manifest that captures only what is system-specific.

From that definition, the framework should be able to produce canonical declarative assets such as:
- JSON Schemas,
- OpenAPI specifications,
- AsyncAPI specifications,
- integration contracts,
- lifecycle/state definitions,
- examples,
- documentation,
- and later, downstream realization inputs.

The vision is not “generate code first.”

The vision is:

**define systems once, in a precise declarative language, and make those definitions reusable, reviewable, operational, and consistent.**

---

## 4. Product Boundary

### In Scope

- A declarative language for describing system boundaries and integrations
- A manifest as the source of truth
- A small set of universal primitives for describing systems externally
- A dispatch-specific extension for lifecycle and state tracking
- Generation of machine-readable assets from those definitions
- A clean package boundary for downstream realization tooling

### Explicitly De-emphasized in This PRD

- Detailed implementation roadmap phases
- Builder orchestration strategy
- Repo topology
- Organization-specific scaffolding mechanics
- Detailed code generation flow
- Detailed infrastructure automation design
- Internal business logic implementation details

### Explicit Non-Goal

This product is **not** trying to be a general-purpose “describe all software” metamodel.

It is a bounded declarative framework for:
- external system shape,
- inter-component and inter-system interaction,
- and, where relevant, dispatch lifecycle state.

---

## 5. Users and Primary Use Cases

### Primary Users

- Engineers and architects defining system boundaries and contracts
- Technical product or data analysts who help define message shapes and lifecycle semantics
- Platform or framework builders who want stable declarative inputs for downstream tooling

### Primary Use Cases

1. **Define a system boundary**  
   Describe the components that matter and the interfaces they expose.

2. **Define integrations**  
   Specify how components and external systems communicate and what crosses those boundaries.

3. **Define a dispatch system**  
   Add lifecycle and state semantics for work that is submitted, dispatched, and tracked over time.

4. **Generate canonical assets**  
   Produce schemas, specs, examples, and docs from a single source of truth.

5. **Feed downstream realization**  
   Use those assets as stable inputs to later automation or generation tooling.

---

## 6. Core Product Thesis

The product should not start as a fully generic modeling framework.

It should start as:
- a **small universal core** that describes external system shape, and
- a **specialized event dispatch extension** that solves the higher-value lifecycle problem.

That means:
- Layer 1 must stay small, stable, and reusable.
- Layer 2 must solve the painful recurring event-state problem directly.
- Layer 3 must consume the outputs of Layers 1 and 2 without redefining the product around realization.

This gives the product focus without locking it into a single one-off system type.

---

## 7. Product Structure

## 7.1 Layer 1 — Declarative System Universals

Layer 1 defines the smallest reusable set of primitives needed to describe a system externally.

It is intentionally not a full internal architecture language.

### Purpose

To provide a declarative language for describing:
- what the system is,
- what externally meaningful components exist,
- what interfaces those components expose,
- what information crosses those interfaces,
- and what integration mechanisms connect boundaries.

### Layer 1 Design Principles

1. **External shape over internal implementation**  
   Layer 1 describes boundaries and interactions, not internal business logic.

2. **Minimal primitive set**  
   Every primitive must earn its place. If a concept can be expressed as metadata on another primitive, it should not become its own top-level primitive.

3. **Bounded opinionation**  
   The framework should make common mechanisms first-class without trying to model every possible integration pattern.

4. **Portable outputs**  
   The generated assets should rely on standard formats where possible.

### Layer 1 Minimal Primitive Set

Layer 1 should be reduced to the following core primitives:

- **System**  
  The bounded thing being defined.

- **Component**  
  A logical unit whose boundary matters externally.

- **Interface**  
  A named boundary through which a component receives or emits information.

- **Contract**  
  The declared structure of what crosses an interface.

- **Integration**  
  A declared relationship connecting one interface to another boundary.

That is the core.

The following concepts should exist, but generally as attributes or classifications rather than necessarily as separate top-level primitives:
- ingress vs egress,
- request vs response vs event,
- internal vs external boundary,
- sync vs async,
- mechanism type,
- cardinality,
- directionality,
- ownership.

### Why This Primitive Set

This is the smallest set that still lets the framework describe:
- boxes,
- the boundaries on those boxes,
- what crosses the boundaries,
- and how boundaries connect.

Anything beyond that should be treated with suspicion unless it clearly reduces ambiguity or increases downstream usefulness.

### Layer 1 Interface Model

Each interface should declare, as applicable:
- its owning component,
- whether it is inbound, outbound, or bidirectional,
- its interaction mechanism,
- its contract references,
- and any protocol-specific metadata needed to generate standard artifacts.

### Supported Interaction Mechanism Categories

The framework should assume a bounded set of first-class mechanisms such as:
- synchronous HTTP/API,
- asynchronous queue or topic,
- event stream,
- file exchange,
- scheduled or batch exchange,
- callback/webhook,
- polling,
- adapter/custom.

These categories should be explicit and opinionated. They should not attempt to model every protocol nuance at the core language level.

### Layer 1 Expected Outputs

From Layer 1, the framework should be able to generate:
- JSON Schemas,
- OpenAPI specs,
- AsyncAPI specs,
- interface and integration reference docs,
- example payloads,
- and diagram-ready metadata.

### Layer 1 Non-Goals

Layer 1 should not attempt to define:
- database schema design broadly,
- internal service orchestration logic,
- deployment topology,
- retry policy internals,
- or realization-specific code structure.

Those may appear in extensions or downstream tooling, but they are not part of the universal core.

---

## 7.2 Layer 2 — Declarative Event Dispatch State Model

Layer 2 extends Layer 1 for a recurring and high-value system class:

A system accepts work, persists a canonical representation of that work, dispatches it to downstream systems, and tracks lifecycle state over time.

This is the part that solves the actual recurring pain.

### Purpose

To provide a declarative way to define event dispatch lifecycle and state semantics so teams stop repeatedly inventing new models for the same class of systems.

### Layer 2 Design Principles

1. **State is first-class**  
   In dispatch systems, lifecycle state is not incidental. It is central.

2. **Canonical internal view**  
   The framework must support a canonical internal state model even when downstream systems are inconsistent.

3. **Declarative lifecycle, not hardcoded lifecycle**  
   The system author should declare statuses, transitions, categories, and correlation rules instead of re-implementing them each time.

4. **Dispatch-specific, not universal**  
   Layer 2 is intentionally specialized. It should not be forced into systems that do not need lifecycle state.

### Layer 2 Minimal Primitive Set

Layer 2 should be narrowed to the following core primitives:

- **Work Type**  
  The kind of work the system accepts.

- **Work Item**  
  The canonical submitted unit of work.

- **Lifecycle Model**  
  The declared set of statuses, categories, and transition rules for a work item.

- **Dispatch Attempt**  
  An outbound attempt to hand work to a downstream system.

- **Status Update**  
  A recorded lifecycle-affecting observation about a work item or dispatch attempt.

- **Correlation Rule**  
  The declared mapping or identifier strategy that relates internal work to downstream systems.

That is the core Layer 2 model.

The following should generally be expressed as attributes or substructures rather than separate top-level primitives unless there is a strong reason otherwise:
- terminal vs non-terminal,
- success vs failure vs in-progress category,
- reason codes,
- retryability,
- status source,
- status timestamps,
- downstream reference IDs,
- required details by status,
- submit/query/history surface patterns.

### What Layer 2 Must Make Declarative

Layer 2 must let a system author declare:
- what work types the system accepts,
- the canonical payload shape of each work type,
- the lifecycle model for each work type or family,
- allowed transitions,
- terminal states,
- what details are required for specific status updates,
- how dispatch attempts are represented,
- how downstream progress is learned,
- how correlation works,
- and what canonical retrieval or observation surfaces exist.

### Canonical State Principle

Layer 2 should assume the framework owns a canonical state model even when downstream systems:
- expose partial state,
- expose delayed state,
- expose inconsistent state,
- or expose no push-based state at all.

This is a critical product distinction. The framework is not merely documenting downstream statuses. It is declaratively defining the canonical lifecycle model of the dispatch system itself.

### Layer 2 Expected Outputs

From Layer 2, the framework should be able to generate:
- work submission schemas,
- lifecycle/state schemas,
- status history schemas,
- dispatch attempt schemas,
- canonical submit/query/status-history API specs,
- lifecycle documentation,
- example lifecycle instances,
- and reporting/observability contracts.

### Layer 2 Non-Goals

Layer 2 should not attempt to define:
- every storage table in implementation detail,
- worker scheduling internals,
- execution engine logic,
- vendor-specific downstream behavior as first-class framework concepts,
- or orchestration/runtime semantics beyond what is needed to define lifecycle state.

---

## 7.3 Layer 3 — Realization / Generation

Layer 3 is the downstream consumption layer.

Its purpose is to use the declarative assets defined by Layers 1 and 2 to help generate implementation-adjacent outputs.

### Purpose

To consume declarative assets and turn them into things such as:
- application scaffolds,
- infrastructure definitions,
- configuration files,
- docs packages,
- generated adapters,
- and other realization artifacts.

### Role in This PRD

Layer 3 should be acknowledged but not over-specified.

This PRD should define Layer 3 mainly as:
- a consumer of Layers 1 and 2 outputs,
- a reason to keep declarative outputs stable,
- and a future location for builder or generator contracts.

Layer 3 should not drive the core language design prematurely.

---

## 8. Manifest Strategy

The framework should use a manifest as the source of truth.

### Manifest Principles

The manifest should:
- capture only what varies,
- express declarations rather than implementation detail,
- support progressive authoring,
- be schema-validatable,
- separate universal concerns from dispatch-specific concerns,
- and remain readable enough for design review.

### Manifest Family

The framework should treat the manifest as a family with a stable shared spine:
- a **Layer 1 base shape** for universal system definition,
- an optional **Layer 2 extension block** for event dispatch state,
- and optional future **Layer 3 realization hints**.

This avoids one giant undifferentiated document while preserving a single source-of-truth experience.

### Recommended Top-Level Manifest Shape

The next revision should tighten the manifest around a structure like:

```yaml
manifestVersion: v1
kind: SystemDefinition
system:
  ...
components:
  ...
interfaces:
  ...
contracts:
  ...
integrations:
  ...
dispatch:
  ... # optional Layer 2 extension
realization:
  ... # optional future Layer 3 hints
```

### Recommended Layer 1 Manifest Sections

#### system
Should define:
- system name,
- purpose/summary,
- bounded scope,
- optional tags or domain metadata.

#### components
Should define only components whose boundary matters externally or architecturally.

Each component should minimally declare:
- name,
- type/classification,
- responsibility summary,
- ownership or boundary metadata if needed.

#### interfaces
Should be first-class because they are the main boundary surface.

Each interface should minimally declare:
- name,
- owning component,
- direction,
- mechanism type,
- protocol-specific metadata,
- contract references.

#### contracts
Should define reusable payload shapes and envelopes.

Contracts should support:
- request/response contracts,
- event or message contracts,
- shared schema fragments,
- validation constraints,
- examples.

#### integrations
Should connect interfaces or components declaratively.

Each integration should minimally declare:
- source boundary,
- target boundary,
- mechanism,
- contract reference(s),
- and any mapping or correlation metadata needed at the boundary level.

### Recommended Layer 2 Manifest Extension

The dispatch block should define only the additional concepts required for event dispatch state modeling.

A tightened shape should look roughly like:

```yaml
dispatch:
  workTypes:
    - ...
  lifecycleModels:
    - ...
  correlationRules:
    - ...
  observationSurfaces:
    - ...
```

#### workTypes
Each work type should declare:
- name,
- submission contract,
- target integration(s),
- canonical identity fields,
- lifecycle model reference.

#### lifecycleModels
Each lifecycle model should declare:
- statuses,
- categories,
- allowed transitions,
- terminality,
- required details by status where needed.

#### correlationRules
Should declare how internal work items and downstream activity relate.

#### observationSurfaces
Should declare the canonical submit/query/history-facing surfaces that the dispatch system exposes.

### Manifest Quality Bar

A good manifest should let a reviewer answer, without reading implementation code:
- what this system is,
- what boundaries exist,
- what crosses those boundaries,
- how those boundaries connect,
- whether the system is dispatch/stateful,
- what the lifecycle model is,
- and what canonical artifacts should be generated.

If the manifest does not answer those questions, it is either too weak or too implementation-specific.

---

## 9. Product Positioning Decision

The product should be positioned as:

**a declarative system definition framework with a first-class event dispatch state extension.**

This is the recommended middle path because it preserves both:
- immediate usefulness for dispatch-system work already on the roadmap,
- and future extensibility for adjacent categories of systems.

### Why Not Event Dispatch Only

That would overfit the language to one system class and make future reuse awkward.

### Why Not Fully Generic From Day One

That would risk producing an abstract framework that does not actually solve the high-value repeated pain.

### Why the Recommended Position Works

It keeps Layer 1 small and reusable, while letting Layer 2 carry the heavier value and specificity.

---

## 10. Functional Requirements

### 10.1 Layer 1 Functional Requirements

The framework must:
- allow definition of a system and its components,
- allow declaration of interfaces and integration points,
- allow declaration of bounded interaction mechanism categories,
- allow declaration of message and request/response contracts,
- generate machine-readable interface artifacts,
- generate examples and documentation,
- and validate manifests and outputs.

### 10.2 Layer 2 Functional Requirements

The framework must:
- allow declaration of work types,
- allow declaration of lifecycle models,
- allow declaration of transition rules and terminal states,
- allow declaration of required detail by lifecycle state where needed,
- allow declaration of dispatch attempt representation,
- allow declaration of downstream correlation rules,
- generate canonical state and lifecycle artifacts,
- generate submit/query/status-history-facing contracts,
- and validate examples and lifecycle instances against generated assets.

### 10.3 Layer 3 Functional Requirements

The framework should eventually:
- expose a stable, consumable package of declarative outputs,
- support downstream tooling consuming those outputs,
- and keep realization tooling decoupled from the core definition engine.

---

## 11. Non-Functional Requirements

The framework should be:

- **Declarative-first**  
  Definitions should express what a system is, not how every implementation step works.

- **Opinionated and bounded**  
  It must encode useful conventions without attempting to model all software.

- **Composable**  
  Layer 2 must extend Layer 1 cleanly.

- **Schema-validatable**  
  Manifests and outputs should be machine-validatable.

- **Deterministic**  
  The same input should produce the same outputs.

- **Portable**  
  Outputs should favor standard formats.

- **Reviewable**  
  Definitions should be understandable in architecture and product review contexts.

- **Useful before realization**  
  Layers 1 and 2 must deliver standalone value even before automation exists.

---

## 12. Key Edits from the Previous Draft

The reworked PRD should explicitly preserve the following changes:

1. **No phased rollout framing**  
   The product is defined conceptually by layers, not by phased slices.

2. **Reduced realization dominance**  
   Builders, repo flows, and orchestration mechanics are no longer defining the product.

3. **Smaller Layer 1 core**  
   Layer 1 is narrowed to a minimal universal set of primitives.

4. **Sharper Layer 2 focus**  
   Layer 2 is explicitly about canonical dispatch lifecycle and state modeling.

5. **Tighter manifest shape**  
   The manifest is now treated as a shared spine with a dispatch extension rather than a large flat blob.

6. **Clearer product positioning**  
   The product is neither dispatch-only nor fully generic. It is a system-definition framework with a first-class dispatch extension.

---

## 13. Recommended Next Design Work

The next iteration after this PRD should focus on the following concrete design artifacts:

1. **Boundary-modeling decision package (interfaces vs integrations)**
   Lock the Layer 1 rule that **interfaces and integrations both remain top-level sections**, with distinct roles:
   - interfaces define boundary surfaces and contracts,
   - integrations define concrete wiring between boundaries.

   The next schema draft should include explicit validation rules:
   - every integration must reference a declared source and target interface,
   - interfaces may exist without integrations (declared but not yet connected),
   - integrations may not define inline contracts that bypass interface declarations.

2. **Layer 1 primitive decision record**  
   Explicitly justify each primitive and each non-primitive.

3. **Dispatch surface normalization decision (dispatch block vs interfaces)**
   Clarify that submit/query/history surfaces are modeled as **normal Layer 1 interfaces** and not as a separate top-level primitive family inside `dispatch`.

   The `dispatch` block should keep only dispatch-specific semantics (work types, lifecycle models, correlation rules, status detail rules). Surface-specific behavior should be linked by reference to Layer 1 interfaces.

4. **Lifecycle transition rule model upgrade**
   Extend transition declarations beyond simple `from -> to` edges with an optional rule envelope:
   - preconditions/guards,
   - actor or source constraints,
   - idempotency/re-entry behavior,
   - required status detail fields,
   - transition side-effect classifications (declarative tags only, not execution logic).

   Keep plain `from -> to` as the minimum valid form, with richer rule fields optional.

5. **Canonical dispatch API surface definition**
   Decide the standard submit/query/history patterns the framework should assume and map each pattern to Layer 1 interface definitions.

6. **Generated asset matrix**
   Define exactly which outputs are generated from which manifest sections.

7. **Two worked examples**
   One generic integrated system using only Layer 1, and one event dispatch system using Layers 1 and 2.

---

## 14. Open Questions

1. What is the smallest acceptable Layer 1 component taxonomy, if any?
2. Should interfaces be attached only to components, or can the system itself expose first-class interfaces?
3. How opinionated should the framework be about standard envelope shapes for contracts?
4. Should lifecycle models be reusable across multiple work types by reference?
5. How much of dispatch-attempt representation belongs in Layer 2 versus later realization tooling?
6. How strict should transition guard semantics be (expression language vs bounded condition types)?
7. At what point, if any, should persistence concepts become a formal extension distinct from Layer 2?
8. What generated artifacts are mandatory versus optional in v1?
