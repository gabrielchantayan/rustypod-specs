<!-- Trusted provenance: F-7ab695ebfcd6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7ab695ebfcd6

## Purpose

F-7ab695ebfcd6 conditionally updates one of two selector fields in an object obtained from F-7fba0909c452. Two request identifiers select the respective fields. Either field accepts exactly one of four integer values: 1, 2, 4, or 8.

Requests that do not produce a selector update enter a separate status-recording path. An unrecognized request identifier supplies a fixed candidate status of 1280. A recognized request identifier with a disallowed value instead uses a captured global value. A nonzero candidate is written to the object's first word only when that word is zero.

A null object result prevents all object accesses.

These are descriptions of the visible behavior, not claims about the surrounding API's intended meaning. “Selector” and “status” are descriptive labels; the supplied material does not establish their application-level semantics.

## Interface

### Inputs

The function receives two integer arguments:

| Argument | Role | Relevant values |
|---|---|---|
| Request identifier | Selects a destination field or the fixed-status path | 3317 (`0xCF5`) and 3333 (`0xD05`) are recognized |
| Requested value | Value to store for a recognized request | Exactly 1, 2, 4, or 8 is accepted |

The supplied representation uses integer arguments and integer object words. This contract describes them as 32-bit quantities, consistent with that representation. It does not establish the complete machine calling convention or independently prove the original language's integer ABI.

All tested constants fit in signed 32-bit integers. Selection uses exact equality, not ordering or bitwise containment. No addition, multiplication, shifting, or narrowing of argument values is visible.

### Output

The function has no defined return-value payload. Results are communicated, when applicable, through object-memory writes.

A normal return alone does not distinguish:

- A successful selector update.
- A null object result.
- A rejected value.
- A zero candidate status.
- A candidate status blocked by an existing nonzero status.

### Object layout

F-7fba0909c452 supplies the object pointer. The visible function accesses only these locations:

| Byte offset from object base | Width | Role |
|---|---:|---|
| `0x000` | 32 bits | Existing status; conditionally receives a candidate status |
| `0x90C` | 32 bits | Selector for request 3333 |
| `0x910` | 32 bits | Selector for request 3317 |

The selector offsets follow from word indices 579 and 580 at four bytes per word. They are not adjacent to the first status word.

A successful update at offset `0x910` requires accessible storage through byte offset `0x913`, inclusive. An update at offset `0x90C` requires accessible storage through byte offset `0x90F`, inclusive. The function performs no visible bounds or layout validation.

Byte order, alignment requirements, and the layout of all intervening storage are unspecified.

### Preconditions and ownership

For a defined execution, every memory location actually accessed must be valid for its operation. In particular:

- A nonnull returned pointer must support the reads and writes selected by the request.
- The global status source must be readable after the dependency returns, including when the returned object pointer is null.
- The object must remain valid for the duration of its accesses.
- Any required synchronization must be supplied by the surrounding environment.

The function does not visibly allocate, release, retain, or transfer ownership of the object. Nothing here establishes whether the dependency returns borrowed, shared, or newly allocated storage.

## Behavior

### Acquisition and snapshot order

The visible sequence begins by calling F-7fba0909c452 with no explicit arguments.

After that call returns, the function reads one global integer value and captures it as the initial candidate status. Only then does it test the returned object pointer for null.

Consequently, in the supplied representation:

1. The dependency executes before the global value is captured.
2. The global read occurs even if the object pointer is null.
3. Dependency side effects may affect the captured global value.
4. There is no visible second read of that global value.

If the object pointer is null, the function returns without reading or writing any object field.

The statements above describe the visible abstract access order. They do not establish volatile semantics or exact emitted machine-load behavior.

### Recognized request 3317

When the request identifier equals 3317, the requested value is compared against the set `{1, 2, 4, 8}`.

If it belongs to that set:

- The requested value is written as one 32-bit word at object offset `0x910`.
- The function returns.
- The first status word is neither read nor written by this function.
- The captured global status does not affect the update.

There is no prior read of the selector field and no visible comparison against its existing value. A store is specified even when the field already contains the requested value.

If the value is outside the accepted set, neither selector field is written. Processing continues to the common status-recording path using the captured global value.

### Recognized request 3333

When the request identifier equals 3333, the same four values are accepted.

For an accepted value:

- The requested value is written as one 32-bit word at object offset `0x90C`.
- The function returns without accessing the first status word.
- The selector at offset `0x910` is not accessed by this function.

For a disallowed value, neither selector field is written. Processing continues to the common status-recording path using the captured global value.

### Any other request identifier

For every request identifier other than 3317 and 3333:

- No selector field is written.
- The candidate status becomes 1280 (`0x500`), replacing the captured value for this decision.
- The requested value has no effect on the remaining behavior.
- Processing continues to the common status-recording path.

The earlier global read still belongs to the visible sequence, even though this branch does not use its captured value.

### Common status-recording path

The common path uses the candidate selected above.

If the candidate is zero, the function returns without reading or writing the object's first word.

If the candidate is nonzero, the function reads the 32-bit word at object offset zero:

- When the existing word is zero, the function writes the candidate to that word.
- When the existing word is nonzero, the function leaves it unchanged.

The function then returns. No selector update follows this path.

This is a conditional first-status recording rule for this invocation, not an unconditional error assignment. The nonzero test does not require a positive value: a negative captured status also qualifies for storage.

No visible atomic compare-and-exchange or locking protects the read followed by the conditional write. The function alone therefore does not establish a concurrent first-writer-wins guarantee.

## Side effects

### Direct object effects

For a nonnull object, the function directly performs at most one 32-bit object write:

- An accepted selector request writes its selected field.
- Otherwise, a nonzero candidate may be written to the first word if that word was read as zero.

These outcomes are mutually exclusive within the function's visible control flow.

A successful selector update does not clear or replace an existing status. An existing nonzero status does not prevent a successful selector update, because that path does not inspect the status word.

A rejected value does not reset a selector, substitute a default, or combine the requested value with existing selector bits.

### Reads and external effects

The direct reads are:

- One global status capture after the dependency call.
- A conditional first-word object read on the common path, only when its candidate is nonzero.

There is no visible direct global write.

The dependency can have additional effects not described by this function body. It must not be assumed to preserve globals, avoid allocation, leave the eventual object unchanged, or return the same object across calls.

No logging, exception handling, callback invocation, or explicit resource cleanup is visible in F-7ab695ebfcd6 itself. Fault behavior for invalid storage is not specified.

## Dependencies

### F-7fba0909c452

**Visible call contract:** Invoked once, before the global capture, with no explicit arguments. Its result is interpreted as a pointer to the object described above.

**Visible result handling:** A null result causes an early return after the global capture. A nonnull result is used without further pointer validation.

**Unknowns:** Object provenance, lifetime, ownership, internal state changes, global effects, concurrency properties, and failure mechanisms are not supplied.

### Global status source

One unnamed global integer supplies the candidate status for recognized requests with disallowed values.

Its value is captured after F-7fba0909c452 returns. This ordering matters if that dependency changes the global.

The global's writer, initialization, lifetime, thread-local or shared nature, and semantic meaning are unknown. Its resemblance to an ambient error source is inferred intent, not an established dependency contract.

## Acceptance examples

These examples are conditional static expectations, not observed test results. Each assumes the dependency returns normally and that all memory actually accessed is valid. “Preserved” refers only to direct effects of F-7ab695ebfcd6 after the dependency returns.

| Conditions after acquisition | Expected direct behavior |
|---|---|
| Dependency returns null | Capture the global, then return; no object access |
| Request 3317, value 1 | Write 1 at `0x910`; do not access the first status word |
| Request 3333, value 8 | Write 8 at `0x90C`; do not access the first status word |
| Request 3317, value 4, first status word already 77 | Write 4 at `0x910`; preserve status 77 |
| Request 3333, value 2, selected field already 2 | Perform the selector write of 2; no equality-based skip is visible |
| Request 3317, value 3, captured global 19, first status word zero | Leave selectors unchanged; write 19 to the first word |
| Request 3333, value zero, captured global zero | Leave selectors unchanged; return without reading the first word |
| Request 3333, value 16, captured global 19, first status word 7 | Read the first word and preserve 7; no object write |
| Unrecognized request, value 1, captured global zero, first status word zero | Write 1280 to the first word; do not update a selector |
| Unrecognized request, any value, first status word −1 | Read the first word and preserve −1 |
| Request 3317, value −1, captured global −9, first status word zero | Write −9 to the first word; leave selectors unchanged |

Additional boundary expectations:

- Values 3, 5, 6, and 7 are rejected for both recognized requests, even though they combine accepted single-bit values.
- Values immediately adjacent to accepted values are accepted only if independently in the set. For example, 2 is accepted, but 3 is not.
- A nonzero captured global does not divert an otherwise accepted selector request into status recording.
- If the dependency changes the global from zero to 23 before returning, a subsequent recognized request with a disallowed value uses the captured 23, not the earlier zero.
- A preexisting nonzero first word is never cleared by a direct action of this function.

## Uncertainties

### Established static evidence

The supplied representation establishes the two request constants, the exact accepted-value set, the selected word indices, the fixed status 1280, the conditional status-write rule, and the visible dependency/global/object access ordering.

It also establishes that no arithmetic is performed on the requested value before a selector store.

### Inferred intent

The selector values resemble mutually exclusive flags or modes. The first-word update resembles preserving the earliest recorded error. Neither interpretation establishes names, units, protocol meanings, or a broader state machine.

The contract does not require those interpretations.

### Documented-only unknowns

The following remain documented_only:

- Original declarations and complete ABI details beyond the supplied integer-word model.
- Endianness and alignment constraints.
- The object's complete size, type, and field meanings.
- The global status source's semantics and synchronization.
- Whether the global and object storage can alias.
- Whether external agents can change the object between a read and write.
- Dependency ownership, lifetime, and side-effect guarantees.
- Runtime behavior for invalid pointers, inaccessible global storage, or dependency failure.
- Compiler-level treatment of ordinary versus volatile accesses.

No unknown callee behavior is assumed to preserve globals or manage memory in a particular way.

## Evidence

This contract is derived solely from the supplied static function representation and the supplied opaque dependency mapping.

The byte offsets are calculated from four-byte word indexing: index 579 gives `0x90C`, and index 580 gives `0x910`. The decimal constants 3317, 3333, and 1280 correspond to `0xCF5`, `0xD05`, and `0x500`.

No implementation was executed, no runtime tests were performed, and no dependency body or surrounding object definition was available. The acceptance examples specify conditional consequences of the visible branches; they are not empirical verification.
