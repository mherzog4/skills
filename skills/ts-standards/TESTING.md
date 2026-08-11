# Testing

Read this before you write or change a test. [`SKILL.md`](SKILL.md) holds the one-line rule.

## Which test to write

Add an end-to-end test whenever the real public entrypoint can exercise the behavior in the normal test environment, without an unreliable third party and without unreasonable setup, runtime, or cost. Add lower-level tests where they cover important cases the end-to-end test cannot reach.

Prefer tests in this order:

1. end-to-end through a real public entrypoint
2. integration through a real seam
3. focused and property tests over pure Domain Modules
4. unit tests, when they cover meaningful behavior

A test that asserts how the code is written, rather than what it does, breaks on every refactor and catches no defect.

## Real seams

Never use `vi.mock` or `jest.mock`. An ast-grep rule enforces this.

Substitute a dependency through a seam the production code already has:

- a constructor-injected interface or class
- a local database substitute such as SQLite
- an in-memory Adapter, when the behavior is simple
- a fake external Adapter that records what it received

Prefer a SQLite or local database test over a hand-rolled in-memory fake when SQL, schema, or transaction behavior matters to the case.

## Assert observable behavior

Assert what a caller or an operator can observe:

- the returned value or error
- the persisted state
- the emitted event or message
- the rendered response
- the record a fake Adapter captured, such as a sent email

```ts
const sent = await emails.sentTo(user.email); // a fake Adapter's record
expect(sent).toHaveLength(1);
```

Assert a call spy only when the interaction itself is the only observable behavior:

```ts
expect(sendEmail).toHaveBeenCalledWith(...); // last resort
```

Reach behavior through the public interface. A test does not bypass a parser, a smart constructor, or an invariant, and it is not a reason to export an internal helper.

## Property tests and arbitraries

Use `fast-check` where a property reads clearer than a set of examples:

- parsers and smart constructors
- branded and refined types
- state machines
- serialization roundtrips
- normalization and idempotence
- lawful combinators

```ts
it("parses every rendered address", () => {
  fc.assert(
    fc.property(emailAddressArbitrary, (email) =>
      EmailAddress.parse(EmailAddress.toString(email))._tag === "ok",
    ),
  );
});
```

Generate test data from arbitraries. Export each arbitrary beside the domain module it supports:

```txt
src/billing/
  invoice-number.ts
  invoice-number.test.ts
  invoice-number.arbitrary.ts
```
