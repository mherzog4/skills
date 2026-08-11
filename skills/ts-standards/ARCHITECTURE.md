# Module architecture

Read this before you create a module, add an Adapter, or move behavior across a boundary. [`SKILL.md`](SKILL.md) holds the role table and the dependency diagram.

## Applying the roles in any codebase

For a new feature or a local refactor:

1. Trace one caller-visible operation from ingress to every effect.
2. Put intrinsic meanings, invariants, calculations, and transitions in Domain Modules.
3. Put application policy and effect ordering in an Application Service. Define its dependencies as narrow ports.
4. Put each protocol or technology translation in an inbound or an outbound Adapter.
5. Wire concrete Adapters to ports at the composition root.
6. Verify each role through its public seam: domain results, application outcomes, and boundary records or responses.

Apply these responsibilities inside the project's existing layout and framework vocabulary. Migrate mixed code across the feature's required semantic surface only. Elsewhere, contain the old convention at an Adapter seam.

A password reset shows every role. `EmailAddress` and `ResetToken` are Domain Modules. `PasswordReset` is the Application Service. An HTTP route is an inbound Adapter. The Postgres store and the email provider are outbound Adapters. Bootstrap performs the wiring.

Build the roles the behavior needs. A pure operation may need only a Domain Module. A simple boundary may call an Application Service and introduce no new domain type.

## Deep modules

A deep module hides substantial behavior, invariants, policy, sequencing, or translation behind a cohesive, low-burden interface. Low burden does not mean few functions.

An abstraction earns its place when it does more than forward a call, mirror a table, rename another API, or expose implementation steps.

Use the deletion test:

- Deleting the module makes complexity disappear — it was pass-through waste.
- Deleting the module spreads complexity across callers — it earned its keep.

## Domain Modules

A **Domain Module** is a pure, type-centric abstract data type in the OCaml tradition. It centers one primary domain type, or a tightly related family, and owns what values mean and which operations are legal.

Create one when the code carries a meaningful domain distinction, invariant, calculation, decision, or lifecycle. Keep a primitive or a local pure function when a domain abstraction would prevent no realistic misuse and centralize no rule.

A Domain Module:

- co-locates its type, supporting types, parsers, smart constructors, combinators, predicates, legal transitions, domain projections, formatting, and test generators
- returns refined values from its parsers and constructors, so a caller cannot create an invalid instance
- expresses expected failures as precise values
- stays deterministic and independent of I/O, frameworks, persistence, ambient time, randomness, and mutable global state

It may define a pure permission decision over parsed domain values.

Authentication, authorization context, effect ordering, storage queries, network calls, and transport shapes all live outside it. A caller uses its operations rather than recreating its checks or branding a value with a cast.

```ts
// email-address.ts

/** A parsed, normalized email address. */
export type EmailAddress = Brand<string, "EmailAddress">;

/** Parse an email address from untrusted input. */
export function parse(input: string): Result<EmailAddress, InvalidEmailAddress>;

/** Render an email address as a string. */
export function toString(email: EmailAddress): string;

/** Compare two email addresses for equality. */
export function equals(left: EmailAddress, right: EmailAddress): boolean;
```

A Domain Module may use plain functions, immutable value classes, or static-style classes. A class in this role:

- constructs through `parse`, `make`, or another smart constructor
- makes an invalid instance unconstructable
- keeps fields readonly from the caller's view
- keeps methods cohesive over that one value
- holds no dependencies and no I/O
- uses composition rather than inheritance for its behavior

## Application Service Modules

An **Application Service Module** owns one cohesive application operation or capability — `PasswordReset`, `Invitations`, `SubscriptionLifecycle`. It applies application policy and sequences effects through narrow, application-owned ports, and it delegates intrinsic business rules to Domain Modules.

Create one when an operation coordinates authorization, domain decisions, persistence, external calls, transactions, messages, time, IDs, or telemetry. Create one when several entrypoints must call the same operation. A direct Domain Module call is enough when no application policy and no effect orchestration exist.

An Application Service:

- accepts and returns application and domain types, with precise expected-error unions
- defines the smallest meaningful ports the operation requires
- receives ports, configuration, clocks, and randomness explicitly
- owns which effects occur, under what policy, and in what order
- stays independent of HTTP, CLI, queue, ORM, vendor SDK, and runtime types

Protocol envelopes, response rendering, SQL, vendor translation, and domain invariants all live outside it.

Use constructor injection for a dependency-bearing class. Pass dependencies once at construction, not on every call.

Method count carries no limit. Split methods that represent unrelated capabilities, change for different reasons, or need unrelated dependencies. Name the service after the capability. `Manager`, `Processor`, `Helper`, and a generic `UserService` name nothing, unless the project already established them.

## Adapter Modules

An **Adapter Module** owns one boundary's translation and technology mechanics. Create one whenever application code crosses a framework, protocol, serialization, process, persistence, runtime, or third-party boundary.

An **inbound Adapter** parses an external request, event, or command, invokes an Application Service, and projects the result into the external protocol. Examples: an HTTP route, a GraphQL resolver, a CLI command, a queue consumer.

An inbound Adapter may call a Domain Module directly for a pure operation — one with no authorization, no application policy, no persistence, no external call, and no effect sequencing.

An **outbound Adapter** implements an Application Service port with a concrete technology. It translates raw records, SDK values, and external failures into application and domain types and typed errors. Examples: a Postgres store, a Stripe client, an email sender, a system clock.

An Adapter owns schema translation, framework lifecycle, external error classification, timeouts, and safe diagnostics for its boundary. Raw external types stay inside it or inside the composition root.

Business eligibility, authorization policy, legal state transitions, and application-operation ordering all live outside it. An entrypoint Adapter stays a thin protocol translation layer; business rules do not appear in a controller, a resolver, a command, or a handler.

An Adapter may retry a short-lived technical failure when the operation is safely repeatable and the retry does not change the port's meaning.

A port is not an Adapter. A port is the application-owned contract that states what an operation needs. An outbound Adapter is one replaceable implementation of it. An Adapter that forwards the same shape to another internal module, hiding no translation and no mechanics, is not worth adding.

## Authentication and authorization

Each role owns one part:

- An inbound Adapter verifies boundary credentials and produces a parsed identity — `Principal`, `Session`, `CommandActor`. It projects a missing credential, an invalid credential, and a denied operation into protocol-specific outcomes.
- A Domain Module defines pure permission decisions over parsed domain values.
- An Application Service gathers the required context and enforces those decisions while it carries out the operation.

Permission policy lives in the Domain Module and the Application Service, never in the Adapter.

## Composition root

The composition root parses environment and configuration, acquires resources, constructs concrete Adapters, and injects them into Application Services. Framework bindings and concrete wiring live here. Domain rules, application policy, and reusable boundary translation live elsewhere.

## Application-owned ports

Define a port beside the Application Service that needs it, in the application's language rather than the provider's. Depend on the smallest meaningful capability the operation uses, and let a cohesive concrete Adapter be wider. Port inputs, outputs, and errors are application and domain types.

TypeScript's structural typing makes this cheap:

```ts
type UsersForPasswordReset = {
  findActiveByEmail(email: EmailAddress): Promise<Result<ActiveUser, UserLookupError>>;
};

export class PasswordReset {
  constructor(private readonly users: UsersForPasswordReset) {}
}
```

A wider `PostgresUsers` class with `findActiveByEmail`, `findById`, and `updateProfile` satisfies that port without declaring it. This avoids both a mega-repository and one-method Adapter sprawl.

## Adapter reuse audit

Audit the existing Adapters and Application Services before you create one. Prefer, in order:

1. Reuse an existing Adapter as-is, through a narrow dependency type.
2. Extend an existing Adapter when the new method fits its cohesive capability and changes for the same reason.
3. Create a new Adapter when reuse or extension would create bad coupling or an accidental interface.

A routine feature-level Adapter or Application Service needs no ADR.

Write an ADR when the new module introduces a lasting architectural boundary, a shared pattern, a provider strategy, or a deliberate exception to these standards. The ADR states:

- which existing Adapters and Application Services you checked
- why reuse or extension did not fit
- why the new boundary is a separate cohesive capability

## Repositories and persistence

A repository-like Adapter represents a cohesive domain persistence capability, not one table. It exposes meaningful domain operations and returns parsed domain types and typed errors.

Treat a raw database row and an ORM model as infrastructure shapes. Parse them inside the Adapter. SQL and ORM details stay inside the persistence module.

## Workflows, transactions, and idempotency

Use an ordinary function call or a database transaction for a simple single-boundary operation.

Use a saga or a durable workflow when progress must survive process loss or redelivery, or when the operation needs long delays, compensation, resumability, timers, human approval, cross-service coordination, or several transaction boundaries. A short-lived retry alone does not need durable workflow machinery.

Retries have three owners:

- An Adapter owns a safe, short-lived technical retry.
- An Application Service decides whether to attempt the application operation again.
- A durable workflow owns a retry that must survive a crash, a delay, or a redelivery.

Close a database transaction before a network call or a long-running operation.

Every externally observable mutation that may be retried carries an explicit idempotency strategy:

- an idempotency key
- a natural unique constraint
- a deduplication record
- a state-machine transition guard
- a transactional outbox or inbox

State the strategy in the code. A side effect that is "probably safe" to repeat is not one of them.
