<!-- Trusted provenance: F-a998ccce7cf4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a998ccce7cf4

## Purpose

F-a998ccce7cf4 is a dispatching wrapper that divides an unsigned quantity into calls of at most 256 units, adjusts a position and a second integer argument between full-size calls, and stops issuing data-processing calls when one returns a nonzero result.

Two independent conditions select one of four indirect processing dependencies:

- Whether the fifth input is zero.
- Whether a byte in an associated state object is zero.

The selection is repeated for each processing call. A separate indirect dependency supplies the increment factor used to advance the second integer argument after each full-size call.

These are established control-flow properties. Interpreting the quantity as a byte count, the second argument as a buffer address, the position as a stream offset, or a nonzero result as an error would require information not present in the supplied evidence.

## Interface

### Inputs

The function has five inputs, described here by behavioral role rather than source identifiers.

| Input | Representation in the supplied evidence | Use |
|---|---|---|
| Context | Pointer to an array-like object of 32-bit integer words | Provides an initial position adjustment, an associated-state reference, and an indirect-dispatch table reference. |
| Advancing argument | Signed 32-bit integer | Zero selects a special immediate return. Otherwise forwarded to processing dependencies and advanced between full-size calls. |
| Position | Signed 32-bit integer | Initially adjusted using context state, then advanced by 256 after each full-size call. |
| Quantity | Unsigned 32-bit integer | Determines processing-call sizes and continuation. Its remainder is reduced to 16 bits after each full-size call. |
| Selector | Signed 32-bit integer | Tested only for zero versus nonzero to select a processing dependency family. |

The 32-bit word and address layout is supported by the supplied typed representation and its access offsets. It does not establish a portable interface for systems with different integer or pointer widths.

### Result

The result is pointer-shaped, with three relevant cases:

1. If the advancing argument is zero, the function returns the pointer-shaped value represented by `0xffffffff`.
2. Otherwise, if any processing dependency returns a nonzero pointer-shaped result, that result is returned unchanged.
3. If all processing dependencies return zero, the function eventually returns zero.

No pointed-to result object is read or written by this function. A result is only retained, compared against zero, and returned.

The evidence does not establish whether a nonzero result is an error, an object reference, a status encoded as a pointer, or another kind of token. In particular, the wrapper does not distinguish the special `0xffffffff` value from other nonzero results when a dependency returns it.

### Preconditions

When the advancing argument is nonzero:

- The context must support the reads described below.
- References loaded from the context must identify readable state and dispatch-table storage when accessed.
- Each selected dispatch entry must be callable with the arguments described in this contract.
- Storage required by later reads must remain valid after earlier dependency calls.

The function performs no checks for these conditions.

When the advancing argument is zero, none of those context-access preconditions is exercised: the supplied control flow returns the sentinel without dereferencing the context.

Signed arithmetic limits are a separate precondition for a contract expressed using ordinary signed 32-bit arithmetic. Behavior when those limits are exceeded is documented under **Uncertainties**.

## Behavior

### Immediate sentinel path

The advancing argument is tested for zero before any context memory is read.

If it is zero:

- The position, quantity, and selector do not affect the result.
- No dependency is invoked.
- No context field, associated-state byte, or dispatch entry is read.
- The result is `0xffffffff` in the pointer-shaped return representation.

A zero quantity does not select this path unless the advancing argument is also zero.

### Initial position adjustment

For a nonzero advancing argument, the function reads the 32-bit context word at byte offset `+0x18` and adds it to the input position.

That adjusted position is used for the first processing call.

This context adjustment is read once. Changes to that context word during dependency calls do not cause the adjustment to be recomputed.

### Processing dependency selection

Immediately before each processing call, the function:

1. Reads the associated-state reference from the context word at byte offset `+0x10`.
2. Reads the byte at offset `+0x5c` from that associated state.
3. Uses the selector’s zero/nonzero classification and the byte’s zero/nonzero classification to choose a dispatch-table entry.
4. Reads the dispatch-table reference from the context word at byte offset `+0x00`.
5. Reads and invokes the selected entry.

The selection table is:

| Selector | Associated-state byte | Dependency | Entry offset |
|---|---|---|---|
| Zero | Zero | F-D01 | `+0xa8` |
| Zero | Nonzero | F-D02 | `+0xac` |
| Nonzero | Zero | F-D03 | `+0x90` |
| Nonzero | Nonzero | F-D04 | `+0x8c` |

Every processing dependency receives four arguments, in this order:

1. The original context pointer.
2. The current advancing argument.
3. The current adjusted position.
4. The quantity for this call.

The fifth wrapper input is not forwarded. Its role is solely to choose a dispatch family.

The associated-state reference, state byte, dispatch-table reference, and selected entry are not cached across processing calls. Consequently, dependency-induced changes to those objects can alter later dispatch choices.

### Terminal-size processing

If the current unsigned quantity is at most 256, the function invokes the selected processing dependency once with that exact quantity and returns its result.

This includes a quantity of zero. There is no zero-quantity fast path for a nonzero advancing argument.

For a terminal-size call:

- No increment-factor dependency is invoked afterward.
- No subsequent processing call occurs.
- The result is returned whether zero or nonzero.

### Full-size processing

If the current unsigned quantity exceeds 256, the function invokes the selected processing dependency with a quantity of exactly 256.

After that dependency returns, the following sequence occurs even if its result is nonzero:

1. Subtract 256 from the unsigned quantity and retain only the low 16 bits.
2. Advance the current position by 256.
3. Read the current dispatch-table reference from context offset `+0x00`.
4. Read and invoke F-D05 through entry offset `+0x20`, passing only the context.
5. Multiply F-D05’s signed integer result by 256 and add that product to the current advancing argument.

Only after this sequence does the wrapper cease processing because of a nonzero processing result.

Thus, a nonzero result from a full-size processing call does **not** suppress the subsequent F-D05 call. Conversely, a nonzero result from a terminal-size call is returned without any F-D05 call.

If the full-size processing result is zero, processing continues using the updated arguments and reduced quantity.

### Quantity reduction

The quantity reduction after a full-size call is not ordinary unrestricted 32-bit subtraction alone. It is subtraction followed by a mask to 16 bits.

Consequences include:

- Quantities from 257 through 65,536 are processed as conventional sequences of full-size calls followed by a final call of 1 through 256 units, assuming every processing result is zero.
- Larger initial quantities can lose high-order quantity bits after the first full-size call.
- A full-size call can be followed by a terminal call with quantity zero.
- Once a full-size call has occurred, the remaining quantity is between zero and 65,535.

After that first reduction, each further full-size step subtracts 256 from a value greater than 256, so it cannot increase the quantity or underflow it.

If dependencies return normally, the wrapper’s quantity-based iteration is bounded. With all processing results zero, there are at most 257 processing calls, including the final terminal-size call.

### Advancing argument behavior

The advancing argument is checked for zero only on entry.

It is not checked again after an increment. If arithmetic produces zero later, that zero is forwarded to the next processing dependency rather than triggering the sentinel path.

A negative initial advancing argument is nonzero and therefore enters normal processing. No range or sign validation is performed.

F-D05 is invoked separately after every full-size processing call. Its result is not cached. The increment factor may therefore differ between iterations.

## Side effects

### Direct accesses

The wrapper directly reads:

- The initial context position-adjustment word.
- The associated-state reference and selected state byte before each processing call.
- The dispatch-table reference and selected processing entry before each processing call.
- The dispatch-table reference and F-D05 entry after each full-size processing call.

The supplied body contains no direct writes to caller-visible memory.

Changes to the current quantity, position, advancing argument, and saved result are local state changes. The evidence does not show those values being written back into context fields.

### Dependency-visible ordering

For a full-size iteration, the externally observable dependency order is:

- One processing call with quantity 256.
- One F-D05 call.
- Either return of the saved nonzero processing result or continuation to another processing call.

F-D05 is selected from the dispatch-table reference read **after** the processing call. If the processing dependency changes that reference, the subsequent F-D05 lookup observes the changed reference.

Likewise, a processing dependency or F-D05 may change state used for the next processing selection. The wrapper does not restore it.

The position and quantity updates preceding F-D05 are local. F-D05 receives only the context, not those updated local values.

### Ownership and lifetime

The wrapper itself does not allocate, copy, release, or transfer ownership of any object shown in the evidence.

That observation must not be extended to its dependencies. Their allocation behavior, ownership rules, global-state effects, and ability to invalidate context-related storage are unknown.

Normal continuation requires dependency calls to leave every object needed by subsequent reads valid. The source does not enforce that requirement or recover if it is violated.

### Errors

There is no wrapper-level validation, exception translation, retry, rollback, or error classification.

A nonzero processing result stops further processing calls, subject to the mandatory post-full-size F-D05 invocation. F-D05 has no separately interpreted error channel here: its returned integer is used directly as an arithmetic factor.

## Dependencies

The following opaque IDs are document-local labels for unresolved indirect targets. They are not claims that concrete target identities have been recovered.

| ID | Invocation interface | Role established by this function |
|---|---|---|
| F-D01 | Context, advancing argument, position, unsigned call quantity → pointer-shaped result | Processing target for selector zero and state byte zero. |
| F-D02 | Same as F-D01 | Processing target for selector zero and state byte nonzero. |
| F-D03 | Same as F-D01 | Processing target for selector nonzero and state byte zero. |
| F-D04 | Same as F-D01 | Processing target for selector nonzero and state byte nonzero. |
| F-D05 | Context → signed 32-bit integer | Supplies the factor for advancing the second argument after every full-size call. |

These labels identify dispatch roles, not necessarily distinct functions. Entries may alias, and entry contents may change between reads.

No dependency implementation or contract is supplied. Nothing here assumes that a dependency preserves globals, preserves dispatch state, or retains or releases memory.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless stated otherwise, context accesses are valid, dependencies return normally, and signed arithmetic remains representable.

### Sentinel bypass

Given an advancing argument of zero, the result is `0xffffffff`, with no context reads or dependency calls. This remains true for any quantity, including zero.

### Zero quantity

Given a nonzero advancing argument and quantity zero, the selected processing dependency is called once with quantity zero and the initially adjusted position. Its result is returned. F-D05 is not called.

### Exactly 256 units

Given quantity 256, there is one processing call with quantity 256. F-D05 is not called, regardless of the processing result.

### Quantity 257 with successful continuation

Suppose the initial adjusted position is 1,000, the advancing argument is 5,000, and F-D05 returns 3.

If the first processing result is zero:

- The first processing call receives advancing argument 5,000, position 1,000, and quantity 256.
- F-D05 is then invoked.
- The terminal processing call receives advancing argument 5,768, position 1,256, and quantity 1.
- The terminal result is returned.

The target selected for the terminal call may differ from the first target if intervening calls change dispatch-related state.

### Nonzero result from a full-size call

Given quantity 257, if the first processing call returns a nonzero token, F-D05 is still called once. No terminal processing call follows. The original token is returned unchanged.

This criterion also applies if the reduced remaining quantity exceeds 256.

### High-order quantity truncation

Given initial quantity 65,792 and a zero result from the first processing call:

- The first processing quantity is 256.
- The remaining quantity becomes zero because 65,536 is reduced to its low 16 bits.
- F-D05 is called.
- The next processing call has quantity zero.

The wrapper does not process all 65,792 units under this input.

### Entry-only zero check

Given quantity 257, advancing argument 256, and an F-D05 result of −1, a zero first processing result leads to a terminal processing call whose advancing argument is zero. The wrapper does not return the sentinel merely because the updated argument became zero.

## Uncertainties

### Established static evidence

The branch conditions, dispatch offsets, argument order, quantity thresholds, 16-bit remainder mask, repeated lookups, and mandatory F-D05 call after a full-size processing result are directly established by the supplied body.

### Inferred intent

The structure suggests chunked processing with position tracking and factor-scaled advancement. These are descriptive interpretations only. Units, domain meaning, and success/error semantics are not established.

### documented_only

The following remain unresolved:

- Concrete identities and behavior of all indirect targets.
- Whether the advancing argument encodes an address, index, handle, or another value.
- The units of the position and quantity.
- Whether nonzero processing results represent errors.
- Ownership or validity of returned pointer-shaped tokens.
- Dependency effects on globals, storage lifetime, and dispatch structures.
- Signed-overflow behavior for initial position addition, position advancement, multiplication by 256, and advancing-argument addition.
- Exact ABI details beyond the 32-bit layout represented in the supplied evidence.
- Concurrency, synchronization, reentrancy, and abnormal control transfer from dependencies.

The unsigned quantity subtraction and mask are explicit. The evidence does not justify silently assigning wraparound semantics to the separate signed arithmetic operations. Acceptance criteria involving those operations therefore require representable intermediate and final signed values.

## Evidence

This contract is based solely on the supplied static function body for F-a998ccce7cf4.

No implementation of an indirect dependency, external interface specification, runtime trace, or test result was provided. No runtime tests have been performed.

The contract distinguishes directly visible wrapper behavior from inferred processing intent and unresolved dependency behavior. No snapshot hash or independent provenance assertion is included.
