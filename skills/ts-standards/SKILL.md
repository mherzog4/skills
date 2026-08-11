---
name: ts-standards
description: Correct-by-construction TypeScript standards — errors as values, parse don't validate, deep modules, real test seams. Use when writing or refactoring TypeScript, or when another skill needs these coding standards.
---

These standards describe how to design and write TypeScript. Apply them to all new code and to the full behavior you refactor. Inspect existing code before you add a pattern, a library, an Adapter, or an abstraction. Follow an existing convention only when it is compatible with these standards.

Read [`ARCHITECTURE.md`](ARCHITECTURE.md) before you create a module, add an Adapter, or move behavior across a boundary.
Read [`TESTING.md`](TESTING.md) before you write or change a test.

For a change large enough to need typed contracts and call stacks agreed before the code exists, run `/blueprint` first. It produces the design; this document governs the writing.

## Decision priority

When rules pull in different directions, use this order:

1. Preserve correctness, safety, and debuggability.
2. Apply these standards to all new code and to the full behavior being refactored.
3. Follow compatible project architecture and conventions.
4. Contain an incompatible existing pattern at the nearest boundary. Do not copy it into new code.
5. Leave unrelated old code unchanged unless the user asks for a broader migration.

For example, in a codebase that throws exceptions, keep the exception style at the framework edge. Represent known failures as typed values in the code you write, then translate them at that edge into the outcome the framework requires. Preserve existing logging, tracing, metrics, and error-reporting hooks.

## Core principles

- **Errors as values** — an expected failure appears in the return type, not in a `throw`.
- **Parse, don't validate** — parse at the boundary and keep what the parse learned.
- **Illegal states unrepresentable** — model a lifecycle as a tagged union.
- **Correct by construction** — an API that cannot be misused beats a documented convention.
- **Functional core, imperative shell** — pure domain, effects at the edges.
- **Deep modules** — hide substantial behavior behind a low-burden interface.
- **Real seams** — inject dependencies in tests. Never mock a module.

## Errors and failures

### Expected failures are values

Every known failure mode appears in the return type as a custom tagged error, even when the immediate caller cannot recover. A caller handles the error or returns it upward. At the outermost boundary, translate it into a valid outcome: an HTTP response, a CLI exit code, a retry decision, a dead letter, or a startup error message.

Known failures include domain, parsing, authorization, integration, I/O, persistence, configuration, and workflow failures.

Preferred order:

1. `better-result`, when available and appropriate.
2. A small local tagged union:

```ts
type Result<T, E extends Error> =
  | { readonly _tag: "ok"; readonly value: T }
  | { readonly _tag: "err"; readonly error: E };
```

Write:

```ts
Promise<Result<User, UserLookupError>>
```

not:

```ts
Promise<User> // rejects for ordinary lookup/storage failures
```

A rejected promise is a `throw`. Catch an unclassified third-party rejection inside the owning Adapter and translate it into a known tagged error before it crosses the Adapter boundary. A rejection escapes application code only for a defect.

### Defects throw or panic

Throw or panic only when a defect makes correct execution impossible. A caller with no recovery strategy is not a defect. Defects include:

- a violated internal invariant
- an impossible branch
- a temporary `notYetImplemented` path
- a catastrophic runtime condition

A known configuration failure is a value. The composition root reports it safely and terminates startup.

Use the project's shared defect helpers, or the panic helpers from its result library:

```ts
export function casesHandled(unexpectedCase: never): never;
export function shouldNeverHappen(msg?: string): never;
export function notYetImplemented(msg?: string): never;
```

Use `casesHandled` to close an exhaustive union. When the project has these helpers, use them instead of `absurd` or a one-off `assertNever`.

### Custom errors

An expected failure uses a custom tagged error. Extend `Error`, or `TaggedError` from `better-result`.

Use `_tag` as the discriminant on every tagged error and every tagged union.

A custom error carries:

- a stable tag, written with `as const`
- a useful message
- structured contextual fields
- safe telemetry fields
- an optional `cause: unknown`

```ts
export class UserStoreUnavailable extends Error {
  readonly _tag = "UserStoreUnavailable" as const;

  constructor(
    readonly operation: "findActiveByEmail",
    readonly provider: "postgres",
    readonly cause: unknown,
  ) {
    super(`User store unavailable during ${operation}`);
  }
}
```

Keep an error union precise at a module boundary:

```ts
Result<User, UserNotFound | UserStoreUnavailable>
```

Reserve a broad `AppError`-style type for entrypoints, orchestration, logging, and rendering.

### Combining results

Collect many results with an `all` or `traverse` combinator. Choose fail-fast or accumulate-all deliberately, and let the error type name the choice.

`Promise.all` rejects on the first rejection. It does not combine error values, so it does not replace a result combinator.

### External calls

Every outbound network call takes a timeout and accepts an `AbortSignal`. A timeout produces a tagged error, not a rejection.

## Sensitive data and telemetry

Emit structured traces across requests, jobs, workflows, application modules, Adapters, and external calls. A trace makes a failure diagnosable through safe fields: domain IDs, operation names, provider names, state tags, retry counts, typed error tags, and safe summaries.

Keep secrets out of errors, traces, logs, and snapshots.

Wrap a sensitive value in `Redacted<T>` — a token, an API key, a password, a raw credential, or a secret. Use the repo's existing wrapper, or add one shared `Redacted<T>` module. Wrap the value at the boundary. Unwrap it only where the raw value goes out, usually inside the Adapter that makes the external call.

## Parse, don't validate

Boundary code turns unknown or less-structured input into application or domain types before that input enters inner code.

Add a separate protocol projection only when its shape or meaning differs enough to earn one. `DTO` describes a boundary role in prose. Name the symbol after its real protocol or persistence meaning — `CreateUserRequest`, `StripeCustomerResponse`, `UserRecord` — and keep `DTO` and `Dto` out of symbol names:

```ts
unknown -> CreateUserRequest -> CreateUserInput -> EmailAddress/UserId/etc.
```

Otherwise parse straight into the application input:

```ts
unknown -> CreateUserInput
```

Keep a schema-inferred transport shape at the boundary. It does not travel through the application:

```ts
unknown -> z.infer<typeof CreateUserSchema> // stops at the Adapter
```

Use names that preserve meaning:

- `parseX(input): Result<X, ParseXError>` for untrusted or less-structured input
- `makeX(...)` / `createX(...)` for a smart constructor over already-typed pieces
- `isX(value): value is X` for a true predicate
- `assertX(...)` rarely, at a test or framework boundary

When a function returns a refined value, name it `parseX`. It parsed something.

### Schemas

A schema library is a boundary parser. Keep it out of core logic.

Preference:

1. the repo's established schema library
2. Zod 4
3. a hand-written smart constructor for a small domain type, when that reads clearer

Prefer Standard Schema compatibility in a generic helper. Schema parsing produces refined domain types and typed custom errors.

## Branded types and correct construction

Use a branded or refined type when it prevents a realistic mistake:

- IDs: `UserId`, `OrgId`, `WorkflowId`
- parsed strings: `EmailAddress`, `NonEmptyString`, `Url`
- constrained numbers: `PositiveInt`, `Cents`, `Percentage`
- units and time: `Milliseconds`, `Bytes`, `UsdCents`, `Instant`

Construct a branded value through a parser or a smart constructor. Where a domain type exists, pass that type rather than a raw string or number.

Represent a domain timestamp as `Instant`. Keep raw `Date` inside Adapters.

A function that requires a value takes that value. Push optionality outward, then branch or parse before the call.

Use `Partial<T>` as an application input only when partiality is the real domain concept. Otherwise write an explicit input type for each operation.

## State machines and boolean blindness

Model a meaningful lifecycle as a tagged union:

```ts
type Invoice =
  | { readonly _tag: "Draft"; readonly id: InvoiceId; readonly lines: NonEmptyArray<LineItem> }
  | { readonly _tag: "Sent"; readonly id: InvoiceId; readonly sentAt: Instant }
  | { readonly _tag: "Paid"; readonly id: InvoiceId; readonly paidAt: Instant };
```

not as a bag of flags:

```ts
type Invoice = {
  readonly isSent: boolean;
  readonly isPaid: boolean;
  readonly sentAt?: Date;
  readonly paidAt?: Date;
};
```

Control behavior through a named option or a domain type:

```ts
createUser(input, { emailVerification: "skip" }); // not createUser(input, true)
```

A boolean is the right return type for a predicate:

```ts
isExpired(token): boolean;
hasPermission(user, permission): boolean;
```

## Modules

**Domain Module**, **Application Service Module**, and **Adapter Module** name responsibilities, not folders, suffixes, or TypeScript constructs. A module may be a function, an object, a class, a file, or a package with a cohesive public interface. Use the roles at any scale. Build three layers only when the behavior needs three.

```txt
external input -> inbound Adapter -> Application Service -> Domain Module
                                           |
                                           +-> application-owned port
                                                 -> outbound Adapter -> external system
```

Dependencies point inward. A Domain Module knows neither services nor Adapters. An Application Service knows application-owned port contracts, not concrete technologies. An Adapter depends on those contracts and translates at the edge. The composition root constructs concrete Adapters and supplies them to Application Services.

Classify code by the responsibility that would make it change:

| A change to… | Belongs in |
|---|---|
| business meaning, invariant, calculation, legal transition | Domain Module |
| application policy, authorization, effect sequence | Application Service Module |
| protocol, framework, database, runtime, third-party API | Adapter Module |
| construction, configuration, resource wiring | composition root |

Split an abstraction when it owns more than one of these reasons to change.

[`ARCHITECTURE.md`](ARCHITECTURE.md) holds the full treatment: role definitions, deep modules, ports, the Adapter reuse audit, persistence, authorization, workflows, and idempotency.

## Testing

Add an end-to-end test whenever the real public entrypoint can exercise the behavior in the normal test environment. Add lower-level tests where they cover important cases.

Use real seams. Never use `vi.mock` or `jest.mock`.

[`TESTING.md`](TESTING.md) holds the test tiers, the seam catalogue, observable-behavior assertions, and `fast-check` arbitraries.

## TypeScript style and safety

Enable these compiler options:

- `strict: true`
- `noUncheckedIndexedAccess: true`
- `exactOptionalPropertyTypes: true`
- `noImplicitOverride: true`
- `noFallthroughCasesInSwitch: true`

Declare values immutable:

```ts
type CreateUserInput = {
  readonly email: EmailAddress;
  readonly roles: ReadonlyArray<Role>;
};
```

Mutation belongs inside localized imperative shell code, a performance-sensitive internal, a builder, or an Adapter behind a precise interface.

### Casts, `any`, and non-null assertions

Biome bans `any` and `!`. An ast-grep rule bans a non-`as const` assertion. Branch, parse, or refine instead.

Use `satisfies` to check a literal against a type without a cast and without widening. `as const` is fine.

A rare exception applies to a highly generic helper, a branding internal, an interop boundary, or a combinator that TypeScript cannot express. Every non-`as const` cast carries a Rust-style safety comment on the line above:

```ts
// SAFETY: TypeScript cannot express the brand. parseEmailAddress checked the normalized string before branding. Callers cannot construct EmailAddress except through this parser.
return normalized as EmailAddress;
```

A rare `any` carries a targeted Biome ignore with the same justification:

```ts
// biome-ignore lint/suspicious/noExplicitAny: SAFETY: this helper preserves arbitrary function parameters; TypeScript cannot express this variadic constraint without any.
type Fn = (...args: any[]) => unknown;
```

## Imports, exports, and files

Import directly from the file that owns the abstraction. Biome's `noBarrelFile` and `noReExportAll` enforce this.

A namespace import preserves a domain module's shape:

```ts
import * as EmailAddress from "./email-address";

EmailAddress.parse(input);
```

Use a named import for a class or a focused shared helper:

```ts
import { PasswordReset } from "./password-reset";
```

Use `import type` and `export type` for type-only imports and exports.

Export what callers use. Keep an internal helper unexported. Tests reach behavior through the public interface, so a test is not a reason to export.

Name a file after what it owns — `email-address.ts`, `billing-period.ts`, `string-case.ts`. Keep `utils.ts`, `helpers.ts`, `common.ts`, and `misc.ts` out of the tree.

One explicit module may hold the tiny ubiquitous generics when no more precise owner exists:

- `casesHandled`, `shouldNeverHappen`, `notYetImplemented`
- `Redacted`
- `Tags`, `ExtractTag`, `ExcludeTag`
- common `Result` helpers when the project does not use `better-result`
- broad type utilities

Keep domain and application policy with their owning modules.

File size carries no limit. Prefer cohesion and discoverability. Split a file when it has unrelated reasons to change, or when a caller must understand unrelated concepts to read it.

## Comments and JSDoc

A comment explains an invariant, a trade-off, a non-obvious domain rule, or a safety justification. Code that reads clearly needs no narration.

Every exported symbol carries JSDoc. Every public method and property of an exported class carries JSDoc. Internal code carries documentation when its complexity earns it. Documentation lives on the original declaration; a re-export does not repeat it.

Write the documentation explicitly on each symbol. `@inheritDoc` and similar inheritance tags do not appear.

```ts
/**
 * Parse an email address from untrusted input.
 *
 * @param input - The untrusted string to parse.
 * @returns A parsed email address, or `InvalidEmailAddress` when the input is invalid.
 */
export function parse(input: string): Result<EmailAddress, InvalidEmailAddress>;
```

Use `@throws` for an unrecoverable defect, framework-required behavior, or a temporary `notYetImplemented` path. An expected typed error appears in the return type, not in `@throws`.

Document the fields of a complex exported object type:

```ts
/** Input required to create a user. */
export type CreateUserInput = {
  /** The actor creating the user. */
  readonly actor: AdminUser;

  /** The parsed email address for the new user. */
  readonly email: EmailAddress;
};
```

## Configuration and resources

Parse environment and configuration at startup into a typed config with branded and redacted values. Return a known configuration failure as a tagged error value. The composition root prints a safe startup message and terminates.

Read `process.env` in the config module only. Everywhere else, take the parsed config as a dependency.

Keep top-level side effects in true entrypoint and bootstrap files. A module starts no server, opens no connection, reads no environment, registers no handler, and performs no I/O at import time.

Bootstrap or imperative shell code owns resource creation and cleanup, explicitly.

Constants and pure lookup tables are fine. Keep mutable global state and mutable singletons out. When a framework requires a singleton, isolate it at the boundary.

Inject `Clock` and `Random` into a dependency-bearing module. A pure domain function takes an explicit `now` or an explicit random value.

## Tooling

The repo's config is the source of truth for every mechanical rule. This document owns the rules a tool cannot check.

- **Biome** formats, and enforces `noExplicitAny`, `noNonNullAssertion`, `useImportType`, `noNamespace`, `noBarrelFile`, and `noReExportAll`.
- **ast-grep** enforces the rules Biome cannot express: the non-`as const` assertion ban, JSDoc on exported symbols, `_tag` as the discriminant, `DTO` out of symbol names, raw `Date` out of the domain, `process.env` outside the config module, and `vi.mock` / `jest.mock` anywhere.
- **tsc** enforces the compiler options above.

Suppress a rule with a targeted ignore comment that carries a justification. A blanket disable does not appear.

## Before you report the work as done

Check each item against the diff. Do not check from memory.

1. The project's Biome check passes.
2. The project's ast-grep scan passes.
3. The project's typecheck passes.
4. Every expected failure you added appears in a return type.
5. Every non-`as const` cast you wrote carries a SAFETY comment.
6. Every symbol you exported carries JSDoc.
7. Every module you added has one role: Domain, Application Service, Adapter, or composition root.
8. Every test you wrote runs through a real seam.

Name any item you could not complete, and say why.
