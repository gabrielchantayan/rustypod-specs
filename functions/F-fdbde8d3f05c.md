<!-- Trusted provenance: F-fdbde8d3f05c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fdbde8d3f05c

## Purpose

F-fdbde8d3f05c performs an ordered sequence of indirect calls, memory reads, memory writes, and a global assignment, ending with an attempted call to address zero.

No application-level purpose is established. Repeated fourfold scaling and bit extraction could reflect encoded references or packed metadata, but those interpretations are not part of the behavioral contract. The evidence does not establish initialization, allocation, serialization, device control, or cleanup.

This contract describes the explicit operations in the supplied static representation. It does not establish that the memory accesses are valid, that dependencies return, or that the terminal call is reachable.

## Interface

### Explicit arguments and return

| Argument | Represented type | Explicit use |
|---|---|---|
| First | Signed 32-bit integer | Shifted left by two, stored, and supplied twice to the first dependency. |
| Second | 32-bit value of unspecified interpretation | No explicit use. |
| Third | Unsigned 32-bit integer | Its bottom two bits are cleared to construct the initial destination base. |

The apparent return type is void. No explicit result or status is returned to the caller.

The unused second argument is not evidence that dependencies ignore its incoming register contents. Actual calling conventions and implicit register dependencies are unresolved.

### Ambient inputs

The operation also depends on:

- An incoming return-link value used to calculate the first call target.
- An ambient unsigned 32-bit quantity used for field extraction and, later, address construction.
- A global slot whose value is captured as a memory base.
- An incompletely represented register-result pointer used for a late byte store.
- Memory contents at the addresses selected during execution.
- The behavior and implicit inputs of all dependencies.

The explicit arguments alone are insufficient to determine the complete behavior.

### Widths and arithmetic

The representation distinguishes 8-bit bytes, 16-bit halfwords, 32-bit words, and two 64-bit dependency results. Each 64-bit result is separated into numerical upper and lower 32-bit halves. This does not establish memory endianness or physical return-register order.

Unsigned 32-bit shifts and masks have direct bit-level meanings. In particular:

- Clearing the third argument’s bottom two bits and adding `0x14` uses unsigned 32-bit arithmetic, including wraparound.
- Adding one to the ambient unsigned address base uses unsigned 32-bit arithmetic.
- Shifting the second paired result’s unsigned lower half left by two retains the low 32 bits.

Several other expressions use signed 32-bit values. Negative signed left shifts, signed overflow, and overflowing signed multiplication or addition have no established portable result. Converting a 32-bit bit pattern outside the signed range into a signed value also requires execution-model information not supplied here.

Accordingly, concrete acceptance examples involving signed scaling assume values for which the represented arithmetic is defined. No general modulo-\(2^{32}\) rule is imposed on signed expressions.

The result assigned to the final global slot has **unresolved width**. The destination slot’s storage width is also unresolved. Neither width follows from the surrounding 32-bit operations.

## Behavior

The sequence below is conditional on each preceding access completing and each preceding dependency returning normally. Dependency IDs identify call sites, not verified external function identities.

### 1. Initial store and forwarding calls

The first argument is shifted left by two. The resulting 32-bit value is written at offset `0x14` from the third argument after clearing that argument’s bottom two bits.

Third-argument values differing only in those two bits select the same destination. The store occurs before any dependency call.

F-000000000001 receives two copies of the scaled first argument. Its target is calculated by adding `0xf9c` to the incoming return-link value. The representation does not establish that this calculation produces a valid executable address.

The following calls occur afterward, in order:

1. F-000000000002 produces a 32-bit value passed to F-000000000003.
2. F-000000000004 produces a 32-bit value passed to F-000000000005.
3. F-000000000006, F-000000000007, and F-000000000008 execute without explicit arguments.

No additional explicit data-memory access appears between these calls. Their internal accesses and effects remain unknown.

### 2. Ambient-field word store

F-000000000009 produces a signed 32-bit selector.

The routine stores a 32-bit word at four times that selector plus `0x14`. The stored value is the ambient unsigned quantity shifted right by 23 bits, yielding a value from 0 through 511.

The selector is used directly in address construction. No pointer-sized read through it is represented.

F-000000000010 receives four times the selector. F-000000000011, F-000000000012, and F-000000000013 then execute in that order, without explicit arguments.

### 3. First paired result and global-base capture

F-000000000014 produces a 64-bit result.

Its lower 32 bits are interpreted as a signed value and shifted left by two. This scaled value is retained for later use.

Its upper 32 bits are interpreted as a signed selector. The retained scaled value is stored as a 32-bit word at four times that selector plus `0x14`.

After that store, the routine captures a global slot’s value into a represented signed 32-bit base value. The global declaration and underlying access width are not supplied. At this point, the routine does not yet read the word at offset `0x14` from the captured base.

F-000000000015 receives four explicit arguments:

1. The retained scaled lower value.
2. The upper-half selector.
3. The retained scaled lower value again.
4. The retained scaled lower value again.

F-000000000016 through F-000000000018 follow in order.

### 4. Repeated store and delayed data capture

F-000000000019 produces another signed 32-bit selector.

The retained scaled value from F-000000000014 is stored as a 32-bit word at four times this new selector plus `0x14`.

Immediately after that store, the routine reads a 32-bit unsigned word at offset `0x14` from the previously captured global base. This is the **captured data word** used by both later data-derived byte stores.

The base capture and the pointed-to read are separate events:

- The base is captured before F-000000000015.
- The data is read after F-000000000019 and its associated store.

Thus, intervening dependencies can change the data eventually observed. Merely changing the global base slot after capture does not redirect the represented read. The immediately preceding store can itself affect the read if the locations overlap.

F-000000000020 receives four times the new selector. F-000000000021 through F-000000000023 then execute in order.

### 5. Second paired result and first data-derived byte store

F-000000000024 produces a second 64-bit result. Its numerical upper and lower 32-bit halves are retained.

Before the next call, the routine reads one unsigned byte at offset one from the ambient unsigned base. The loaded byte plus `0x11` is used directly as a destination address.

At that destination, the routine writes bits 0–7 of the captured data word. The destination therefore ranges from `0x11` through `0x110`, inclusive. There is no intervening pointer-sized dereference.

The stored byte may be zero. There is no nonzero-value condition or branch controlling this store.

F-000000000025 receives:

1. The second paired result’s upper half.
2. The same upper half.
3. The same upper half.
4. Its lower half shifted left by two using unsigned 32-bit arithmetic.

F-000000000026 then produces a 32-bit value passed to F-000000000027. F-000000000028 through F-000000000030 follow in order.

### 6. Second data-derived byte store

F-000000000031 produces a signed 32-bit value retained for the next call.

Before that call, the routine reads another unsigned selector byte. Its address is derived from the second paired result’s lower half by retaining bits 0–29, shifting right by 21, and adding one. The resulting read address ranges from 1 through `0x200`, inclusive.

The loaded selector byte plus `0x11` selects the destination, again within `0x11` through `0x110`.

The routine writes bits 22–29 of the captured data word to that destination. Bits 30–31 are discarded by narrowing the shifted value to one byte. This store also has no nonzero-value condition.

If both selector reads choose the same destination, this later write replaces the earlier byte there, subject to intervening dependency effects. The captured data word is not explicitly reloaded between the two writes.

F-000000000032 receives the result of F-000000000031 shifted left by two.

### 7. Remaining calls and global assignment

F-000000000033 through F-000000000035 execute in order.

F-000000000036 produces a signed 32-bit value. F-000000000037 receives two copies of that value shifted left by two.

The remaining nonterminal calls are:

1. F-000000000038.
2. F-000000000039, whose 32-bit result is passed to F-000000000040.
3. F-000000000041, whose result is assigned directly to a global slot.

For the last assignment, neither the callee’s return type nor the global’s declaration is supplied. Consequently:

- The returned value’s width and interpretation are unresolved.
- The destination’s storage width and representation are unresolved.
- Any conversion involved in the assignment is unresolved.
- The destination’s independence from other storage is not established.

The assignment is an established event; describing it as a 32-bit or word-sized transfer would exceed the evidence.

### 8. Final zeroing and terminal call

After the global assignment, the routine performs these operations in order:

1. Writes one zero byte through an incompletely represented register-result pointer.
2. Writes a zero halfword to an address derived from the second paired result’s lower half.
3. Attempts F-000000000042 at address zero with three explicit zero arguments.

The halfword destination is obtained by retaining the lower half’s bits 0–29 and shifting right by seven. Its range is zero through `0x007fffff`, inclusive.

This destination can be odd. No alignment, accessibility, or range check is represented. Whether an odd halfword access completes depends on the execution environment.

The byte-zeroing pointer’s source, validity, and lifetime are unresolved. Its appearance after F-000000000041 does not establish that this dependency alone produced or owns it.

The final operation is an attempted call, not an explicit trap or error return. If the target faults or does not return, normal completion is not reached. If address-zero callable behavior exists and returns normally, the representation shows no further work before the void return.

## Side effects

### Explicit data-access order

Excluding accesses internal to dependencies, the represented order is:

| Order | Access |
|---:|---|
| 1 | Initial 32-bit store through the third-argument-derived destination. |
| 2 | 32-bit store of the ambient quantity’s upper nine bits. |
| 3 | 32-bit store of the first paired result’s scaled lower value. |
| 4 | Capture of the global base slot’s value; underlying access width unresolved. |
| 5 | Repeated 32-bit store of the retained scaled value. |
| 6 | 32-bit read through the captured base. |
| 7 | Unsigned byte read through the ambient-derived address. |
| 8 | Byte store of captured-data bits 0–7. |
| 9 | Unsigned byte read through the second paired result’s derived address. |
| 10 | Byte store of captured-data bits 22–29. |
| 11 | Assignment to the result-receiving global; value and storage widths unresolved. |
| 12 | Zero-byte store through the unresolved pointer. |
| 13 | Zero-halfword store through the derived address. |

Calls are interleaved as specified under Behavior. This order does not imply that dependencies are memory-free or that the machine supplies synchronization guarantees.

### Aliasing and retained state

Aliasing is not excluded. Writes can affect later reads, global storage, selector bytes, dependency state, or later destinations.

After its capture, the data word is explicitly retained rather than reloaded. Later changes to its original memory location do not themselves change that retained value in the representation.

The two uses of the ambient unsigned quantity are separated by many calls. No dependency implementation establishes preservation of the underlying register or ambient state.

### Ownership and failures

No allocation, release, ownership transfer, locking, validation, or rollback is explicit. Accessing an address does not establish ownership or lifetime.

Unknown dependencies must not be assumed to preserve globals or pointed-to memory, nor to allocate or release storage.

There is no explicit recovery branch or error code. A fault, nonlocal exit, or nonreturning dependency can prevent all subsequent effects. No represented action reverses earlier writes.

## Dependencies

These opaque IDs designate ordered call sites. Different IDs do not establish different runtime targets.

| IDs | Explicit role |
|---|---|
| F-000000000001 | Return-link-derived target; receives two scaled first-argument values. |
| F-000000000002–F-000000000005 | Two 32-bit producer-to-consumer chains. |
| F-000000000006–F-000000000008 | Three calls without explicit arguments. |
| F-000000000009–F-000000000010 | Signed selector producer and scaled-selector consumer. |
| F-000000000011–F-000000000013 | Three calls without explicit arguments. |
| F-000000000014–F-000000000015 | First 64-bit producer and four-argument consumer. |
| F-000000000016–F-000000000018 | Three calls without explicit arguments. |
| F-000000000019–F-000000000020 | Signed selector producer and scaled-selector consumer. |
| F-000000000021–F-000000000023 | Three calls without explicit arguments. |
| F-000000000024–F-000000000025 | Second 64-bit producer and four-argument consumer. |
| F-000000000026–F-000000000027 | 32-bit producer-to-consumer chain. |
| F-000000000028–F-000000000030 | Three calls without explicit arguments. |
| F-000000000031–F-000000000032 | Signed producer and shifted-result consumer. |
| F-000000000033–F-000000000035 | Three calls without explicit arguments. |
| F-000000000036–F-000000000037 | Signed producer and consumer receiving two shifted copies. |
| F-000000000038–F-000000000040 | One call followed by a 32-bit producer-to-consumer chain. |
| F-000000000041 | Producer assigned directly to a global; return width unresolved. |
| F-000000000042 | Address-zero target receiving three explicit zeros. |

Apart from the first and final targets, the calls use embedded code-location calculations with fixed displacements. Their executability, calling conventions, and actual prototypes are not established.

“Without explicit arguments” does not mean independent of registers, flags, memory, or other ambient inputs.

## Acceptance examples

These are conditional static criteria, not executed tests. Each assumes that preceding operations complete, selected memory is accessible, signed arithmetic is defined, and dependencies have the stipulated behavior.

### Initial destination

With first argument 3 and third argument `0x1003`, the initial store writes the 32-bit value 12 at `0x1014`. F-000000000001 subsequently receives 12 twice.

Changing the third argument to `0x1000` leaves that destination unchanged.

### Nine-bit extraction

If the ambient quantity at the field-extraction step is `0xffffffff` and F-000000000009 returns `0x100`, the word store writes 511 at `0x414`.

### Captured base and delayed contents

Suppose the captured global base is `0x2000`. An intervening dependency changes the global slot to `0x3000` and changes the word at `0x2014`.

The later explicit data read still targets `0x2014`. It observes the contents at read time, including any overlapping effect of the immediately preceding word store. Changing the global slot alone does not redirect that read.

### Zero data does not suppress stores

If the captured data word is zero, both data-derived byte stores write zero. Their selector reads still occur, and the loaded selector bytes still determine their destinations.

### Extraction and repeated destination

For captured data `0x2ac0005a`, the first data-derived byte is `0x5a`; the second is `0xab`.

If both selector reads yield 4, both destinations are `0x15`. The later store writes `0xab` after the earlier `0x5a`, absent other effects altering the scenario.

### Distinct derived addresses

If the second paired result’s lower half is `0x3fffffff`, the second selector byte is read at `0x200`, while the final zero-halfword destination is `0x007fffff`.

The halfword destination is odd. Successful completion requires an environment permitting that access.

### Final assignment without an invented width

If F-000000000041 returns normally, its result is assigned to the designated global before either final zeroing store.

Acceptance does not require a particular numerical value, 32-bit truncation, or a particular number of bytes written by that assignment. Those properties are unresolved.

### Failure ordering

If the initial store faults, the representation establishes no subsequent dependency call.

If the final address-zero call faults, preceding completed stores and the global assignment have no explicit rollback.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and accuracy of the apparent function boundary.
- Actual ABI, argument conventions, dependency identities, clobbers, and internal effects.
- Concrete results of signed arithmetic or signed conversions outside their portable defined cases.
- Preservation of the ambient register-derived quantity across calls.
- Source, validity, lifetime, and ownership of the final byte-zeroing pointer.
- Global declarations, access widths, overlap, volatility, and synchronization requirements.
- F-000000000041’s return type and width.
- The final global assignment’s destination width and any conversion semantics.
- Endianness, memory mapping, alignment rules, and address-zero behavior.
- Exceptions, nonlocal exits, nonreturning dependencies, and full-sequence reachability.

Word-indexed references and packed metadata are possible interpretations of the scaling and masks. They are inferred intent only, not required behavior.

## Evidence

The supplied static representation is the sole behavioral evidence. It establishes the explicit access widths where typed accesses are present, the masks and shifts, call ordering, argument forwarding, delayed data capture, and absence of explicit validation or recovery branches.

Both data-derived byte stores are unconditional along the represented sequence; neither excludes zero.

The final nonterminal call is assigned directly to an undeclared global without an explicitly supplied return type. That establishes the assignment but not the returned value’s width or the destination’s storage width.

An accompanying warning reports overlapping global symbols. It supports caution about storage independence but does not establish a particular overlap layout.

No dependency implementations, ABI documentation, memory map, or execution traces were supplied. No runtime tests have been performed. The acceptance examples describe conditional consequences of static evidence, not observed executions.
