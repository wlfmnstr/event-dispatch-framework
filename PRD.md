# Event Dispatch System Generator — Product Requirements Document

**Status:** DRAFT v0.1
**Author:** [Engineering Architecture]
**Last Updated:** 2026-03-03

---

## 1. Executive Summary

The Event Dispatch System Generator is a developer tool that standardizes and accelerates the creation of event dispatch systems. It takes a structured description of what makes a particular dispatch system unique (its events, statuses, integrations, and infrastructure) and produces the declarative artifacts and orchestrated outputs needed to stand up that system.

The tool's core responsibility is **declarative generation**: producing schemas, API specifications, and other machine-readable definitions that describe the system's data structures and interfaces. These declarative artifacts then serve as deterministic inputs to downstream builders and tooling — tools that may generate application code, infrastructure-as-code, other implementation artifacts, or simply perform automatable tasks needed for event dispatch systems to come into existence.

The tool eliminates the repeated design work that every new event dispatch system currently requires. Instead of each system re-solving the same structural questions (how to envelope events, how to track status, how to define integration contracts), engineers describe only what's unique about their system and the generator handles the rest.

**Expected scale:** Many event dispatch systems over time.
**Primary users:** Engineering teams, self-serve.
**Primary tech stack:** .NET/C# for dispatch services (though other languages and frameworks should be arbitrary), Terraform for IaC (though other IaC tooling like Pulumi, Bicep, Etc. should also be arbitrary), Azure for hosting (this one is likely fairly stardard across this enterprise).

---

## 2. Problem Statement

Standing up a new event dispatch system today involves:

1. **Repeated structural design** — Every system re-solves how to structure events, track statuses, define API contracts, and handle integration patterns. These decisions should be made once and inherited. Queues, batch files, API calls, and other common external system integrations should be defined broadly once and not need repeated boiler-plating and bootstrapping.
2. **Inconsistency** — Without a shared standard, each system invents its own field names, envelope shapes, status vocabularies, and integration patterns. This makes cross-system tooling, monitoring, and reasoning harder.
3. **Slow bootstrapping** — From "we need to dispatch events to system X and monitor their progress" to "we have a running dispatch service" involves manual creation of schemas, API contracts, application code, infrastructure, and organizational plumbing (APIM, RBAC, resource groups). Much of this is boilerplate that could be generated and/or automated.
4. **Tribal knowledge** — The decisions about what a dispatch system needs (database, queues, APIM registration, etc.) live in people's heads. When new systems are needed, the same conversations happen again. Further, engineering-level best practices and patterns (like sharing libraries and code, approaches to monorepos, where to put different IaC stuffs, etc.) should be baked into the tooling and abstractions and made available and useful at bootstrap-time - rather than relying on engineering documentation surrounding these patterns and practices be up-to-date and (re)discovered by developers with each new systems creation.

---

## 3. Vision

A developer describes a new event dispatch system by filling in a **system manifest** — a structured document that captures only what varies between systems. The generator:

1. **Produces declarative artifacts** — JSON Schemas, OpenAPI specs, AsyncAPI specs, and documentation that fully and precisely define the system's data structures and interfaces.
2. **Orchestrates downstream builders** — Invokes external generation tools (dotnet project generator, Terraform builders, etc.) with the right inputs derived from the declarative artifacts and manifest, producing application code, IaC, and organizational contributions.
3. **Validates at every layer** — The manifest is validated against a meta-schema. Generated schemas are validated as well-formed. Generated instances are validated against schemas. Downstream builder inputs are validated against builder contracts.

The end state is: **describe once, generate everything deterministically**.

---

## 4. Users & Use Cases

### Primary User: Application Engineer

An engineer who needs to stand up a new event dispatch system. They know:
- What events their system will dispatch (action types and payload shapes)
- What statuses those events will pass through
- What external system they're integrating with and how (queue, file, API)

They don't want to think about:
- Envelope structure, canonical field names, or schema design patterns
- Boilerplate application architecture for the dispatch service
- Infrastructure provisioning details
- Organizational registration (APIM, resource groups, RBAC)

### Use Cases

| # | Use Case | Description |
|---|---|---|
| UC-1 | **Guided discovery** | Engineer knows they need to dispatch events but hasn't nailed down the details. Gets starter schemas and a manifest that guides them toward the decisions they need to make. |
| UC-2 | **New system from scratch** | Engineer creates a system manifest, runs the generator, gets a fully scaffolded dispatch system with all artifacts. |
| UC-3 | **Add events to existing system** | Engineer updates an existing manifest to add new action types or statuses, re-runs generation to get updated schemas and specs. |
| UC-4 | **Validate existing instances** | Engineer validates event or status record JSON instances against generated schemas to verify conformance. |

---

## 5. Core Concepts & Terminology

| Term | Definition |
|---|---|
| **Event Dispatch System** | A system that receives events, dispatches them to an external processor, tracks their status through a lifecycle, and exposes that status via APIs or other mechanisms. |
| **System Manifest** | A structured document (YAML) that describes everything unique about a particular event dispatch system. The single input to the generator. |
| **Canonical Schema** | The base Event Envelope and Status Record JSON Schemas that define the invariant envelope structure shared by all dispatch systems. |
| **Extended Schema** | A domain-specific schema that inherits from a canonical schema and constrains the open slots (payload, detail, etc) for a particular system. |
| **Declarative Artifact** | Any generated output that is a machine-readable definition rather than executable code: schemas, OpenAPI specs, AsyncAPI specs, documentation (in some cases). |
| **Builder** | An external tool that accepts declarative artifacts (and/or manifest data) as input and produces implementation artifacts: application code, IaC, repositories and repo structure, etc. |
| **Builder Contract** | The interface specification that a builder must implement to be invocable by the generator's orchestration layer. |
| **Dispatch Integration** | The outbound mechanism by which the dispatch service sends events to an external system — e.g., Service Bus queue, file drop, REST API call. |
| **Status Ingress** | The mechanism by which the dispatch service learns about the outcome of dispatched events. Varies widely between systems — an RPA bot may post results to a queue, a host system may drop a batch file on a schedule, an API may offer a polling endpoint, or results may arrive only on error. Status ingress is often the most system-specific and complex integration concern, and the generator treats it as a customization point rather than something it constrains. |

### Identity Model

Two identifiers travel with every event. Understanding their roles and ownership is fundamental to the system design.

| Identifier | Who generates it | Required on input? | Purpose |
|---|---|---|---|
| **`event_id`** | Always the dispatch system | No — never consumer-provided | Primary key for a single event record. All status records reference it. Used for exact lookups: "what is the status of this specific event?" |
| **`correlation_id`** | Consumer when available, dispatch system as fallback | Optional — system generates one if not provided | Groups related events across a business workflow. Used for broad lookups: "what happened with this enrollment?" Bridges the consumer's world (their case number, request ID, etc.) to the dispatch system's world. |

Every stored event record always has both identifiers (the canonical schema requires both). The distinction is about *input* behavior: `event_id` is never accepted from consumers, and `correlation_id` is accepted but not demanded. Generated OpenAPI specs, dispatch service code, and API documentation must reflect this — `correlation_id` is optional on the request, present on the response, and auto-generated when absent.

---

## 6. Architecture Overview

The system has two primary layers and a realization workflow.

```
┌─────────────────────────────────────────────────────────┐
│                    SYSTEM MANIFEST                       │
│              (YAML — what makes this system unique)      │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   DECLARATIVE CORE                       │
│                                                          │
│  Inputs:  System manifest + canonical schemas            │
│  Outputs: Extended schemas, OpenAPI spec, AsyncAPI spec, │
│           system documentation                           │
│                                                          │
│  • Validates manifest                                    │
│  • Generates extended event/status schemas               │
│  • Generates API specifications                          │
│  • Generates system documentation                        │
│  • Self-validates all outputs                            │
└────────────────────────┬────────────────────────────────┘
                         │
                    SYSTEM PACKAGE
               (directory of declarative
              artifacts in standard formats)
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                  REALIZATION LAYER                        │
│                                                          │
│  • Reads system package + tool configuration             │
│  • Determines which builders are needed                  │
│  • Invokes builders locally via standard contract        │
│  • Each builder produces changes in a target repo        │
│  • Presents summary to developer for review              │
│  • Developer selectively pushes when ready               │
└────────┬───────────┬───────────┬───────────┬────────────┘
         │           │           │           │
         ▼           ▼           ▼           ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
    │ dotnet  │ │  cloud  │ │resource │ │  apim   │
    │ service │ │   iac   │ │ groups  │ │ config  │
    │ builder │ │ builder │ │ builder │ │ builder │
    └─────────┘ └─────────┘ └─────────┘ └─────────┘
```

### 6.1 Why This Layering

**Declarative Core is the foundation.** Even without any builders, the declarative artifacts have standalone value: they give engineers validated schemas, API contracts, and documentation from day one. They can manually build the rest.

**The system package is the interface.** The declarative core produces a directory of artifacts in well-known standard formats (JSON Schema, OpenAPI, AsyncAPI, YAML). This system package is the stable contract between generation and realization. Builders consume from it. The generator writes to it. Neither needs to know about the other.

**Builders conform to a standard contract.** Every builder — whether it lives in this tool, in a target repo, or in its own repo — implements the same interface: receive a system package path, produce local changes in a target repository. New builders can be added without modifying the core, and builders can be developed and maintained independently.

**The manifest is the single source of truth for what the system is.** Everything flows from the manifest. The declarative core reads it to produce schemas and specs. The realization layer reads it (via the system package) to understand what the system needs. Builders receive the system package and extract what they need from it.

---

## 7. System Manifest Specification

The system manifest is a YAML file that captures everything unique about a particular event dispatch system. It is designed for **progressive complexity** — you can start with just a name and some action types and get useful output. As you add more detail, the generator produces more.

### 7.1 Manifest Structure

```yaml
# ============================================================
# SYSTEM IDENTITY
# ============================================================
system:
  name: host-insurance-update          # URL-safe slug, required
  title: Host Insurance Update          # Human-readable, required
  description: >                        # Brief description, required
    Dispatches insurance update operations to host systems via RPA.
  version: "1.0"                        # System version, optional

# ============================================================
# EVENTS
# ============================================================
events:
  action_types:
    host.insurance.update:
      title: Insurance Update Request
      description: >
        A request to perform one or more insurance record
        operations in a host system.

      # Optional: JSON Schema fragment constraining the payload.
      # If omitted, payload remains an open object (the canonical default).
      payload_schema:
        type: object
        required: [accountNumber, actions]
        properties:
          accountNumber:
            type: string
          actions:
            type: array
            items:
              type: object
              required: [actionType, actionPayload]
              properties:
                actionType:
                  type: string
                actionPayload:
                  type: object

      # Optional: example payload for documentation and validation.
      example_payload:
        accountNumber: "ACC-12345"
        actions:
          - actionType: addInsurance
            actionPayload:
              insurerName: "Acme Insurance"

# ============================================================
# STATUSES
# ============================================================
statuses:
  values:
    received:
      description: Event received and validated by the dispatch service.
      category: queued                 # Optional: maps to a universal work-state
    dispatched:
      description: Event sent to the integration point.
      category: in_flight
    in_progress:
      description: External system has begun processing.
      category: in_flight
    completed:
      description: All actions completed successfully.
      category: terminal_success
      requires_detail: true
      detail_schema:
        type: object
        required: [actionResults]
        properties:
          actionResults:
            type: array
    failed:
      description: Processing failed.
      category: terminal_failure
      requires_detail: true
      detail_schema:
        type: object
        required: [errorCode, errorMessage]
        properties:
          errorCode:
            type: string
          errorMessage:
            type: string
    cancelled:
      description: Event was cancelled before completion.
      category: terminal_cancelled

# ============================================================
# INTEGRATIONS  (Phase 3+)
# ============================================================
integrations:
  dispatch:                             # How events reach the external system
    type: service-bus-queue             # service-bus-queue | service-bus-topic |
                                        # file-drop | rest-api | custom
    properties:
      queue_name: insurance-updates-outbound

  status_ingress:                       # How statuses come back
    type: service-bus-queue
    properties:
      topic_name: insurance-updates-status
      subscription_name: dispatch-service

# ============================================================
# INFRASTRUCTURE
# ============================================================
# Declares what the system needs to function. The realization
# layer and its builders determine how to provision these.
infrastructure:
  - type: container-app               # The dispatch service needs hosting
  - type: database                     # State/status tracking needs persistence
  - type: service-bus                  # Dispatch and status ingress need messaging
    role: dispatch
  - type: service-bus
    role: status-ingress
  - type: api-gateway                  # The dispatch API needs gateway registration

# ============================================================
# METADATA CONVENTIONS  (informational, never enforced)
# ============================================================
metadata_conventions:
  environment:
    type: string
    description: Deployment environment.
    example: production
  tenantId:
    type: string
    description: Organization identifier.
```

### 7.2 Progressive Complexity

The manifest is designed so that you can start small and add sections as needed:

| What You Provide | What You Get |
|---|---|
| `system` + `events` (action types, names only) + `statuses` (names only) | Canonical schemas with action_type const/enum and status enum. Payload and detail remain open. System documentation. |
| Above + `payload_schema` / `detail_schema` on events/statuses | Extended schemas with constrained payload/detail shapes. Full validation of instances becomes possible. |
| Above + `integrations` | AsyncAPI specs for message-based integrations. OpenAPI spec for the dispatch API. Integration documentation. |
| Above + `infrastructure` | Infrastructure requirements documentation. System package is complete and ready for realization. |

This progression maps to natural workflow: think about your data first, then your integrations, then your infrastructure needs. The system package produced at any stage has standalone value and can be consumed by builders whenever realization is desired.

### 7.3 What Varies vs. What's Inherited

The manifest captures **only what varies** between event dispatch systems. Everything else is inherited from the canonical schemas and builder templates.

**Always varies (required in manifest):**
- System identity (name, title, description)
- Action types (what kinds of events this system dispatches)
- Status values (what lifecycle states events pass through)

**Sometimes varies (optional in manifest, has sensible defaults):**
- Payload/detail schemas (defaults to open object if not specified)
- Integration mechanisms (dispatch and status ingress types)
- Infrastructure requirements (what resources the system needs)
- Status category mapping for cross-system observability

**Never varies (inherited, not in manifest):**
- Envelope structure (canonical event/status schemas)
- Dispatch service architecture (API + dispatcher + state tracker)
- Status tracking patterns
- Monitoring and observability patterns

### 7.4 Why YAML

The manifest is YAML rather than JSON because:
- Multi-line descriptions are natural (no escaped newlines)
- Comments are supported (useful for documenting decisions)
- Less syntactic noise for a human-authored document
- JSON Schema fragments within YAML are a well-understood pattern
- The tool reads YAML and can produce a canonical JSON representation if needed downstream

---

## 8. Declarative Outputs

The declarative core generates these artifacts from the system manifest. These are the tool's **primary outputs** and its core value proposition.

### 8.1 Extended Event Schema

A JSON Schema that composes the canonical Event Envelope via `allOf` with domain-specific constraints.

**Always generated:**
- `action_type` constrained to `const` (single action type) or `oneOf` (multiple)

**Generated when payload_schema is provided:**
- `payload` constrained to the specified schema fragment

**File:** `<system-name>/schemas/event-schema.json`

### 8.2 Extended Status Schema

A JSON Schema that composes the canonical Status Record via `allOf` with domain-specific constraints.

**Always generated:**
- `status` constrained to `enum` of declared status values

**Generated when detail_schema is provided:**
- Per-status `if/then` blocks constraining `detail` shape
- `required: [detail]` on statuses where `requires_detail: true`

**File:** `<system-name>/schemas/status-schema.json`

### 8.3 OpenAPI Specification

An OpenAPI 3.0 spec defining the dispatch service's HTTP API.

**Always generated (when `integrations` section is present or by default for the standard dispatch API):**
- `POST /events` — Submit a new event for dispatch
- `GET /events/{event_id}` — Retrieve an event and its current status
- `GET /events/{event_id}/statuses` — Retrieve the full status history for an event
- `GET /events?correlation_id={id}` — Query events by correlation
- Request/response schemas reference the generated extended schemas

**File:** `<system-name>/specs/openapi.yaml`

### 8.4 AsyncAPI Specification

An AsyncAPI 2.x spec defining the message-based integration contracts.

**Generated when `integrations` includes message-based types (service-bus-queue, kafka topic, event hub/grid, etc.):**
- Channel definitions for outbound event dispatch
- Channel definitions for inbound status ingestion
- Message schemas reference the generated extended schemas

**File:** `<system-name>/specs/asyncapi.yaml`

### 8.5 Example Instances

Generated example JSON files for documentation and testing.

**Always generated:**
- Example event(s) — one per action type
- Example status records — full lifecycle (one record per status value)

**File:** `<system-name>/examples/`

### 8.6 System Documentation

Auto-generated Markdown documentation.

**Always generated:**
- System overview (from manifest identity)
- Event reference (action types, payload descriptions)
- Status reference (status values, detail descriptions, lifecycle diagram)
- File manifest (what was generated and where)

**Generated when integrations/infrastructure are present:**
- Integration reference (mechanisms, channels, contracts)
- Infrastructure reference (required resources)

**File:** `<system-name>/README.md`

### 8.7 Observability Contract

A machine-readable contract that tells downstream tooling how to interpret and aggregate this system's status data.

**Always generated:**
- Status category mapping (if provided in the manifest)
- Standard API paths for event and status retrieval
- Queryable dimensions (action_type, status, correlation_id, created_at)

**File:** `<system-name>/observability.yaml`

---

## 9. System Package & Builder Interface

### 9.1 The System Package

The system package is the directory of declarative artifacts produced by the declarative core. It is the **stable interface** between generation and realization. Every file in the system package is either a well-known standard format (JSON Schema, OpenAPI, AsyncAPI) or a structure defined by this tool (the manifest).

```
generated/<system-name>/
├── manifest.yaml                    # The source of truth for what this system is
├── schemas/
│   ├── event-schema.json            # JSON Schema draft-07
│   └── status-schema.json           # JSON Schema draft-07
├── specs/
│   ├── openapi.yaml                 # OpenAPI 3.x
│   └── asyncapi.yaml                # AsyncAPI 2.x/3.x (when applicable)
├── examples/
│   ├── event-<action-type>.json     # One per action type
│   └── status-lifecycle.json        # Full lifecycle example
├── observability.yaml               # Observability contract (status categories, API paths)
└── README.md                        # Auto-generated documentation

generated/catalog.yaml                 # Auto-maintained index of generated systems
```

Any tool that can read these standard formats can consume the system package. A builder that needs the OpenAPI spec reads `specs/openapi.yaml`. A builder that needs to know the system's infrastructure requirements reads `manifest.yaml`. No intermediate packaging or transformation is needed — the system package IS the contract.

### 9.2 Builders

A builder is any tool that consumes a system package (or parts of it) and produces implementation artifacts — application code, IaC, pipeline definitions, configuration contributions to existing repositories, etc.

**Where builders live.** Builders can live in any of these locations depending on what makes sense:
- **In this tool's repository** (`builders/` directory) — for builders maintained as part of the generator, such as IaC contributors
- **In a target repository** — for builders that live alongside the codebase they generate into, such as the existing dotnet project generator in the monorepo
- **In their own repository** — for standalone builders with independent release cycles

The tool's configuration tells it where to find each builder (a local path, a repo URL + branch, or a reference to a bundled builder).

### 9.3 Builder Contract

Every builder implements a standard interface so the realization layer can invoke it uniformly, regardless of where the builder lives or what it produces.

```
BUILDER CONTRACT v1
───────────────────

Input:
  The builder receives:
    --system-package <path>     Path to the system package directory
    --output-dir <path>         Where to write output (typically a local
                                clone/checkout of the target repository)
    --config <json>             Optional builder-specific configuration
                                (target repo paths, naming conventions, etc.)

  The builder reads whatever it needs directly from the system
  package: manifest.yaml for system details, schemas/ for data
  structures, specs/ for API contracts.

Output:
  The builder writes its artifacts to --output-dir and produces
  a result summary:
  {
    "status": "success" | "failure",
    "artifacts": [
      { "path": "...", "description": "..." }
    ],
    "errors": [],
    "warnings": []
  }
```

The contract is deliberately simple. The system package provides all the context a builder needs in standard formats. The builder reads what it understands and ignores the rest. This means:
- A dotnet builder reads `specs/openapi.yaml` and `schemas/` to generate models and API scaffolding
- A Terraform builder reads `manifest.yaml` to understand infrastructure requirements and generates `.tf` files
- An APIM contributor reads `specs/openapi.yaml` and generates API gateway configuration
- All of them implement the same CLI interface

### 9.4 Tool Configuration

The tool needs to know where its builders are and basic operational details (like which repo to target for each builder). This is the tool's own configuration — not an architectural layer, just practical setup.

```yaml
# .edsg/config.yaml (in the tool's project or user home)
builders:
  dotnet-container-app:
    source: git://ensemble-health/monorepo#builders/dispatch-generator
    description: Generates .NET dispatch service project
    default_config:
      target_repo: ensemble-health/monorepo

  cloud-resources:
    source: ./builders/cloud-resources    # Bundled in this tool
    description: Generates Terraform for cloud resources
    default_config:
      target_repo: ensemble-health/cloud-iac

  resource-groups:
    source: ./builders/resource-groups
    description: Contributes resource group definitions
    default_config:
      target_repo: ensemble-health/resource-groups

  access-control:
    source: ./builders/access-control
    description: Contributes RBAC/access control entries
    default_config:
      target_repo: ensemble-health/access-control

  apim-config:
    source: ./builders/apim-config
    description: Contributes API gateway registration
    default_config:
      target_repo: ensemble-health/apim-config
```

### 9.5 Realization Workflow

Realization is a local CLI operation. The tool clones/checks out the repositories it needs, runs the appropriate builders, and presents the developer with a complete picture of what was produced. The developer reviews and pushes when ready.

```
$ edsg realize generated/host-insurance-update/ --dry-run

  Host Insurance Update — Realization Plan
  ─────────────────────────────────────────

  1. dispatch-service (dotnet-container-app)
     → ensemble-health/monorepo  [new project: src/HostInsuranceDispatch]
     → ensemble-health/monorepo  [new pipeline: pipelines/host-insurance-dispatch.yml]

  2. cloud-resources (terraform)
     → ensemble-health/cloud-iac  [new module: event-dispatch/host-insurance-update/]
     Service Bus queue, SQL database, Container App

  3. resource-groups (iac-contributor)
     → ensemble-health/resource-groups  [modified: production/host-insurance.tf]

  4. access-control (iac-contributor)
     → ensemble-health/access-control  [modified: policies/host-insurance-dispatch.tf]

  5. api-gateway (apim-contributor)
     → ensemble-health/apim-config  [new: apis/host-insurance-dispatch.yaml]

  5 repositories affected. Run without --dry-run to prepare branches.

$ edsg realize generated/host-insurance-update/

  ✓ All changes prepared locally.

  Ready to push:
  [1] ensemble-health/monorepo          branch: feature/host-insurance-dispatch
  [2] ensemble-health/cloud-iac         branch: feature/host-insurance-dispatch
  [3] ensemble-health/resource-groups   branch: feature/host-insurance-dispatch
  [4] ensemble-health/access-control    branch: feature/host-insurance-dispatch
  [5] ensemble-health/apim-config       branch: feature/host-insurance-dispatch

  Push all? Or select individually? [a/s/q]
```

**Dry-run is first-class.** `--dry-run` shows exactly what would happen without making any changes. This is available from the first version of the realization layer — developers should always be able to preview before committing.

### 9.6 Known/Planned Builders

| Builder | What It Produces | Where It Lives |
|---|---|---|
| `dotnet-container-app` | .NET dispatch service with API, dispatcher, state tracking, models from schemas, ADO pipeline | Target monorepo (existing builder, to be updated to consume system packages) |
| `cloud-resources` | Terraform for system-specific cloud resources (Service Bus, database, Container App) | Bundled in this tool |
| `resource-groups` | Resource group IaC contributions | Bundled in this tool |
| `access-control` | RBAC/access control IaC contributions | Bundled in this tool |
| `apim-config` | API gateway registration from OpenAPI spec | Bundled in this tool |

---

## 10. Phased Delivery Plan

### Phase 1: Canonical Schemas — MVP

**Status:** The canonical Event Envelope and Status Record schemas mostly documented here in the PRD, though additional work may need to be put into making them complete and producing the official schemas.

**Artifacts produced:**
- `system-assets/event-schema.json`
- `system-assets/status-record-schema.json`
- example event and status records utilizing both of those schemas.

### Phase 2: System Manifest + Declarative Core — MVP

**Goal:** Engineers can describe a new event dispatch system in a YAML manifest and get a validated system package (schemas, API specs, documentation). The existing dotnet project generator can be manually invoked with the generated artifacts.

**Scope:**
- Define and implement the system manifest schema (YAML, validated against a meta-schema)
- Implement the declarative core:
  - Extended event schema generation (from manifest `events` section)
  - Extended status schema generation (from manifest `statuses` section)
  - OpenAPI spec generation (standard dispatch API)
  - Example instance generation
  - System documentation generation
- Self-validation of all generated outputs
- CLI interface: `edsg init`, `edsg generate`, `edsg validate`
- Migrate existing domain map concept into the new manifest structure (the current generator work is subsumed, not discarded)

**Inputs:** System manifest (YAML)
**Outputs:** System package (extended schemas, OpenAPI spec, examples, documentation)

**What the user can do after Phase 2:**
- Describe a new dispatch system in a manifest
- Get validated, production-ready schemas and an OpenAPI spec
- Manually invoke the existing dotnet project generator with the system package
- Validate event/status instances against generated schemas

### Phase 3: Builder Contract + Realization Layer

**Goal:** The builder contract is formalized. The realization layer can invoke builders locally with dry-run support. The dotnet builder is updated to consume system packages via the standard contract. First IaC builders are created.

**Scope:**
- Builder contract v1 specification and implementation
- Realization layer: `edsg realize` with `--dry-run`
- Tool configuration for builder locations and target repos
- Update existing dotnet builder to implement the builder contract
- Manifest `integrations` section support + AsyncAPI spec generation
- First IaC builders (cloud-resources, resource-groups)

**What the user can do after Phase 3:**
- Run `edsg realize --dry-run` and see exactly what would be created across which repos
- Run `edsg realize` and get local branches prepared across multiple repos
- Review changes and push selectively

### Phase 4: Full Builder Suite

**Goal:** All known builders are implemented. End-to-end realization from manifest to prepared branches across all required repos.

**Scope:**
- Access control builder
- APIM configuration builder
- Enhance dotnet builder to consume integration details (generate queue/topic client code, status ingestion handlers)
- Manifest `infrastructure` section support

**What the user can do after Phase 4:**
- Describe a complete system in a manifest
- Run `edsg realize` and get the full set of repo contributions prepared: dispatch service, IaC, resource groups, access control, API gateway

### Phase 5: Polish + Advanced Capabilities

**Goal:** Refine the end-to-end experience based on real usage. Add capabilities that emerge as needed.

**Scope:**
- Manifest diffing (show what changed between manifest versions)
- Re-realization (update existing system repos when manifest changes)
- Additional builders as new patterns emerge
- CI/CD integration (run realization from pipelines, not just developer machines)

---

## 11. MVP Specification (Phase 2 Detail)

### 11.1 CLI Interface

The tool is invoked as a CLI. The command name is `edsg` (Event Dispatch System Generator).

```bash
# Initialize a new system manifest (interactive or from template)
edsg init [--name <system-name>]

# Validate a manifest without generating anything
edsg validate <manifest.yaml>

# Generate the system package (declarative artifacts) from a manifest
edsg generate <manifest.yaml> [--output-dir <dir>]

# Validate an instance against a generated schema
edsg validate-instance <schema.json> <instance.json>

# Preview what realization would produce (Phase 3+)
edsg realize <system-package-dir> --dry-run

# Run realization — invoke builders, prepare local branches (Phase 3+)
edsg realize <system-package-dir>
```

**Default output directory:** `generated/<system.name>/` relative to project root.

### 11.2 `edsg init`

Creates a starter manifest with the required sections and commented-out optional sections. The goal is to make it obvious what to fill in.

Optionally interactive: prompts for system name, first action type, and initial statuses. Writes the manifest with those values pre-filled.

### 11.3 `edsg generate`

The primary command. Reads a manifest, validates it, generates the system package.

**Steps:**
1. Parse YAML manifest.
2. Validate manifest against meta-schema. Report errors and stop if invalid.
3. Generate extended event schema(s).
4. Generate extended status schema.
5. Generate OpenAPI spec.
6. Generate AsyncAPI spec (when message-based integrations are declared).
7. Generate example instances.
8. Self-validate: all generated schemas are valid, all examples validate against schemas.
9. Generate system documentation.
10. Write system package to output directory.
11. Print summary of what was generated.

### 11.4 Generated Output Structure

```
generated/<system-name>/
├── manifest.yaml                    # The source of truth for this system
├── schemas/
│   ├── event-schema.json            # Extended event schema
│   └── status-schema.json           # Extended status schema
├── specs/
│   ├── openapi.yaml                 # Dispatch API specification
│   └── asyncapi.yaml                # Message contracts (when applicable)
├── examples/
│   ├── event-<action-type>.json     # One per action type
│   └── status-lifecycle.json        # Full lifecycle example
├── observability.yaml               # Observability contract
└── README.md                        # Auto-generated documentation

generated/catalog.yaml                 # Auto-maintained index of generated systems
```

### 11.5 MVP Acceptance Criteria

1. A minimal manifest (system identity + action type names + status names) produces valid extended schemas with constrained `action_type` and `status` fields, open `payload`/`detail`.
2. A full manifest (with `payload_schema` and `detail_schema`) produces schemas that constrain payload and detail shapes.
3. An OpenAPI spec is generated that correctly references the extended schemas and reflects the identity model (`event_id` system-generated, `correlation_id` optional on input).
4. All generated schemas pass validation.
5. All generated examples validate against their corresponding schemas.
6. `edsg validate-instance` correctly validates instances against generated schemas.
7. The system package structure is documented and stable enough that the existing dotnet project generator can be manually pointed at it.
8. An observability contract is generated for each system package.
9. A system catalog is maintained listing generated systems and their key metadata.

### 11.6 Relationship to Existing Work

The existing generator (`generator/generate.py`) and domain map concept are **not discarded**. They represent early work on the schema generation layer that the declarative core subsumes.

**What carries forward:**
- Canonical schemas (`system-assets/`) — unchanged, the foundation
- Schema composition pattern (`allOf` + `$ref`) — proven, reused
- Validation approach (`jsonschema` + `referencing`) — reused
- Self-validation pattern — reused

**What changes:**
- Domain maps (JSON) are superseded by system manifests (YAML) with a broader scope
- The generator CLI is replaced by `edsg` with a richer command set
- Output directory structure is reorganized
- The generator logic is refactored into a cleaner internal architecture (manifest parser → schema generator → spec generator → doc generator)

---

## 12. Design Principles

These principles govern the tool itself (as distinct from the schema design principles in CLAUDE.md which govern schema output).

### 12.1 Declarative First

The tool's primary value is producing declarative artifacts. Code generation and infrastructure provisioning are secondary capabilities enabled by the declarative foundation. Every feature should ask: "does this improve the declarative output, or does it depend on it?"

### 12.2 Progressive Complexity

A minimal manifest should produce useful output. Adding detail to the manifest should produce richer output. The tool should never require users to specify things they don't yet know or care about.

### 12.3 Deterministic Generation

The same manifest must always produce the same output. No randomness, no ambient state, no "it depends on what's already there." This is critical for the downstream builder model: builders can trust that their inputs are reproducible.

### 12.4 Validate Everything

Every layer validates its inputs and outputs. The manifest is validated. Generated schemas are validated. Generated examples are validated against schemas. Builder inputs are validated against contracts. Fail fast and loud.

### 12.5 Builders Conform to a Standard Contract

Builders — whether bundled in this tool, living in target repos, or maintained independently — all implement the same interface. The declarative core never contains builder logic. The system package is the boundary: the core writes to it, builders read from it.

### 12.6 Convention Over Configuration

Most event dispatch systems at the organization follow the same patterns. The tool should encode those patterns as defaults. The manifest only needs to capture deviations from convention. Example: if every system uses a SQL Server database, the manifest shouldn't require specifying `database.type: sql-server` — that should be the default.

### 12.7 Observable by Default

Systems generated by this tool should be legible to cross-system tooling without extra effort from engineers. The generator must produce the declarative contracts needed for aggregation, visibility, and work-state reporting across systems.

---

## 13. Open Questions

These are decisions that need to be made during implementation. They are flagged here rather than prematurely decided.

| # | Question | Context | Options |
|---|---|---|---|


---

## 14. Constraints & Non-Goals

### Constraints
- Generated schemas must be valid JSON Schema draft-07
- Generated schemas must use `allOf` composition with `$ref` to canonical schemas (never fork/copy)
- The canonical schemas in `system-assets/` are frozen; changes require a version bump
- The declarative core is Python-based (consistent with existing tooling, UV-managed)
- The builder contract is language-agnostic (CLI + JSON) — builders can be written in any language
- All tool operations (generation and realization) must be runnable from a developer's local machine

### Non-Goals
- **Runtime components** — The generator does not produce running services. It produces the artifacts needed to build them.
- **Custom business logic** — The generator handles structural concerns (schemas, API contracts, infrastructure). Domain-specific business rules, validation logic, and processing algorithms are written by engineers in the generated application code.
- **Monitoring dashboards and observability infrastructure** — The generator does not produce dashboards, alerting rules, or logging configuration. It does produce an observability contract (and catalog) so downstream tooling can aggregate and visualize system work states without custom per-system integration.
- **Migration tooling** — The generator creates new systems. Migrating existing non-conformant systems to the standard is out of scope.
- **UI/web interface** — The tool is CLI-first. A web interface is not planned.

---

## 15. Success Metrics

| Metric | Target | How Measured |
|---|---|---|
| Time to first valid schemas | < 30 minutes from starting a manifest | Engineer self-report |
| Time to scaffolded dispatch service | < 2 hours (including manifest authoring) | Engineer self-report |
| Schema conformance | 100% of generated schemas pass draft-07 validation | Automated (self-validation) |
| Instance conformance | 100% of generated examples validate against schemas | Automated (self-validation) |
| Systems generated | 5+ systems using the tool within 6 months of MVP | Count of manifest files |
| Structural consistency | All generated systems share canonical envelope structure | Schema composition audit |
| Cross-system visibility | Work-state aggregation possible from catalog + observability contracts | Catalog + contract audit |

---

## Appendix A: Glossary

| Term | Definition |
|---|---|
| EDSG | Event Dispatch System Generator (the tool described in this PRD) |
| Envelope | The canonical structure wrapping domain-specific content (Event Envelope, Status Record) |
| Open slot | A schema field (`payload`, `detail`, `metadata`) where `additionalProperties: true` allows domain content |
| Canonical field | A field defined in the base envelope schema (e.g., `event_id`, `status`, `created_at`) |
| Domain | The specific business area a dispatch system serves (e.g., insurance updates, eligibility checks) |
| System package | The directory of generated declarative artifacts — the stable interface between the declarative core and builders |
| Realization | The process of invoking builders against a system package to produce implementation artifacts across target repositories |

## Appendix B: Host Insurance Update — Manifest Example

This is how the existing reference implementation would be expressed as a system manifest:

```yaml
system:
  name: host-insurance-update
  title: Host Insurance Update
  description: >
    Dispatches insurance update operations (add, update, reorder)
    to host systems via RPA.

events:
  action_types:
    host.insurance.update:
      title: Insurance Update Request
      description: >
        A request to perform one or more insurance record operations
        in a host system.
      payload_schema:
        type: object
        required:
          - accountNumber
          - patientFirstName
          - patientLastName
          - clientName
          - actions
        properties:
          accountNumber:
            type: string
            description: The account number in the host system.
          patientFirstName:
            type: string
          patientLastName:
            type: string
          clientName:
            type: string
            description: Identifies which client configuration to use.
          actions:
            type: array
            minItems: 1
            items:
              type: object
              required:
                - actionType
                - actionPayload
              properties:
                actionType:
                  type: string
                  description: >
                    The type of insurance operation.
                    Domain vocabulary: addInsurance, updateInsurance,
                    reorderInsurance.
                actionPayload:
                  type: object
                  description: Action-specific data.
      example_payload:
        accountNumber: "ACC-98765"
        patientFirstName: Jane
        patientLastName: Doe
        clientName: acme-health
        actions:
          - actionType: addInsurance
            actionPayload:
              insurerName: BlueCross BlueShield
              policyNumber: "BCBS-2025-78432"
              groupNumber: "GRP-100"
              effectiveDate: "2025-01-15"
              subscriberRelationship: self

statuses:
  values:
    received:
      description: >
        Event received and validated by the dispatch service.
      category: queued
    dispatched:
      description: >
        Event sent to the RPA integration queue for processing.
      category: in_flight
      detail_schema:
        type: object
        properties:
          queueMessageId:
            type: string
            description: Message ID from the outbound queue.
    in_progress:
      description: >
        RPA bot has picked up the event and begun execution.
      category: in_flight
    completed:
      description: >
        All actions in the event completed successfully.
      category: terminal_success
      requires_detail: true
      detail_schema:
        type: object
        required:
          - actionResults
        properties:
          actionResults:
            type: array
            items:
              type: object
              required:
                - actionType
                - status
              properties:
                actionType:
                  type: string
                status:
                  type: string
                message:
                  type: string
    partialSuccess:
      description: >
        Some actions succeeded, others failed.
      category: in_flight
      requires_detail: true
      detail_schema:
        type: object
        required:
          - actionResults
        properties:
          actionResults:
            type: array
            items:
              type: object
              required:
                - actionType
                - status
              properties:
                actionType:
                  type: string
                status:
                  type: string
                errorCode:
                  type: string
                errorMessage:
                  type: string
    failed:
      description: >
        All actions failed or an unrecoverable error occurred.
      category: terminal_failure
      requires_detail: true
      detail_schema:
        type: object
        required:
          - errorCode
          - errorMessage
        properties:
          errorCode:
            type: string
          errorMessage:
            type: string
          actionResults:
            type: array
    cancelled:
      description: >
        Event was cancelled before completion.
      category: terminal_cancelled

integrations:
  dispatch:
    type: service-bus-queue
    properties:
      queue_name: host-insurance-updates-outbound
  status_ingress:
    type: service-bus-queue
    properties:
      queue_name: host-insurance-updates-status

infrastructure:
  - type: container-app
  - type: database
  - type: service-bus
    role: dispatch
  - type: service-bus
    role: status-ingress
  - type: api-gateway

metadata_conventions:
  environment:
    type: string
    description: Deployment environment.
    example: production
  priority:
    type: string
    description: Processing priority.
    example: normal
  tenantId:
    type: string
    description: Organization identifier.
    example: tenant-abc-123
  requestedBy:
    type: string
    description: System or user that initiated the event.
    example: enrollment-portal
  rpaBotId:
    type: string
    description: Identifier of the RPA bot executing the work (used in status records).
    example: bot-ins-prod-03
```
