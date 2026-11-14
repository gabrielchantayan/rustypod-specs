<!-- Trusted provenance: F-a1c9cf91a2cc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a1c9cf91a2cc

## Purpose

The supplied static representation describes a non-returning sequence of indirect calls, address calculations, byte accesses, and 32-bit stores. The sequence repeats indefinitely, carrying derived values into the next iteration.

No application-level purpose is established. The representation does not establish that the accessed locations are ordinary objects, device registers, allocation records, or any other particular kind of storage. Descriptions such as “initialize,” “allocate,” “release,” or “commit” would imply behavior not supported by the available evidence.

This contract therefore specifies the visible data movement and ordering. It does not assign semantics to unresolved dependencies or treat their apparent signatures as verified interfaces.

## Interface

### Explicit inputs

The displayed entry interface has four arguments:

| Position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | Supplied to the first indirect call in each iteration. Replaced before repetition by a derived value. |
| Second | 32 bits | Supplied alongside the first argument. Replaced before repetition by an otherwise unexplained secondary call-result value. |
| Third | 32 bits | No explicit use appears in the supplied body. |
| Fourth | 32 bits, unsigned | Supplies a bit field for an early byte store. Replaced before repetition by a derived value. |

An unused explicit argument is not evidence that an external calling convention permits its omission.

### Additional entry state

The representation also consumes three values not initialized by the displayed interface:

- A 32-bit address-like value used for an initial byte read at offset 1.
- A 32-bit payload used in an initial word store and a later shifted call argument.
- A linkage-derived value used to calculate the first indirect-call target.

These are required entry-state inputs to the displayed behavior. Their origin and validity are **documented_only**.

A shared 32-bit value is read once per iteration in the later portion of the body. Its identity, storage class, and potential overlap with other accesses are not established.

### Outputs and termination

No ordinary return, return value, loop-exit condition, or explicit failure result appears.

If all calls return and all accesses complete, execution repeats. A dependency that does not return, a memory fault, or an invalid call target can prevent further progress; the body supplies no visible recovery path.

### Arithmetic and access conventions

For the formulas below:

- \(W(x)\) denotes the low 32 bits of a value.
- \(B(x)\) denotes its low 8 bits.
- Address arithmetic and left shifts are described as 32-bit bit-vector operations.
- Right shifts of the explicitly unsigned values are logical.
- A byte read produces a value from 0 through 255.
- A word access has width 32 bits.

This bit-vector interpretation captures the represented low-bit transformations without treating signed-looking multiplication and left shifts as portable language-level arithmetic. Exact target arithmetic and exception behavior remain unverified.

For compactness, define two address descriptions:

- A **direct word slot** for \(x\): address \(W(4x+20)\).
- An **indirect byte slot** for base \(a\): first read one byte at \(W(a+1)\), zero-extend it, then access the byte at that value plus 17.

The second description is not a conventional pointer dereference: its intermediate address component is only one byte wide.

## Behavior

### Ordering model

The body contains many intervening indirect calls. They are observable boundaries, not removable padding.

The phases below describe the explicit accesses in order and identify call results needed to explain them. Calls displayed without arguments are not assumed to have no effects or to receive no machine-level inputs. Unknown dependencies may alter shared memory, affect subsequent reads, fail, or not return.

The conceptual value names below describe retained results, not source identifiers.

### 1. Initial word publication and temporary byte field

The first indirect call receives the first and second explicit arguments. Its target depends on the incoming linkage-derived state. Four further calls follow; the last supplies an initial selector.

The incoming payload is written as a 32-bit word to that selector’s direct word slot. The next call receives four times the selector.

Four more calls follow, with the last supplying a primary selector. Its four-times-scaled value is retained.

The function then:

1. Reads a byte at the incoming address-like value plus 1.
2. Writes a byte at the resulting value plus 17.
3. Forms the written byte as  
   \(B((\text{fourth argument}\mathbin{\&}\mathtt{0x3fffffff}) \gg 20)\).

Although the mask preserves 30 bits, the final byte store retains only original input bits 20–27. Bits 28–29 do not survive the narrowing.

The ensuing calls visibly receive, in order:

- The incoming payload shifted left 12 bits.
- A returned 32-bit value together with the retained scaled primary selector shifted left 28 bits.
- The low and high 32-bit halves of a returned 64-bit quantity.

After further calls, a new result is obtained. The function freshly reads the byte at the incoming address-like value plus 1 and writes zero to that newly selected indirect byte slot. It does not reuse the earlier selector byte. The following call receives four times the new result.

### 2. Scaled word construction

After another run of calls, a result is scaled by four and retained as a word payload.

That payload is written at:

\[
W(4096\times\text{retained scaled primary selector}+20).
\]

The next visible call receives four arguments:

- The scaled primary selector multiplied by 1024.
- That same value again.
- The unscaled primary selector shifted left 14 bits.
- The retained word payload.

After additional calls, a new selector is obtained. The retained word payload is written to its direct word slot, and the following call receives four times that selector.

### 3. Two self-derived word sequences

A subsequent selector supplies both an address and a value:

- Store \(W(4x)\) at \(W(16x+20)\).
- Call a dependency with three copies of \(W(4x)\).

After intervening calls, the retained four-times-scaled value is written to another selector’s direct word slot. A following call receives four times that new selector.

A second sequence then repeats the same shape with a fresh selector \(y\):

- Store \(W(4y)\) at \(W(16y+20)\).
- Retain \(W(y\times\mathtt{0x40000000})\) as a byte-region base.
- Call a dependency with three copies of \(W(4y)\).

After further calls, another selector is obtained. The four-times-scaled value from the second sequence is written to that selector’s direct word slot, followed by a call receiving four times the selector.

The retained byte-region base can take only four 32-bit bit patterns under the bit-vector model, because multiplication by \(2^{30}\) retains only the selector’s lowest two bits.

### 4. Paired byte clears and indexed word read

Another dependency produces a 32-bit lookup value.

The function writes zero to the retained byte-region base plus 17, then writes zero to that base plus 1. These are direct addresses: no intervening byte lookup selects the first location.

A call receives the lookup value shifted left two bits. A subsequent returned 32-bit value is passed to another call. After more calls, a result is obtained, and the same two direct byte clears occur again, in the same order. The following call receives four times that result.

Further calls produce another selector. The function:

- Retains four times that selector.
- Stores a previously retained selector shifted left 31 bits at the new selector multiplied by 16, plus 20.
- Reads a 32-bit word at  
  \[
  ((\text{lookup value}\mathbin{\&}\mathtt{0x3fffffff})\gg20)+20.
  \]

The indexed word-read address ranges from 20 through 1043. The representation does not establish alignment or prove that all such addresses are valid.

A following call receives the retained four-times-scaled selector. Another call result is retained for later use. A subsequent call receives:

\[
W((\text{indexed word}\gg22)\ll28).
\]

Only original word bits 22–25 survive this final 32-bit left shift.

### 5. Byte field updates and a cached byte selector

Following another run of calls, a selector is obtained and zero is written to its direct word slot. The next call receives four times that selector.

Further calls produce a new 32-bit field value. The function reads a byte at the retained byte-region base plus 1 and writes \(B(4\times\text{field value})\) to the selected indirect byte slot.

Calls then visibly receive:

- Four times the field value.
- A returned value together with the field value shifted left 30 bits.

After more calls produce another result, the function reads the byte at the retained byte-region base plus 1 again. This time it retains that byte selector across several calls.

Using the retained byte selector, it first writes:

\[
B((\text{field value}\mathbin{\&}\mathtt{0x3fffffff})\gg20)
\]

at selector plus 17.

The ensuing call receives four times the latest result, followed by four further calls. The function then writes four times the earlier retained call result from phase 4 as a 32-bit word at the **cached byte selector plus 20**.

This final word address does not use a fresh byte read. An intervening dependency changing the underlying selector byte does not change this particular destination.

### 6. Shared-value snapshot and another temporary byte field

The function next reads the shared 32-bit value. This snapshot precedes the next indirect call.

That call receives the four-times-scaled selector retained in phase 4. After four more calls, a 64-bit quantity is returned.

The function retains four times its low half. It freshly reads the selector byte at the retained byte-region base plus 1 and writes:

\[
B(\text{shared snapshot}\gg22)
\]

to the selected indirect byte slot.

A dependency receives the high half of the 64-bit result. A later call receives a returned value together with the scaled low half shifted left 28 bits.

After further calls produce another result, the function freshly resolves the indirect byte slot again and writes zero. The following call receives four times the result.

### 7. Late word propagation

Further calls produce a value that is passed, multiplied by four, twice to one call.

After another call group, a fresh result is multiplied by four and written at:

\[
W(4096\times\text{scaled low half from phase 6}+20).
\]

A following call receives that earlier scaled low half shifted left 14 bits.

After more calls, another selector is obtained. The newly formed word payload is written to its direct word slot, and the next call receives four times that selector.

Another call group returns a 64-bit quantity. Four times its low half becomes a new retained scale.

Before processing that result further, the function reads a byte at:

\[
W(16384\times\text{scaled low half from phase 6}+1)
\]

and writes to the resulting byte value plus 17. The stored byte is:

\[
B((\text{latest word payload}\mathbin{\&}\mathtt{0x3fffffff})\gg20).
\]

The high half of the latest 64-bit result is supplied to the next call. Subsequent calls forward a 32-bit result and then the two halves of another 64-bit result.

### 8. Final clears and loop-carried state

After a further result is obtained, an indirect byte slot is resolved using a base equal to the new retained scale multiplied by \(2^{28}\). Zero is written there. The next call receives four times the latest result.

Further calls produce a final selector. Its four-times-scaled value is retained for the final word destination.

The address-like state for the next iteration becomes four times the new retained scale. An indirect byte slot resolved through this new address-like state is cleared.

Calls then visibly receive:

- The new retained scale shifted left 10 bits.
- A returned value together with the phase-4 retained result shifted left 28 bits.

After more calls, another result is produced. The function freshly resolves the indirect byte slot through the new address-like state and clears it again. A following call receives four times the result, followed by four further calls.

The function then:

1. Forms the next payload as the final selector shifted left 14 bits.
2. Stores that payload at the final selector multiplied by 16384, plus 20.
3. Sets the fourth argument for the next iteration to that payload.
4. Calls a dependency with the retained four-times-scaled final selector shifted left 14 bits.
5. Makes one further indirect call.
6. Sets the next first argument to the next payload shifted left 28 bits.
7. Sets the next second argument to an unexplained secondary result.
8. Repeats unconditionally.

Under 32-bit bit-vector arithmetic, the next first argument is zero: the payload has already been shifted left 14 bits, so shifting it another 28 bits discards every original bit.

The linkage-derived state for the next iteration is also replaced before repetition. Consequently, the first call target of a repeated iteration need not be governed by the original entry linkage value.

## Side effects

### Memory access properties

Explicit writes are either one byte or one 32-bit word. No explicit 16-bit store appears.

The body performs:

- Stores to addresses derived from call results.
- Byte-selected writes into addresses 17 through 272.
- Byte-selected word writes into addresses 20 through 275.
- An indexed word read within addresses 20 through 1043.
- Reads through carried and scaled address-like values.
- One shared 32-bit snapshot per iteration.

These are address ranges, not declarations that the ranges contain valid memory.

Aliasing is possible between explicit accesses and dependency effects. The representation warns of overlapping global symbols, but does not establish a complete alias map. Repeated writes and apparently redundant clears remain observable operations, especially if the memory is externally visible or dependencies inspect it.

### Ownership and resource lifetime

No ownership transfer, allocation, deallocation, reference-count change, lock acquisition, or lock release is established.

Passing a value to a dependency does not prove that the dependency owns it, frees it, or preserves storage reachable through it.

### Errors and partial progress

There is no visible input validation, bounds check, alignment check, status test, rollback, or exception handler. Earlier stores can remain observable if a later call or access fails.

Successful progress requires valid call targets, sufficient access permissions, target-compatible alignment behavior, and returning dependencies. These are preconditions, not guarantees supplied by this function.

## Dependencies

The only supplied resolved opaque function identity is **F-a1c9cf91a2cc** itself.

For reference, **F-00000001** denotes the unresolved indirect-call family in this contract. It is a documentation identifier, not a claim that all sites call one function.

The family includes:

- An entry-linkage-derived target.
- Numerous targets formed from embedded linkage-like values and offsets.
- Calls with displayed argument counts from zero through four.
- Results represented as 32-bit values, 64-bit values, and an unexplained secondary output.

Distinct sites must not be merged merely because their displayed argument patterns match. Conversely, distinct target expressions do not prove distinct implementations.

No preservation of globals, ownership behavior, hidden arguments, calling convention, or error protocol is established for F-00000001.

## Acceptance examples

These examples are conditional static expectations, not executed tests.

1. **Fourth-argument byte extraction.** If the fourth argument is `0x3AB00000`, the first temporary byte store writes `0xAB`, provided execution reaches the store and the destination is writable. The upper two bits remaining after the mask and shift are discarded.

2. **Fresh selector after calls.** If an early indirect byte write selects byte address 24, and intervening calls change the selector byte to 9, the later clear targets address 26. It need not clear address 24.

3. **Cached selector exception.** If phase 5 retains selector byte 7, its later word store targets address 27, even if dependencies subsequently change the selector byte in memory.

4. **Narrow address base.** A selector byte of 255 selects byte address 272 for an indirect byte-slot write. It does not select an address formed by loading a 32-bit pointer.

5. **Shift narrowing.** If the indexed word in phase 4 is `0x00400000`, the shifted call argument is `0x10000000`. If it is `0x04000000`, that argument is zero because the selected bit shifts beyond the 32-bit result.

6. **No completion on normal progress.** If every dependency returns and every access succeeds, execution begins another iteration rather than returning.

7. **Repeated-entry first argument.** Under the stated arithmetic model, every iteration after the first supplies zero as the first argument to its initial call. Its second argument remains dependent on unresolved secondary-result behavior.

## Uncertainties

The following are **documented_only**:

- Whether the displayed function boundary and four-argument interface match the actual executable interface.
- The origin of the initial address-like, payload, and linkage state.
- The actual identities and contracts of indirect callees.
- Whether apparently argumentless calls consume additional machine state.
- Whether 64-bit results truly represent paired return values.
- The origin and validity of the secondary output carried into the next iteration.
- Memory mapping, byte order, alignment requirements, and fault behavior.
- Whether any locations are volatile, device-backed, concurrently modified, or aliased.
- The meaning and exact storage layout of the shared snapshot.
- Whether omitted unreachable material affects interpretation of the recovered boundary.

Repeated temporary byte fields followed by clears suggest an intent to bracket dependency activity. That interpretation is an inference only; restoration of an earlier value is not performed.

## Evidence

Established evidence is limited to the supplied static representation: its displayed signature, explicit accesses, arithmetic expressions, indirect-call order, retained values, and unconditional repetition.

The unreachable-block and overlapping-symbol warnings limit confidence in structural and layout interpretation. They do not establish application semantics.

No implementation inspection, dependency resolution, runtime execution, memory trace, or runtime tests have been performed. The acceptance examples are arithmetic and ordering consequences conditional on the represented operations and stated preconditions.
