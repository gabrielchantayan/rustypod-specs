<!-- Trusted provenance: F-49cfbfcd90ad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-49cfbfcd90ad

## Purpose

F-49cfbfcd90ad selects and returns a value through two ordered, zero-result gates followed by a field-based fallback.

The first gate depends on a value obtained without using the function’s input. If that gate produces zero, the function immediately returns a designated global value. Otherwise, the second gate obtains a value using an address derived from the input. If the second gate produces zero, the function returns a different global value. Only when both gates produce nonzero results does the function consult a field at input offset 44 and potentially return a third global value.

**Established behavior:** the gates test dependency results against zero. The supplied material does not establish that either gate performs string comparison, equality comparison, validation, or lookup.

**Inferred intent:** the function appears to select a preferred value, possibly a history-related value, with special-case overrides and a default. This interpretation is not part of the required behavior.

## Interface

### Input

The function has one input, referred to here as the **base**. Its declared type in the supplied representation is an integer, but the function uses it to form addresses.

Two offsets are relevant:

- **Offset 40 bytes:** the address supplied to F-44362605ab79.
- **Offset 44 bytes:** the location read directly during the final fallback stage.

The function passes the address at offset 40, not a value directly loaded from that address by this function. Whether F-44362605ab79 treats that address as an embedded object, a field, a string, or another representation is unknown.

The input does not need to be dereferenced on the first early-return path. Requirements for its validity are therefore path-dependent.

### Output

The function returns one integer-shaped value selected from:

1. A first global return slot.
2. A second global return slot.
3. The first direct read of the field at offset 44.
4. A default global return slot.

No conversion, allocation, copying of an object, or ownership transfer is explicitly shown. The returned bits could represent an integer, an address, a handle, or another encoded value; their semantic type is not established.

### Widths and arithmetic

The supplied representation explicitly identifies the results of F-3ad65987df9a and F-44362605ab79 as four-byte quantities. Those quantities are passed to the gate dependency.

The base, gate results, offset-44 field reads, and return value are represented using an integer type. A 32-bit integer/address model is consistent with the representation, but the supplied material does not independently establish the target ABI. Under that model:

- Each offset-44 read occupies four bytes.
- Returned values occupy 32 bits.
- The address expressions add 40 or 44 to the input’s numeric value.

There is no explicit overflow check, range check, alignment check, or address-validity check. Behavior when address formation overflows, or when the resulting address is invalid or unsuitable for the target’s access rules, is not established. The contract does not replace these cases with a defined error return.

### Data labels

The following document-local labels distinguish otherwise unidentified data:

| Label | Role |
|---|---|
| D-A | Fixed data operand supplied to the first gate |
| D-B | Global return slot used when the first gate returns zero |
| D-C | Global operand supplied to the second gate |
| D-D | Global return slot used when the second gate returns zero |
| D-E | Global default return slot used in the final fallback stage |

These labels identify roles, not contents. No value, string encoding, mutability, ownership rule, or semantic relationship among these data items is established.

## Behavior

### First gate

F-49cfbfcd90ad first calls F-3ad65987df9a with no explicit arguments.

After that call returns, it invokes the gate dependency with:

- The four-byte result of F-3ad65987df9a.
- The fixed data operand D-A.

If the gate returns zero, F-49cfbfcd90ad reads and returns D-B.

This path ends the function. It does not call F-44362605ab79, perform the second gate invocation, or directly read the field at offset 44.

D-B is not shown as being captured before the dependency calls. Consequently, if either preceding dependency changes D-B through an external mechanism, the returned value is the value available when the return slot is read, not necessarily its value at function entry.

A nonzero gate result continues execution. Its magnitude and sign do not otherwise affect the shown control flow.

### Second gate

The function forms the address at base plus 40 bytes and passes that address to F-44362605ab79.

After that call returns, the function invokes the same gate dependency with:

- The four-byte result of F-44362605ab79.
- The value obtained from D-C.

If this gate returns zero, the function reads and returns D-D.

This path does not directly read the field at offset 44. F-44362605ab79 may itself access that field or other memory, but its implementation and access footprint are not supplied.

The ordering matters: F-44362605ab79 completes before D-C is used for the second gate invocation. The contract must not assume that D-C, D-D, or the object addressed by the base remained unchanged during that call.

As in the first gate, all nonzero results continue execution regardless of sign or magnitude.

### Final field selection

When both gate invocations return nonzero results, the supplied representation contains two separate reads of the integer-shaped field at base plus 44 bytes:

1. The first read provides a candidate return value.
2. The second read determines whether the default global value is selected.

If the second read is zero, the function reads D-E and returns that value instead of the candidate.

If the second read is nonzero, the function returns the value obtained by the first read.

Under stable-memory conditions, this behaves as a conventional nonzero-field-or-default selection:

- A nonzero field value is returned directly.
- A zero field value selects D-E.

The two-read structure nevertheless matters to the contract. It does not establish that the tested value and the candidate value are identical. If the reads can legitimately observe different values, the second read controls default selection while the first read supplies the non-default result.

In particular, the function can return zero without reading D-E when the first read yields zero and the second yields a nonzero value. Conversely, a nonzero first read can be discarded when the second read yields zero.

These are consequences of the supplied representation, not evidence that concurrent mutation is supported. The actual memory model, volatility, synchronization, and instruction-level read behavior remain unknown.

### Selection precedence

The return sources have strict precedence:

1. D-B if the first gate returns zero.
2. Otherwise, D-D if the second gate returns zero.
3. Otherwise, D-E if the second offset-44 read is zero.
4. Otherwise, the first offset-44 read.

No later source overrides an earlier selected source. Selected global return values are not themselves tested for zero. Thus D-B, D-D, and D-E may each be returned as zero without triggering another fallback.

## Side effects

### Explicit effects

The supplied function body shows no explicit writes to:

- The base-addressed object.
- Any global data slot.
- Caller-provided output storage.

It also shows no explicit allocation, deallocation, reference-count adjustment, resource acquisition, or resource release.

Its explicit memory effects consist of data reads and dependency invocations.

### Dependency-mediated effects

The absence of direct writes does not establish purity. Any invoked dependency may change globals, modify accessible objects, acquire or release resources, or fail to return normally. No dependency body or side-effect contract is supplied.

In particular:

- The first two dependency invocations occur before D-B can be returned.
- F-44362605ab79 occurs before the second gate reads its D-C operand.
- Both gate stages complete before the final field reads.
- A gate invocation may affect a global value subsequently selected for return.

No ownership guarantee follows from the returned value. The function does not visibly retain, duplicate, validate, or release it. A caller must not infer that receiving a returned address, if it is an address, grants ownership or guarantees its lifetime.

### Errors and preconditions

There is no explicit error-handling branch separate from the selection logic. Zero gate results are ordinary branch conditions, not established error indicators.

To complete a particular path normally:

- Every dependency reached on that path must accept its supplied arguments and return normally.
- Any reached global read must be valid.
- The address passed to F-44362605ab79 must satisfy that dependency’s undocumented requirements.
- If the final stage is reached, both direct offset-44 reads must be valid.

The first early return avoids all base-derived accesses shown in this function. It does not establish that the dependencies themselves avoid unrelated memory accesses.

## Dependencies

### F-3ad65987df9a

- Invoked first, with no explicit arguments.
- Produces a four-byte value used as the first gate’s first operand.
- Its meaning, ownership behavior, global accesses, and failure modes are unknown.

### F-44362605ab79

- Invoked only if the first gate returns nonzero.
- Receives the address at base plus 40 bytes.
- Produces a four-byte value used as the second gate’s first operand.
- Whether it reads, mutates, initializes, or releases anything reachable from that address is unknown.

### F-unmapped-01

This is a document-local opaque ID for the gate dependency whose mapped identity was not supplied.

- It is invoked once on every normally executing path.
- It is invoked a second time only if its first result is nonzero.
- Its first operand comes from one of the two mapped dependencies.
- Its second operand is D-A on the first invocation and D-C on the second.
- A zero result selects the corresponding immediate global-return branch.
- Any nonzero result advances to the next stage.

Comparison-like use is evident from the zero tests, but the dependency’s actual operation and accepted operand layouts are unknown.

## Acceptance examples

These examples are conditional behavioral requirements derived from the supplied representation, not executed tests.

### First gate selects D-B

Given that F-3ad65987df9a returns normally and the first F-unmapped-01 invocation returns zero:

- The result is the subsequently read value of D-B.
- F-44362605ab79 is not invoked.
- No second gate invocation occurs.
- Neither direct offset-44 read occurs.

This remains the required selection even if D-B is zero.

### Second gate selects D-D

Given that the first gate returns any nonzero value, F-44362605ab79 returns normally, and the second gate returns zero:

- F-44362605ab79 receives base plus 40 bytes.
- The result is the subsequently read value of D-D.
- Neither direct offset-44 read occurs.

A negative first gate result is treated the same as any other nonzero result.

### Stable nonzero field

Given that both gates return nonzero and both offset-44 reads yield the same nonzero value:

- That field value is returned.
- D-E is not read for default selection.

### Stable zero field

Given that both gates return nonzero and both offset-44 reads yield zero:

- D-E is read and returned.
- D-E is returned even if its own value is zero.

### Different field observations

If the execution environment permits the two reads to observe different values:

- First read zero, second read nonzero: return zero without selecting D-E.
- First read nonzero, second read zero: discard the first value and return D-E.
- Two different nonzero reads: return the first value, not the second.

These examples do not impose synchronization guarantees absent from the evidence.

### Dependency changes a selected global

If the first gate changes D-B and then returns zero, the function returns the value read from D-B after that invocation. An implementation that unconditionally snapshots D-B before calling the dependencies would not preserve this ordering in such an environment.

## Uncertainties

The following are **documented_only**:

- The precise ABI, pointer width, integer signedness rules, and overflow behavior.
- The contents and encodings of D-A through D-E.
- Whether the returned bits denote text, an object, a handle, or a scalar.
- The actual operation and mapped identity of F-unmapped-01.
- Dependency mutation, ownership, lifetime, exception, and failure behavior.
- The layout and minimum size of the base-addressed object.
- The relationship, if any, between the regions at offsets 40 and 44.
- Whether globals are stable, volatile, thread-local, synchronized, or externally mutable.
- Whether the two represented offset-44 reads correspond to two observable machine reads.
- Behavior for invalid addresses, misalignment, arithmetic overflow, or concurrent unsynchronized access.

None of these unknowns is resolved by assigning conventional meanings to function names or data labels.

## Evidence

The supplied static representation establishes the ordered dependency calls, zero/nonzero branches, offsets 40 and 44, three global return alternatives, and the two-read final selection.

The dependency map establishes the identities F-3ad65987df9a and F-44362605ab79. No mapped identity or implementation was provided for the gate dependency.

Purpose beyond value selection is inferred, not established. No runtime tests, memory observations, dependency executions, or independent ABI verification have been performed.
