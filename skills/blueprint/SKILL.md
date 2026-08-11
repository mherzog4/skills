---
name: blueprint
description: Design a change before you write it — typed contracts plus entrypoint-to-side-effect call stacks, as an implementation-ready handoff. Converts existing context into a blueprint, or grills you first when the context is thin. Invoke with `/blueprint`, `/blueprint billing retries`, or `/blueprint @src/queue/worker.ts`.
disable-model-invocation: true
---

# Blueprint

A blueprint is a **typed call-stack architecture handoff**: code-shaped contracts plus execution flows. Where precision matters, write TypeScript pseudocode instead of prose.

This skill is design-only. Do not implement. Return the blueprint inline. Save a file only when the user asks for one.

## Branch selection

1. Use **Path A: convert context to a blueprint** when the conversation, the docs, or the codebase already hold enough background to describe the change.
2. Use **Path B: grill first** when the user wants a new blueprint but has not supplied the problem, the constraints, the design direction, the affected code, or the acceptance criteria.

When the codebase can answer a question, inspect the codebase instead of asking.

Completion criterion: the branch follows from actual available context, and no architectural decision is invented.

## Path A: convert context to a blueprint

### 1. Load the standards and the local context

Load the `ts-standards` skill. It owns the error model, the parsing rules, the module roles, the naming rules, and the test seams that this blueprint must produce.

Then inspect the code and the docs for local precedent on each of those axes, plus local vocabulary and runtime patterns.

Completion criterion: the blueprint uses project vocabulary, and it introduces no pattern, library, Adapter, schema style, or test strategy before you check local precedent.

### 2. Extract the design problem

Fill the outline's **Context / Current State** through **Design Constraints** sections, and its **Risks and Open Questions** section. Alongside them, capture what the outline has no heading for:

- users and callers;
- affected systems;
- likely entrypoints;
- operational and runtime concerns.

Record an unknown as an open question. Plausible design is not a substitute for a fact.

Completion criterion: conversation, code, docs, or an explicit open question grounds every claimed requirement and constraint.

### 3. Explore design alternatives

Produce materially different alternatives before you choose the recommendation. Alternatives differ in interface shape, seam placement, ownership, call stack, runtime topology, or module boundaries. A different name is not a different alternative.

For each alternative, sketch the contracts listed in step 4, plus:

- the entrypoint-to-side-effect call stack;
- the parsing and projection strategy;
- the test seam strategy;
- the tradeoffs.

Compare the alternatives on:

- caller burden;
- module depth and leverage;
- locality of invariants and change;
- seam placement;
- boundary parsing and projections;
- error and cancellation model;
- testability through real seams;
- operational and runtime fit;
- implementation complexity.

Completion criterion: the recommendation follows the comparison. It does not precede it.

### 4. Specify the recommended typed contracts

For the recommended design, outline every new, changed, or deleted:

- domain value;
- branded or refined type;
- state machine variant;
- input and output type;
- request and response shape;
- function signature;
- class or module interface;
- expected-failure error type;
- Adapter interface;
- protocol boundary shape;
- persistence shape or projection;
- runtime-boundary codec;
- public API.

Name each seam, Adapter, implementation, and ownership boundary, and name what crosses it. State what each side may know, and what stays on its own side.

Name a boundary shape after its protocol or persistence meaning — `CreateUserRequest`, `StripeCustomerResponse`, `UserRecord`. `DTO` describes the role in prose; it does not appear in a symbol name.

Completion criterion: every new or changed boundary carries a concrete type, interface, or API sketch, or an explicit reason it needs no new contract.

### 5. Specify call stacks and data flow

For every new, changed, or deleted behavior, show the call stack from entrypoint to side effects and response.

Include the type and data flow:

```txt
raw input
  -> boundary shape / unknown
  -> parser
  -> canonical domain or application input
  -> Application Service interface
  -> outbound Adapter call
  -> typed result or error
  -> projection
  -> serialized output
```

When you change existing behavior, show the current flow beside the proposed one. Run `/call-stack <feature>` to trace the current flow from the real code rather than reconstructing it by hand.

Include the failure, retry, cancellation, transaction, idempotency, observability, authorization, and runtime-hop flows that the change reaches.

Completion criterion: every affected behavior carries an end-to-end call stack and a type and data-flow trace.

### 6. Map files and modules

List:

- files and modules to add;
- files and modules to change;
- files and modules to delete;
- test files;
- config, migration, and runtime files.

For each file, state what it owns: a contract, a code path, a boundary, an Adapter, a domain concept, or a test responsibility.

Give each module one role from `ts-standards`: Domain Module, Application Service Module, inbound Adapter, outbound Adapter, or composition root.

Completion criterion: every contract and every call-stack step maps to a file, a module, or an open question.

### 7. Write the RGR TDD test plan

`ts-standards/TESTING.md` owns the test tiers, the real-seam catalogue, the observable-behavior assertions, and the `fast-check` arbitraries. This step owns the slicing.

Plan vertical Red-Green-Refactor slices: one failing behavior test, the minimal implementation, repeat. A horizontal plan — all tests first, all code later — is not a slice plan.

Cover proportionately:

- happy paths;
- failure paths;
- parser rejection and accepted shapes;
- domain invariants and state transitions;
- Adapter contracts;
- persistence and runtime semantics;
- cancellation, retry, and idempotency paths;
- observability and safe summaries;
- end-to-end flows for high-consequence behavior.

Completion criterion: every public behavior, invariant, important failure path, changed boundary, and changed seam carries a red test slice, or an explicit reason to leave it untested.

### 8. Produce the blueprint

Follow the outline below. Return it inline, unless the user asked for a file path.

Return the blueprint and stop. Implementation is a separate request.

Completion criterion: another engineer can implement from the output without asking you a question that the codebase already answers.

## Path B: grill first

1. Hold the blueprint.
   - State that the context does not yet support an implementation-ready blueprint.
   - Completion criterion: no invented requirement, API, file, or call stack exists yet.
2. Run the grilling interview.
   - Ask one question at a time. Supply your recommended answer with each question.
   - When the codebase can answer a question, inspect the codebase instead of asking.
   - Completion criterion: the interview holds enough for Path A — problem, users and callers, constraints, affected systems, desired behavior, boundaries, likely APIs, invariants, risks, and acceptance tests.
3. Convert to the blueprint.
   - Run Path A.
   - Completion criterion: the artifact is a typed call-stack architecture handoff, not interview notes.

This interview is bounded: it ends when Path A can start. To stress-test whether the idea is worth building at all, use `/volley` first — that one is open-ended by design.

## Required blueprint outline

Use this shape, unless the task is small enough to compress without losing a contract or a call stack:

```md
# <Title>

## Summary

## Context / Current State

## Goals

## Non-Goals

## Invariants

## Design Constraints

## Alternatives Considered

### Option 1: <name>

### Option 2: <name>

### Option 3: <name>

## Recommendation

## Proposed Design

## Domain Model and Types

## Types, Interfaces, and APIs

## Seams, Boundaries, Adapters, and Implementations

## Call Stacks and Data Flow

### Current / Old Flow

### Proposed / New Flow

### Failure Flow

### Retry / Cancellation / Idempotency Flow

### Observability Flow

## Files to Add / Change / Delete

## RGR TDD Test Plan

## Risks and Open Questions
```

Omit a section that truly does not apply. Keep the typed contracts, the seams, the call stacks, and the tests even when they are hard to specify.

## Writing rules

- Code first. TypeScript pseudocode defines the contracts, the APIs, and the data flow.
- Prose explains why. Types and call stacks define what changes.
- Focus on types, interfaces, APIs, inputs and outputs, seams, boundaries, Domain Modules, Application Service Modules, inbound and outbound Adapters, and call stacks.
- `ts-standards` owns the design rules this blueprint applies — domain types over primitives, real seams, and abstractions that earn their existence. Apply them. Do not restate them in the blueprint.
- Keep one source of truth. Where two sections cover one rule, let one point at the other.
- An unknown stays an open question. A blueprint that feels complete because you invented a product requirement, a domain rule, an API, or a call stack is worse than one with open questions in it.
