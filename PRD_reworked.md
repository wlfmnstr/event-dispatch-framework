# Declarative System Definition Framework — Product Requirements Document

**Status:** Draft v0.2  
**Author:** Engineering Architecture  
**Last Updated:** 2026-03-08

---

## 1. Executive Summary

This product defines systems declaratively.

Its purpose is to let engineers describe the shape of a system, its interfaces, its integration points, and—when applicable—the lifecycle and state model of dispatched work, using structured manifests and schemas rather than repeated design-by-conversation.

The product is organized into three conceptual layers:

1. **Layer 1 — Declarative System Universals**  
   A general-purpose declarative model for describing system boundaries, ingress and egress interfaces, integration mechanisms, contracts, and message shapes.

2. **Layer 2 — Declarative Event Dispatch State Model**  
   A specialized extension for systems that accept work, dispatch it to external processors or downstream systems, and then track lifecycle state, progress, and outcomes over time.

3. **Layer 3 — Realization / Generation**  
   A future-oriented layer that consumes the declarative assets from Layers 1 and 2 and uses them to generate implementation artifacts, scaffolds, infrastructure definitions, and other downstream outputs.

The center of gravity of the product is Layers 1 and 2. Those are the actual product definition. Layer 3 exists to make the definitions operational, but it is downstream of the core language and should not dominate the PRD.

---

## 2. Problem Statement

Today, teams repeatedly re-architect systems that share the same basic shape.

At a broad level, teams repeatedly redefine:
- what the main boxes in a system are,
- how those boxes communicate,
- what interfaces and contracts exist at each system boundary,
- what requests, responses, and messages look like,
- which integration mechanisms are used between components and external systems.

For event dispatch systems specifically, teams also repeatedly re-architect:
- how incoming work is represented,
- how dispatch attempts are tracked,
- how lifecycle statuses are modeled,
- how outcomes and failures are recorded,
- how asynchronous progress is reconciled back into a canonical state model,
- how APIs and storage models align around those lifecycle concepts.

The repeated pain is not primarily “how do we write an OpenAPI spec.” The repeated pain is that the same categories of systems keep being re-thought from scratch, especially around lifecycle and state tracking.

This creates four problems:

1. **Repeated design work**  
   Engineers keep solving the same structural problems instead of specifying only what is unique.

2. **Inconsistent patterns**  
   Similar systems end up with different envelopes, different naming, different lifecycle semantics, and different integration conventions.

3. **Slow delivery**  
   Too much effort is spent rediscovering architecture and contracts before implementation can begin.

4. **Weak reuse**  
   Best practices remain embedded in people and teams rather than encoded into a declarative system definition language.

---

## 3. Product Vision

A developer should be able to define a system by authoring a structured manifest that captures only the system-specific parts.

From that definition, the framework should be able to produce canonical declarative assets such as:
- JSON Schemas,
- OpenAPI specifications,
- AsyncAPI specifications,
- integration contracts,
- lifecycle/state definitions,
- examples,
- documentation,
- and, later, inputs to realization tooling.

The product vision is not “generate code first.” The product vision is:

**define systems once, in a precise declarative language, and make those definitions durable, reusable, and operational.**

---

## 4. Product Boundary

This PRD intentionally separates three concerns.

### In Scope

- Defining a declarative language for describing systems
- Defining a manifest structure as the source of truth
- Defining what universal system primitives exist
- Defining what additional primitives are needed for event dispatch systems
- Producing machine-readable artifacts from those definitions
- Establishing a clean handoff to downstream realization tooling

### Explicitly De-emphasized in This PRD

- Detailed phased delivery plans
- Builder orchestration strategy
- Repo topology decisions
- Organization-specific scaffolding mechanics
- Detailed code generation workflows
- Detailed infrastructure automation design

Those concerns matter, but they are downstream of getting the declarative model right.

---

## 5. Users and Primary Use Cases

### Primary User

An engineer, architect, or technical product/data analyst who is defining a system that:
- exposes interfaces,
- integrates with other systems,
- exchanges structured messages,
- and in many cases dispatches work and tracks state over time.

### Primary Use Cases

1. **Define a system boundary**  
   Describe the boxes that are part of a system and the interfaces they expose.

2. **Define integration contracts**  
   Specify how components and external systems communicate and what those messages look like.

3. **Define an event dispatch system**  
   Extend the universal system definition with lifecycle and state semantics for dispatched work.

4. **Generate canonical artifacts**  
   Produce schemas, specs, examples, and documentation from the declarative definition.

5. **Use the resulting assets downstream**  
   Feed those assets into implementation, generation, review, or realization workflows.

---

## 6. Core Product Thesis

The product should not begin as a generic “model anything” framework.

It should begin with a **small universal core** plus a **high-value event dispatch extension**.

That means:
- Layer 1 should be broad enough to describe the universal shape of integrated systems.
- Layer 2 should solve the actual recurring pain around event dispatch lifecycle and state tracking.
- Layer 3 should exist as a consumer of those definitions, not as the center of the product.

This gives the product focus without trapping it in a narrow one-off solution.

---

## 7. Product Structure

## 7.1 Layer 1 — Declarative System Universals

Layer 1 defines the most reusable primitives for describing systems broadly.

It is intended for systems that can be understood as one or more bounded components with interfaces and integrations between them.

### Purpose

To provide a declarative language for describing:
- system identity,
- component boundaries,
- ingress and egress,
- interface types,
- integration mechanisms,
- request/response or message contracts,
- and canonical schemas associated with those interactions.

### Layer 1 Design Principle

Layer 1 should describe **how boxes interact**, not the full internal implementation of each box.

It is primarily an external/system-boundary definition layer.

### Universal Primitives

Layer 1 should define at least the following primitives:

- **System**  
  The overall bounded thing being defined.

- **Component**  
  A logical box within the system, such as an API service, processor, worker, adapter, or batch job.

- **Interface**  
  An exposed boundary where a component can receive or emit information.

- **Ingress**  
  A way information enters a component or system.

- **Egress**  
  A way information exits a component or system.

- **Integration**  
  A declared connection between two boundaries, internal or external.

- **Contract**  
  The schema or specification that defines what can cross an interface.

- **Interaction Mechanism**  
  The transport or communication pattern used by an integration.

### Supported Interaction Mechanism Categories

The framework should assume a bounded, opinionated set of common integration mechanisms, such as:
- synchronous HTTP/API,
- asynchronous message queue or topic,
- event stream,
- file-based exchange,
- scheduled/batch exchange,
- callback/webhook,
- polling-based retrieval,
- custom adapter type where necessary.

The goal is not to model every possible mechanism in existence. The goal is to make the common mechanisms first-class.

### Expected Layer 1 Outputs

From Layer 1, the system should be able to generate:
- JSON Schemas,
- OpenAPI specs,
- AsyncAPI specs,
- interface and integration reference docs,
- example messages and payloads,
- system boundary diagrams or diagram-ready metadata.

### Important Constraint

Layer 1 should not assume that all systems are stateful event dispatch systems.

Statefulness may appear later as an extension.

---

## 7.2 Layer 2 — Declarative Event Dispatch State Model

Layer 2 extends Layer 1 for a specific and recurring class of systems:

A system receives requested work, dispatches that work to an external processor or integrated system, then tracks the lifecycle of that work over time.

This is the part that solves the actual recurring pain.

### Purpose

To provide a declarative way to define the lifecycle and state model of event dispatch systems so teams stop repeatedly inventing the same architecture and data model.

### Layer 2 Design Principle

Layer 2 should treat state tracking as a first-class concern.

Not every system needs it, but when a system does need it, it is too important to leave implicit.

### Layer 2 Core Concepts

Layer 2 should define at least the following additional primitives:

- **Dispatch Request / Work Item**  
  The unit of work accepted by the system.

- **Action Type**  
  The kind of work being requested.

- **Canonical Event Record**  
  The stored representation of a submitted work item.

- **Dispatch Attempt**  
  An outbound attempt to hand the work to an external or downstream system.

- **Lifecycle Status**  
  A declared state in the progression of work.

- **Status Transition**  
  An allowed movement between lifecycle states.

- **Outcome Detail**  
  Structured success, failure, or progress detail associated with a lifecycle update.

- **Status Ingress Mechanism**  
  How the system receives or determines downstream progress or completion.

- **Correlation Model**  
  How a unit of work is identified across internal and external systems.

- **Terminality Model**  
  Which statuses are terminal and what they imply.

### What Layer 2 Should Make Declarative

Layer 2 should let a system author declare:
- what kinds of work the system accepts,
- what payload shapes each work type has,
- what statuses exist,
- what statuses are valid transitions,
- what status categories exist,
- what details are required for specific statuses,
- how downstream progress is learned,
- how correlation works,
- and what canonical APIs or retrieval patterns exist for observing state.

### Expected Layer 2 Outputs

From Layer 2, the system should be able to generate:
- event/request schemas,
- status record schemas,
- lifecycle definitions,
- state-model documentation,
- canonical API specs for submit/query/status-history patterns,
- example request and lifecycle instance data,
- observability or reporting contracts.

### Important Constraint

Layer 2 should be an extension of Layer 1, not a separate product.

An event dispatch system should still be describable in Layer 1 terms. Layer 2 adds lifecycle semantics on top.

---

## 7.3 Layer 3 — Realization / Generation

Layer 3 is the downstream consumption layer.

Its purpose is to use the assets defined by Layers 1 and 2 to help generate or realize actual systems.

### Purpose

To consume declarative assets and turn them into implementation-adjacent outputs such as:
- application scaffolds,
- infrastructure definitions,
- configuration files,
- docs packages,
- repo contributions,
- and other generated artifacts.

### Role in This PRD

Layer 3 should be acknowledged, but not over-specified.

This PRD should define Layer 3 mainly as:
- a consumer of Layers 1 and 2 outputs,
- a downstream realization concern,
- and a place where builder/generator contracts may later exist.

The PRD should avoid making Layer 3 the dominant design concern until Layers 1 and 2 are stable.

---

## 8. Manifest Strategy

The product should use a manifest as the source of truth.

### Manifest Principles

The manifest should:
- capture only what varies,
- encode explicit declarations rather than implementation detail,
- support progressive authoring,
- be validatable by schema,
- and cleanly separate Layer 1 and Layer 2 concerns.

### Recommended Structure

The manifest should be organized into sections roughly like:
- system identity,
- components,
- interfaces,
- integrations,
- contracts/schemas,
- optional event dispatch extension,
- optional realization hints.

### Separation Within the Manifest

The PRD should explicitly distinguish:
- **Layer 1 sections** that are universal,
- **Layer 2 sections** that are specific to event dispatch state modeling,
- **Layer 3 sections** that may exist later as optional downstream consumption hints.

That way the same manifest family can support both broad systems and dispatch-specific systems without collapsing them into one undifferentiated blob.

---

## 9. Product Positioning Decision

The product should be positioned as:

**a declarative system definition framework with a first-class event dispatch state extension.**

This is the recommended middle path because it preserves both:
- immediate usefulness for current dispatch-system work,
- and future extensibility for adjacent categories of systems.

### Why Not Event Dispatch Only

If the product is defined too narrowly, it risks baking every concept around one system type and making later expansion awkward.

### Why Not Fully Generic From Day One

If the product is defined too generically, it risks becoming an abstract modeling framework that does not solve the painful problem that motivated the work.

### Why the Recommended Position Works

It keeps the universal core small and reusable, while allowing the event dispatch layer to carry the real value and specificity.

---

## 10. Functional Requirements

### 10.1 Layer 1 Functional Requirements

The system must:
- allow definition of a system and its components,
- allow declaration of interfaces and integration points,
- allow declaration of interaction mechanisms,
- allow declaration of request/response/message contracts,
- generate machine-readable interface artifacts,
- generate documentation and examples,
- validate manifests and generated outputs.

### 10.2 Layer 2 Functional Requirements

The system must:
- allow declaration of accepted work types,
- allow declaration of lifecycle statuses,
- allow declaration of transition rules or lifecycle semantics,
- allow declaration of status-specific detail requirements,
- allow declaration of correlation and identity concepts,
- generate canonical state and lifecycle artifacts,
- generate API and schema artifacts for submit/query/status-history patterns,
- validate examples and instances against the generated contracts.

### 10.3 Layer 3 Functional Requirements

The system should eventually:
- expose a stable package of declarative outputs,
- support downstream tooling consuming those outputs,
- make it possible for realization tooling to remain decoupled from the core definition engine.

---

## 11. Non-Functional Requirements

The product should be:

- **Declarative-first**  
  Definitions should describe what a system is, not prescribe every implementation step.

- **Opinionated but bounded**  
  The product should encode common patterns and conventions without trying to model all software.

- **Composable**  
  Layer 2 must build cleanly on Layer 1.

- **Validatable**  
  Manifests and outputs should be schema-validatable.

- **Deterministic**  
  The same input should produce the same outputs.

- **Portable**  
  Generated artifacts should rely on standard formats where possible.

- **Useful before realization**  
  Layers 1 and 2 should deliver value even if no code or infrastructure is generated.

---

## 12. What This PRD Should Replace in the Previous Draft

The reworked PRD should explicitly remove or revise the following patterns from the previous version:

1. **Remove phased rollout framing**  
   Do not structure the product as Phase 1 / Phase 2 / Phase 3 delivery slices.

2. **Reduce realization-layer dominance**  
   Do not let builders, repo flows, or orchestration mechanics define the product.

3. **Recast the product around layers**  
   Replace phase-based language with Layer 1 / Layer 2 / Layer 3 conceptual structure.

4. **Broaden the foundation**  
   Rewrite the first half of the document to describe universal system-definition primitives, not just dispatch systems.

5. **Sharpen the core pain solved**  
   Make the event dispatch state model the specialized value layer that addresses the repeated architecture pain.

6. **Keep realization downstream**  
   Preserve the idea that declarative assets should later feed generation and realization, but keep that as a dependent concern.

---

## 13. Recommended Outline for the Next PRD Revision

The next edited PRD should be organized roughly as follows:

1. Executive Summary  
2. Problem Statement  
3. Vision  
4. Product Boundary  
5. Users and Use Cases  
6. Core Product Thesis  
7. Layer 1 — Declarative System Universals  
8. Layer 2 — Declarative Event Dispatch State Model  
9. Layer 3 — Realization / Generation  
10. Manifest Strategy  
11. Functional Requirements  
12. Non-Functional Requirements  
13. Open Questions

---

## 14. Open Questions

The next round of design work should answer these questions:

1. What is the minimum irreducible set of Layer 1 primitives?
2. What exact manifest structure cleanly separates Layer 1 from Layer 2?
3. How opinionated should Layer 2 be about lifecycle transitions versus allowing free-form status sets?
4. What identity and correlation model should be canonical across dispatch systems?
5. How much of storage/state representation belongs in Layer 2 versus remaining an implementation concern?
6. Which interaction mechanisms are first-class versus extension points?
7. What package of generated outputs is the stable handoff into Layer 3?
8. What terminology should become the durable product language?

---

## 15. Immediate Next Steps

1. Rewrite the existing PRD around the layered model in this document.
2. Remove all phase-oriented language.
3. Rewrite the current “architecture” section into Layer 1, Layer 2, and Layer 3 sections.
4. Rewrite the current manifest section to separate universal primitives from event-dispatch extensions.
5. Reduce detailed builder/orchestration content to a short downstream realization section.
6. Start a focused design pass on the Layer 1 primitive set.
7. Start a focused design pass on the Layer 2 state/lifecycle primitive set.

