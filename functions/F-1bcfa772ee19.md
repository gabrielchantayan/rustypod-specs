<!-- Trusted provenance: F-1bcfa772ee19; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1bcfa772ee19

## Purpose

F-1bcfa772ee19 initializes context metadata, transforms two context-associated objects through dependency calls, and uses a derived 32-bit control value to drive a sequence of additional transformations.

The established behavior is a call-and-state protocol. The supplied body does not establish the mathematical meaning of the objects or of the dependency operations. In particular, describing the routine as a particular arithmetic algorithm would require additional evidence.

The routine has no explicit return value. Its observable results are writes to the context, changes made by dependencies through supplied addresses or handles, and its dependency-call sequence. Termination depends on both integer behavior and the effects of dependencies.

## Interface

### Arguments

The routine accepts two addresses:

- **Context address:** an address interpreted as an array of 32-bit words.
- **Operand address:** an address interpreted initially as an array of signed 32-bit words and subsequently passed to dependencies as an object address.

The directly accessed context layout is:

| Byte offset | Width | Use |
|---|---:|---|
| 0 | 32 bits | First object address or handle, retained for later calls |
| 4 | 32 bits | Second object address or handle, retained for later calls |
| 8 | 32 bits | Metadata written by this routine |

The directly accessed operand layout is:

| Byte offset | Width | Use |
|---|---:|---|
| 0 | 32 bits | Input to the first scale calculation |
| 4 | 32 bits | Input to the metadata calculation, including conditional rereads |

These layouts describe only direct accesses. Dependencies may require larger objects, different alignment guarantees, or additional accessible storage.

Two global 32-bit quantities participate in the initial calculations:

- A **scale coefficient**, multiplied by the operand’s first word.
- A **metadata coefficient**, multiplied by the operand’s second word.

Their values and mutability are not established.

### Local storage

The routine reserves a 524-byte local work area and passes its address to dependencies. It does not directly initialize that area before the first dependency receives it.

The area is reused throughout the invocation. Its address is valid only during the call. There is no visible allocation, explicit release, or transfer of ownership.

### Result and validity requirements

There is no explicit success code, error code, or result object returned by the routine itself.

At minimum, the direct accesses require:

- Readable first and second operand words.
- Readable first and second context words.
- Writable third context word.
- Objects or handles acceptable to every invoked dependency.

No direct null, alignment, length, handle-validity, or allocation-failure checks are present. Dependency-specific requirements are documented_only.

Aliasing is not prohibited by the visible interface. It can affect subsequent reads and calls; callers must not assume that the context, operand, or referenced objects are independent.

## Behavior

### 1. Derive the scale and retain context objects

Before any dependency call, the routine performs these operations in the following statement-level order:

1. Multiply the scale coefficient by the operand’s first signed 32-bit word.
2. If the resulting signed value is negative, negate it.
3. Multiply the resulting 32-bit value by 32 to obtain the **scale word**.
4. Read and retain the context word at byte offset 4 as the **second object**.
5. Multiply the metadata coefficient by the operand word at byte offset 4 to obtain the initial **metadata state**.
6. Read and retain the context word at byte offset 0 as the **first object**.

The retained object values are not subsequently reloaded from their context slots by this routine. Dependencies can modify those slots, but later direct calls still use the retained values. The context address itself is also supplied to one dependency later.

The scale word is retained independently of later changes to the operand or globals.

The signed multiplication and conditional negation have overflow-sensitive cases. A mathematically unbounded absolute-value interpretation is not justified for all inputs; see **Uncertainties**.

### 2. Construct and store metadata

A 32-bit accumulator starts at one. The routine performs exactly 31 rounds, using bit positions 1 through 31.

Each round:

- Logically shifts the metadata state right by one bit.
- Examines bit zero of the shifted state.
- If that bit is set:
  - Adds the power of two corresponding to the current bit position to the accumulator.
  - Rereads the metadata coefficient and the operand’s second word.
  - Replaces the metadata state with their product plus the shifted state.
- Otherwise, retains the shifted state without that reread.

Thus each branch decision uses the state **after** its right shift. It does not examine the pre-shift low bit.

The accumulator’s bit zero remains set. Each higher bit is added at most once. Under 32-bit bit-vector arithmetic, the stored metadata is the additive inverse of this accumulator, modulo \(2^{32}\).

The routine writes that value to the context word at byte offset 8. This is the only explicit caller-memory write visible outside dependencies.

No dependencies are called during the 31 rounds. Nevertheless, the conditional coefficient and operand reads remain observable accesses; their values are not replaced by a guaranteed immutable snapshot.

### 3. Initialize the first object and work area

After storing metadata, the routine calls, in order:

1. F-73c0d997b6c0 with the first object and the value one.
2. F-796113959089 with the first object and the scale word.
3. F-bcd78fb452c3 with the local work area and the operand address.

It then enters the first normalization phase, with the first object as the active object.

The body does not establish whether these operations initialize, assign, shift, resize, or otherwise transform their arguments. Their argument order and position in the protocol are established.

### 4. Normalization protocol

All normalization phases follow the same call-and-branch structure. Each phase has:

- An **active object**.
- The operand address as a persistent second argument.
- The local work area.
- A signed **direction value**, initially positive one.

The phase repeatedly calls F-08a778c50842 with the active object and operand address.

- A negative result ends that phase.
- A zero or positive result causes another call to F-08a778c50842, this time with the active object and local work area.

If the second result is negative:

- The direction becomes negative one.
- F-aa5c6722c7c4 is not called during that iteration.
- The routine calls F-08a778c50842 with the work area and operand address.
- Only if that additional result is strictly positive does it call F-796113959089 with the work area and negative one.

If the second result is zero or positive:

- The routine first calls F-aa5c6722c7c4 with the active object and work area.
- If direction is still positive one, it calls F-796113959089 with the work area and positive one, without an intervening work-area/operand comparison.
- If direction is negative one, it first calls F-08a778c50842 with the work area and operand address, and calls F-796113959089 with the work area and negative one only when that result is strictly positive.

Direction is sticky within a phase: once changed to negative one, it does not return to positive one. A fresh normalization phase resets it to positive one.

The next iteration always begins with a fresh active-object/operand call. No maximum iteration count or progress check is present.

These are branches on dependency return signs, not an established mathematical comparison contract. Even calls whose return values are ignored may have side effects.

### 5. Prepare the second object

When the first normalization phase ends, the routine:

1. Calls F-bcd78fb452c3 with the second object and first object.
2. Computes a signed power-of-two bound using the retained scale word.
3. Calls F-bcd78fb452c3 with the local work area and operand address.
4. Calls F-796113959089 with the second object and one.
5. Normalizes the second object using the protocol above.

The power-of-two computation starts at signed one and repeatedly doubles while the current signed value is less than or equal to the scale word interpreted as signed 32-bit.

For a nonnegative signed scale below \(2^{30}\), this yields the smallest power of two strictly greater than the scale. In particular:

- A scale of zero yields a bound of one.
- A scale of 32 yields a bound of 64.
- A scale exactly equal to a representable power of two causes another doubling.

For a negative signed interpretation of the scale word, the doubling loop performs no iterations and leaves the bound at one.

Larger nonnegative scales reach a signed-left-shift boundary; their behavior is not established as portable signed arithmetic.

This computation occurs after the first copy-like dependency call and before the next work-area call. If it fails to terminate, the latter call and subsequent operations do not occur.

### 6. Process the remaining control positions

After the second-object normalization ends, a control word is initialized to the bound divided by two.

The routine then repeats:

- If the control word interpreted as signed is less than two, return.
- Call F-dca73609ec09 with these four arguments, in order:
  1. Operand address.
  2. Second object.
  3. Context address.
  4. Second object again.
- Divide the signed interpretation of the control word by two.
- Test whether the resulting control word shares any set bit with the retained scale word.
- If no bit is shared, proceed directly to the next control-position iteration.
- If a bit is shared:
  1. Call F-796113959089 with the second object and one.
  2. Call F-bcd78fb452c3 with the work area and operand address.
  3. Normalize the second object with a newly initialized positive direction.
  4. Resume control-position processing with the already-reduced control word.

The duplicate second-object arguments to F-dca73609ec09 are intentional in the visible call. Its contract must accommodate that aliasing for this caller to be valid.

For positive, normally computed bounds, the control value decreases by halving. The scale’s highest processed bit is handled through second-object preparation rather than through this remaining-position loop. No equivalent mathematical interpretation is established beyond the actual calls and bit tests.

## Side effects

The routine directly writes only the metadata word at context byte offset 8. Dependency calls may mutate:

- The first object.
- The second object.
- The operand object.
- The context.
- The local work area.
- Global state.

No preservation guarantee for globals, operand contents, context slots, or referenced objects follows merely from a dependency’s position or apparent role.

There is no visible rollback. If execution stops, faults, or fails to terminate after metadata storage or object mutation, earlier effects remain observable unless a dependency independently reverses them.

There is no visible cleanup call before normal return. This does not establish that cleanup is unnecessary: resource ownership and any internally allocated contents of the work area are unknown.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-73c0d997b6c0 | Called once with the retained first object and one, after metadata storage |
| F-796113959089 | Called with an object or work-area address and a 32-bit quantity; quantities include the scale word, positive one, and negative one |
| F-bcd78fb452c3 | Called with destination-position and source-position arguments; exact copying or initialization semantics are unknown |
| F-08a778c50842 | Returns a signed result used through negative, nonnegative, and strictly-positive tests |
| F-aa5c6722c7c4 | Called with active object and work area on nonnegative active/work-area results |
| F-dca73609ec09 | Called during remaining-position processing, with the second object supplied twice |

No dependency’s implementation, allocation behavior, failure mechanism, or global-preservation properties are supplied.

## Acceptance examples

These are conditional static acceptance cases, not runtime test results.

### Metadata with an initially zero state

If the initial metadata product is zero, every shifted state remains zero and no conditional product reread occurs. The accumulator remains one, and the context metadata becomes `0xFFFFFFFF`.

This outcome is independent of later dependency behavior because the metadata write precedes all dependency calls.

### Metadata with only bit one initially set

If the initial metadata state is two, the first shifted state is one, so the first round adds two to the accumulator and performs the conditional product reread.

If that reread product is also two, the replacement state is three. The next shift again produces one. If the reread product remains two throughout, every round takes the branch, the accumulator becomes `0xFFFFFFFF`, and stored metadata becomes one.

### Immediate normalization exit

If the first active-object/operand call in a normalization phase returns a negative value, that phase performs no active/work-area call, no F-aa5c6722c7c4 call, and no work-area adjustment.

### Zero is not negative

If an active/work-area result is zero while direction is positive one, F-aa5c6722c7c4 is called, followed directly by the positive-one work-area adjustment.

If direction is already negative one, that same zero result still causes F-aa5c6722c7c4, but adjustment requires a subsequent strictly-positive work-area/operand result. A zero result there suppresses adjustment.

### Zero scale

Assuming the preceding dependencies and normalization phases terminate, a zero scale produces bound one and initial control zero. The routine returns after the second-object normalization without calling F-dca73609ec09.

### Scale 32

Assuming ordinary representable arithmetic and terminating normalization phases, scale 32 produces bound 64 and initial control 32.

F-dca73609ec09 is called at control values 32, 16, 8, 4, and 2. Each call is followed by halving before the bit test. None of the tested values 16, 8, 4, 2, and 1 intersects scale 32, so no intervening bit-triggered normalization occurs.

### Scale 96

Under the same conditions, scale 96 produces bound 128 and initial control 64. The first remaining-position call is followed by control 32. Because 96 contains bit 32, the positive-one second-object operation, work-area refresh, and fresh normalization occur before processing resumes.

## Uncertainties

- **documented_only — Dependency meaning:** Numerical comparison, copying, subtraction, scaling, squaring, and reduction are possible interpretations of portions of the protocol, but none is established by the supplied body alone.
- **documented_only — Overflow model:** Exact behavior of overflowing signed multiplication, negation of the minimum signed value, and signed left shifts requires target-level arithmetic evidence. Under wrapping 32-bit arithmetic, scale multiplication retains only the low 32 bits and therefore produces a word with five low zero bits.
- **documented_only — Large-scale termination:** With wrapping left shifts, a nonnegative signed scale at least \(2^{30}\) can drive the bound from \(2^{30}\) to a negative value and then zero, after which the loop remains at zero. This is a conditional consequence, not a verified target behavior.
- **documented_only — Dependency failures:** No explicit failure handling is visible. Negative F-08a778c50842 results are control-flow inputs, not established error indications.
- **documented_only — Ownership and initialization:** The validity of the uninitialized local work area as a dependency destination, and any cleanup obligations for its contents, require dependency contracts.
- **documented_only — Concurrent mutation:** No synchronization or atomic-access guarantee is established.
- **Inferred intent:** The scale-driven traversal and repeated normalization suggest a structured arithmetic transformation. That interpretation does not replace the observable contract.

## Evidence

The supplied static body establishes the argument accesses, 32-bit fields, 524-byte local area, 31-round metadata construction, metadata write, dependency argument order, normalization branches, signed bound computation, control-word halving, and explicit return condition described above.

The dependency mapping establishes only opaque identities, not implementation semantics.

No runtime tests have been performed. All acceptance examples are conditional deductions from the supplied static evidence.
