<!-- Trusted provenance: F-3e0ee9c2a43a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3e0ee9c2a43a

## Purpose

The supplied static representation describes a routine that performs an ordered series of indirect calls, interleaved with word and byte stores, and then enters an unconditional repeating sequence. No normal return is represented.

The observable work includes:

- Writing the first input to an address derived from that same input.
- Obtaining values from opaque dependencies and using them as address indices, stored values, and later call arguments.
- Extracting bit fields from inputs and dependency results.
- Repeatedly reading byte-sized selectors and writing through addresses derived from those selectors.
- Maintaining a loop-carried value obtained from a dependency.

No application-level purpose is established. In particular, the evidence does not establish that the addresses denote ordinary buffers, device registers, tables, or valid allocated objects.

**Evidence limitation:** the supplied representation reports problematic instruction decoding and removed unreachable regions. This contract describes its visible execution sequence, not independently verified machine behavior. Any requirement below is conditional on execution reaching the relevant operation and on the representation being accurate.

## Interface

### Explicit inputs

| Input | Represented width and interpretation | Visible use |
|---|---|---|
| First input | Unsigned 32-bit value | Initial word-store address and value; first dependency argument; masked index for later selector reads |
| Second input | Opaque 32-bit quantity | No explicit use in the visible body |
| Third input | Unsigned 32-bit value | Logically shifted right by 7 and passed to one dependency |
| Fourth input | Unsigned 32-bit value | Stored as a word; logically shifted right by 7 for a byte-store address and a dependency argument |

No explicit length, validity flag, error-output pointer, or ownership-transfer marker is present.

The absence of explicit use of the second input does not establish that it has no machine-level effect: indirect-call conventions and implicit register inputs are not recovered.

### Implicit execution inputs

The first indirect call target depends on entry link-register state. Other indirect targets are expressed through address-derived calculations. Consequently, execution context and callable target validity are prerequisites beyond the four explicit inputs.

Dependencies may read or modify memory and global state. Their preservation properties are unknown.

### Output and termination

The represented interface has no return value. The visible path ends in an unconditional loop, with no explicit success or failure return.

This does not establish that an invocation always runs forever: a dependency may not return, an access may fault, or omitted or misrepresented instructions may alter control flow.

### Width and arithmetic conventions

The representation distinguishes:

- 32-bit unsigned inputs and several unsigned dependency results.
- 32-bit signed intermediates for some other dependency results and arithmetic.
- 32-bit word stores.
- 8-bit selector reads and 8-bit stores.

Unsigned masks and right shifts have their ordinary 32-bit meanings. Byte stores retain only the low eight bits of their source value.

Some address calculations and left shifts use represented signed intermediates. Overflow and shifts of negative or excessively large values are not safely defined by that representation. When this contract gives low-32-bit arithmetic examples for those operations, it explicitly assumes the corresponding target-machine behavior; it does not impose that behavior as established fact.

Byte order, word alignment requirements, pointer width, and the validity of the accessed address space are not independently established.

## Behavior

### Notation and dependency identity

`F-U01` through `F-U60` are opaque identifiers assigned here to **call sites**. They are not recovered function identities. Two sites may resolve to the same function, and a target may depend on execution context. Only F-3e0ee9c2a43a has a supplied function identity.

A call described as having no explicit arguments has no recovered argument expressions in the supplied representation. This does not prove that the machine-level callee receives no arguments.

Named results below are descriptions of data roles, not recovered program identifiers.

### 1. Initial write and initial calls

The first visible operation is a 32-bit store:

- Address: 20 plus four times the first input.
- Value: the first input.

The multiplication and addition originate from unsigned 32-bit arithmetic; wraparound therefore matters when interpreting the represented expression.

The routine then performs these calls in order:

1. `F-U01`, with the first input.
2. `F-U02`, with no explicit arguments; its 32-bit result is passed to `F-U03`.
3. `F-U04`, with no explicit arguments; its 32-bit result is passed to `F-U05`.
4. `F-U06`, `F-U07`, and `F-U08`, each with no explicit arguments.
5. `F-U09`, with no explicit arguments, producing an index result.

The initial store precedes every listed call. No status check or conditional branch is shown between them.

### 2. Fourth-input word store

Using the result of `F-U09`, the routine writes the fourth input as a 32-bit word at:

- 20 plus four times that result.

It then calls:

1. `F-U10`, with four times that result.
2. `F-U11`, `F-U12`, and `F-U13`, with no explicit arguments.
3. `F-U14`, with no explicit arguments, obtaining a second index result.

The address arithmetic based on these represented signed results has unresolved overflow behavior.

### 3. Paired word stores around intervening calls

The routine forms an address base equal to sixteen times the result of `F-U14`. It writes a 32-bit word at that base plus 20, storing four times the same result.

It also derives a selector index from the first input:

- Clear bit 31.
- Logically shift the remaining value right by 7.

That index is between 0 and 16,777,215 inclusive.

The routine then calls:

1. `F-U15`, with the address base as both its first and second explicit arguments, and the third input logically shifted right by 7 as its third.
2. `F-U16`, `F-U17`, and `F-U18`, with no explicit arguments.
3. `F-U19`, with no explicit arguments.

After these calls, it writes another 32-bit word at the same previously computed base plus 20. This time it stores the base itself.

These two stores use the same represented destination, but they are separated by five calls. Neither write may be treated as redundant: a dependency may observe or modify that location.

Next, it calls:

1. `F-U20`, with the result of `F-U19` shifted left by 2.
2. `F-U21`, `F-U22`, and `F-U23`, with no explicit arguments.
3. `F-U24`, with no explicit arguments, obtaining a 32-bit unsigned result.

### 4. First selector-based byte write

The selector operation used repeatedly below consists of:

1. Reading an unsigned byte at the first-input-derived selector index plus 1.
2. Adding 17 to the byte value read.
3. Writing one byte at that resulting address.

Thus, the final byte-write address is between 17 and 272 inclusive. It is not the selector-read address plus 17.

For this first selector-based write, the stored byte is the low eight bits of the `F-U24` result shifted left by 2.

The routine then calls:

1. `F-U25`, with that shifted result as all three explicit arguments.
2. `F-U26`, with no explicit arguments.
3. `F-U27`, with the result of `F-U26`.
4. `F-U28`, `F-U29`, and `F-U30`, with no explicit arguments.
5. `F-U31`, with no explicit arguments.

### 5. Second selector-based byte write

The routine reads the selector byte again from the same first-input-derived selector address. It does not reuse the previous selector value.

At the address selected by this new read, it stores the low eight bits of the following value derived from the earlier `F-U24` result:

- Mask with 1,073,741,823.
- Logically shift right by 20.

The intermediate is at most 1,023; conversion to a byte further discards its upper two bits. The stored byte therefore reflects bits 20 through 27 of the original result.

Next it calls:

1. `F-U32`, with the result of `F-U31` shifted left by 2.
2. `F-U33`, `F-U34`, and `F-U35`, with no explicit arguments.
3. `F-U36`, with no explicit arguments, obtaining another unsigned 32-bit result.

### 6. Fourth-input-derived byte write and selector clear

The routine stores one byte at:

- The fourth input logically shifted right by 7, plus 28.

The stored byte is the low eight bits of the `F-U36` result shifted left by 2.

It then freshly reads the first-input-derived selector byte and writes zero to the selected address. The fourth-input-derived write precedes this selector read and zero write.

It next calls:

1. `F-U37`, with the fourth input logically shifted right by 7.
2. `F-U38`, with no explicit arguments.
3. `F-U39`, with the `F-U36` result masked with 1,073,741,823 and logically shifted right by 6.
4. `F-U40`, `F-U41`, and `F-U42`, with no explicit arguments.
5. `F-U43`, with no explicit arguments.

The argument to `F-U39` ranges from 0 through 16,777,215.

### 7. Further selector clear and loop preparation

The routine again reads the first-input-derived selector byte and writes zero to the selected address.

It then calls:

1. `F-U44`, with the result of `F-U43` shifted left by 2.
2. `F-U45`, `F-U46`, and `F-U47`, with no explicit arguments.
3. `F-U48`, with no explicit arguments, obtaining an unsigned 32-bit result.

From the `F-U48` result it derives an initial loop field:

- Mask with 1,073,741,823.
- Logically shift right by 25.

This field ranges from 0 through 31.

A second selector-read address is derived from the earlier `F-U36` result:

- Take that result shifted left by 2.
- Multiply the represented shifted value by 1,024.
- Add 1.

The multiplication passes through a represented signed intermediate. Its overflow semantics are unresolved. Under ordinary low-32-bit machine arithmetic, the net scaling before addition is a left shift by 12.

The routine reads a byte at this second selector-read address, adds 17 to that byte, and writes zero at the resulting address.

It then calls:

1. `F-U49`, with twice the initial loop field.
2. `F-U50`, with no explicit arguments, obtaining the initial loop-carried result.

### 8. Unconditional repeating sequence

The loop executes at least once if reached and has no represented exit condition.

Each iteration performs, in order:

1. Call `F-U51` with the loop-carried result and the current loop field logically shifted right by 8.
2. Call `F-U52`, `F-U53`, and `F-U54`, with no explicit arguments.
3. Call `F-U55`, with no explicit arguments.
4. Read a fresh selector byte at the second selector-read address, add 17, and write zero to the selected address.
5. Call `F-U56` with the result of `F-U55` shifted left by 2.
6. Call `F-U57`, `F-U58`, and `F-U59`, with no explicit arguments.
7. Call `F-U60`, with no explicit arguments.
8. Replace the loop-carried result with the `F-U60` result shifted left by 2.
9. Replace the loop field with the `F-U60` result shifted left by 26.
10. Repeat.

The first invocation of `F-U51` has a second explicit argument of zero because the initial loop field is at most 31.

Under low-32-bit shift semantics, later loop fields retain only the low six bits of the corresponding `F-U60` result, placed in bits 26 through 31. Shifting that field right by 8 yields those six bits in positions 18 through 23. The signed left shifts used to produce these values remain an arithmetic uncertainty.

## Side effects

### Memory access and ordering

The visible body performs word writes, selector-byte reads, and byte writes. It shows no explicit word reads from the destinations it writes.

Each selector read is a separate observable operation. Dependencies may change the selector between reads, so successive writes may target different addresses even when the selector-read address is unchanged.

Aliasing is not excluded. A destination may overlap another destination, a selector byte, dependency-managed state, or executable memory. Such overlap may affect later observations. The specified order must not be replaced with a final-memory-state-only interpretation.

No atomicity, volatility, memory barrier, synchronization, or concurrency guarantee is established.

### Ownership and lifetime

There is no visible allocation, deallocation, or ownership-transfer operation in the body itself.

This does not constrain the dependencies. No dependency is known to preserve allocations, retain no pointers, release no memory, or leave globals unchanged. All directly accessed addresses must remain valid at the moment of access for execution to continue normally.

### Errors

The body contains no explicit validation, error conversion, retry, or recovery path. Dependency results used as indices are not checked. Dependency results not used locally are not tested for success.

A failed call or invalid access has no specified recovery behavior.

## Dependencies

All calls are opaque. Their only established properties here are call order, explicit argument expressions, and which returned values are subsequently used.

| Call-site group | Visible role |
|---|---|
| `F-U01` | Receives the first input; target depends on entry execution state |
| `F-U02`–`F-U05` | Two producer-to-consumer result transfers |
| `F-U06`–`F-U14` | Intervening calls and production/use of word-store indices |
| `F-U15`–`F-U24` | Repeated base arguments, third-input field, and production of the first byte payload |
| `F-U25`–`F-U36` | Repeated payload arguments, another result transfer, and production of the second byte payload |
| `F-U37`–`F-U48` | Fourth-input field, payload field, and initial loop-field production |
| `F-U49`–`F-U50` | Loop initialization |
| `F-U51`–`F-U60` | Repeating calls and loop-state replacement |

No callee is established to be pure, nonblocking, successful, returning, or memory-preserving. No two sites are established to denote different functions.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

1. **Initial unsigned address calculation.** With first input 1, the first visible operation writes the 32-bit value 1 at address 24. With first input 1,073,741,824, represented unsigned 32-bit scaling wraps to zero, giving address 20 and preserving the original input as the stored value. Both examples require writable destinations.

2. **Selector indirection.** With first input 0, the first-input-derived selector-read address is 1. If a particular read there returns 5, that operation writes at address 22, not address 18. If a dependency changes the selector to 9 before the next read, the next selected destination is 26.

3. **Byte truncation.** If `F-U24` returns 1, the first selector-based payload byte is 4. If it returns 64, that byte is 0 because the shifted value is 256. If it returns 268,435,456, the later field-derived byte is also 0: the right-shifted intermediate is 256 and is truncated to eight bits.

4. **Ordered word replacement.** If `F-U14` returns 3 and arithmetic is valid, the routine first stores 12 at address 68. After `F-U15` through `F-U19` return, it stores 48 at address 68. A dependency observing that address between the writes may observe 12.

5. **Fourth-input-derived destination.** With fourth input 128, the corresponding byte-store address is 29 and the explicit argument to `F-U37` is 1. The subsequent selector-based zero write occurs after that byte store.

6. **First loop argument.** For every unsigned result of `F-U48`, the first second argument supplied to `F-U51` is zero.

7. **Later loop state, conditional on wrapping shifts.** If `F-U60` returns 1, the next `F-U51` receives 4 as its first explicit argument and 262,144 as its second. This example requires low-32-bit shift behavior.

8. **No inferred short circuit.** An apparent error-like return from an opaque dependency does not, by itself, authorize skipping later visible calls or writes. The body shows no such check; the dependency may still independently prevent continuation.

## Uncertainties

The following are **documented_only**:

- Actual instruction-level control flow, given the decoding warnings and removed regions.
- Concrete identities, calling conventions, implicit inputs, and effects of the indirect dependencies.
- Address-space meaning, access permissions, alignment, byte order, and device-memory behavior.
- Signed-overflow and signed-left-shift behavior in derived addresses and loop values.
- Whether any call returns normally or alters control state outside the recovered model.
- Application-level meaning of the inputs, extracted fields, stores, and repeating sequence.
- Behavior after invalid accesses, invalid call targets, or dependency failures.

An initialization or polling interpretation is possible, but remains inferred intent rather than an acceptance requirement.

## Evidence

**Established static evidence:** four represented 32-bit inputs; an unused second explicit input; ordered indirect calls; 32-bit stores; fresh unsigned-byte selector reads; byte truncation; unsigned masks and right shifts; and an unconditional repeating tail.

**Inferred intent:** none is required to explain the observable sequence. No device, protocol, allocation model, or semantic status interpretation is assumed.

**Unknowns:** dependency behavior, machine-level arithmetic details beyond the represented unsigned operations, and fidelity of the recovered control flow.

No runtime tests, execution traces, or independent implementation checks have been performed.
