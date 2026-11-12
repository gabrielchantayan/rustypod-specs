<!-- Trusted provenance: F-4deeab2a2696; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4deeab2a2696

## Purpose

F-4deeab2a2696 describes a visible sequence of indirect dependency calls, byte accesses, 32-bit accesses, and assignments to an undeclared global. Values obtained from ambient machine state, dependency results, and memory determine subsequent arguments, destinations, and stored values.

No application-level purpose is established. In particular, the evidence does not establish initialization, serialization, allocation, resource release, or a validated object layout.

The supplied representation reports invalid instruction data and ends where control-flow recovery is truncated. This contract therefore specifies the **conditional behavior of the visible path**, not a verified, complete execution of the original machine function.

## Interface

### Explicit inputs and output

The displayed interface has two 32-bit inputs:

- The first has no visible use.
- The second is passed to F-U01.

No return value is displayed. No normal return is visible.

### Ambient inputs

The visible path also requires:

- An ambient base address, initially used for byte accesses at offsets 1 and 13.
- An ambient unsigned 32-bit word, supplying values for the first two byte stores.
- An ambient call-target origin. F-U01’s target is formed by adding 3816 to this origin.
- A secondary unsigned 32-bit machine-result value observed after F-U14 and used as an argument to F-U15.

These are dependencies of the supplied representation, not established public parameters. The secondary result’s producer and calling-convention meaning are unresolved.

### Widths and arithmetic

Explicit byte accesses are eight bits. Explicit word accesses and ordinary result values are 32 bits. Two dependency results are displayed as 64-bit values and separated into low and high 32-bit halves.

In this contract:

- **Low byte** means the least significant eight bits.
- **Scaled by a factor** means the apparent 32-bit bit-vector result, retaining the low 32 bits.
- Right shifts of values described as unsigned insert zeroes.
- Address arithmetic is described in the apparent 32-bit address space.

Some scaling expressions in the evidence involve signed-looking intermediates. The bit-vector description records the apparent intended operation; it does not establish portable signed-overflow semantics or independently validate the original instructions.

The byte order of 32-bit memory accesses is unknown.

### Global assignment destination

Two visible assignments target the same undeclared global. This contract calls it the **unnamed global** without assigning it an address or storage width.

The first assignment supplies the second byte selector. The second supplies the integer value 192. Neither the byte width of the first value nor the magnitude of the second establishes the destination’s storage width.

The global’s declaration, authoritative address mapping, conversion behavior, and physical write footprint are **documented_only** unknowns.

### Preconditions

Reaching every visible event requires readable source addresses, writable destinations, callable targets, compatible actual calling interfaces, and dependencies that return control. The global assignments additionally require a valid destination under its missing declaration.

No visible checks enforce these conditions. Some computed word addresses may be unaligned; whether such accesses are supported is unknown.

## Behavior

### Ordering and dependency notation

F-U01 through F-U65 identify successive visible indirect call sites. They are opaque identifiers, not recovered function identities. Different identifiers may resolve to the same callee.

A consecutive range denotes calls in increasing order. “No explicit arguments” describes only the supplied representation; it does not establish the absence of hidden arguments or ambient inputs.

The following phases preserve the order of visible memory events and calls. Dependencies may introduce additional effects, including changes to memory read later. No preservation of global memory is assumed.

### Initial selector accesses and calls

The first explicit memory operation reads an unsigned byte at the ambient base plus 1. That byte selects a destination by addition of 17. The function stores the ambient word’s low byte there. This destination is between addresses 17 and 272 inclusive; the selector is a byte value, not a loaded pointer.

F-U01 then receives the second explicit input. F-U02 is called without explicit arguments, and its unsigned 32-bit result is retained as the **initial result**. F-U03 receives that result. F-U04–F-U07 follow without explicit arguments, with F-U07’s 32-bit result retained.

The byte at ambient base plus 1 is read again. This **second selector** is retained independently of the first. At address second selector plus 17, the function stores the low byte of the ambient word shifted right by 22.

F-U08 receives F-U07’s result scaled by 4. F-U09–F-U12 follow without explicit arguments.

### Derived byte and ordered global assignments

The second selector determines a **small base**, equal to twice its unsigned value and therefore between 0 and 510.

A **derived byte** is the low byte of the initial result shifted right by 3. The function first stores this byte at ambient base plus 13, then assigns the second selector to the unnamed global.

F-U13 receives, in order, the small base, 152, and 608. F-U14 follows without explicit arguments. F-U15 then receives the secondary unsigned machine-result value shifted right by 3. The evidence does not establish this value as a conventional return from F-U14.

F-U16–F-U19 follow without explicit arguments. F-U20 receives F-U19’s result scaled by 4 and produces another retained 32-bit result.

Next, the function stores the derived byte at small base plus 13 and then assigns 192 to the unnamed global. These are distinct ordered events. The second event is not established as a one-byte store or as an access to absolute address 13.

F-U21 receives F-U20’s result scaled by 4. F-U22–F-U25 follow without explicit arguments.

### Small-base word access and loaded-address transition

F-U25’s result, called the **scaling result**, supplies two values:

- Its value scaled by 4 is stored as a 32-bit word at small base plus 36.
- Its value scaled by 32 is retained as the **expanded result**.

After the word store, the function reads a 32-bit word at small base plus 116. This is the **loaded address word**.

F-U26 receives four arguments, in this order:

1. The loaded address word.
2. The loaded address word scaled by 2.
3. The expanded result.
4. The loaded address word with bit 31 cleared, then shifted right by 2.

F-U27–F-U30 follow without explicit arguments. The function then stores the expanded result as a 32-bit word at loaded address word plus 36.

A fresh 32-bit read at loaded address word plus 116 supplies F-U31’s argument. This read is separate from the earlier read at small base plus 116, even if the computed addresses coincide.

F-U32–F-U34 follow without explicit arguments.

The offsets 36 and 116 establish address calculations only. They do not establish object boundaries, field ownership, or validation.

### Split results and indexed word stores

F-U35 is called without explicit arguments and has a displayed 64-bit result. Its low and high 32-bit halves have separate uses:

- The low half scaled by 4 is retained as the **saved scaled result**.
- At address high half scaled by 4, plus 20, the function stores the initial result shifted right by 3 as a full 32-bit word.

This store does not truncate the value to the earlier derived byte. Its value can contain up to 29 significant bits.

F-U36 receives the saved scaled result three times, followed by F-U35’s high half shifted right by 3.

F-U37 then produces another displayed 64-bit result. F-U38 receives that result’s low half, high half, low half again, and high half shifted right by 3, in that order.

F-U39–F-U42 follow without explicit arguments. The function stores the saved scaled result as a 32-bit word at F-U42’s result scaled by 4, plus 20. F-U43 receives F-U42’s result scaled by 4.

F-U44–F-U47 follow without explicit arguments. F-U47’s unsigned 32-bit result is retained as the **late result**; its value scaled by 4 is the **late scaled result**.

### Later selector-based byte stores

The function reads an unsigned byte at loaded address word plus 1. At that byte’s value plus 17, it stores the late scaled result’s low byte.

F-U48–F-U50 follow without explicit arguments, retaining F-U50’s result. The function then reads the byte at loaded address word plus 1 again. At this newly selected destination plus 17, it stores the low byte of the late result after masking with 0x3FFFFFFF and shifting right by 20.

These are separate selector reads. Their resulting destinations need not match, including where intervening dependencies or an earlier aliased write change the selector location.

F-U51 receives F-U50’s result scaled by 4.

### Final zero-byte stores and visible endpoint

F-U52–F-U55 follow without explicit arguments.

The next selector address is the late scaled result multiplied by 1024, plus 1, under the apparent 32-bit arithmetic model. The function reads a byte there and stores zero at that byte’s value plus 17.

F-U56 receives:

1. The late scaled result scaled by 8.
2. The same value again.
3. The late scaled result.
4. F-U55’s result scaled by 4.

F-U57–F-U60 follow without explicit arguments. The function reads another byte using the same selector-address expression and stores zero at the newly read byte’s value plus 17.

F-U61 receives F-U60’s result scaled by 4. F-U62–F-U65 follow without explicit arguments.

The representation then ends at an invalid-instruction termination marker. This establishes neither a normal return nor a particular hardware trap or process outcome.

## Side effects

### Explicit memory events

The visible representation contains **14 store or assignment events**:

| Event category | Count |
|---|---:|
| Explicitly byte-sized stores | 8 |
| Explicitly 32-bit stores | 4 |
| Assignments to the undeclared global | 2 |

It contains **eight explicit reads**:

| Read category | Count |
|---|---:|
| Byte reads | 6 |
| 32-bit reads | 2 |

These counts exclude dependency accesses and indirect-call machinery. The two global assignments are not counted as explicitly byte-sized stores: their destination width is absent from the evidence.

Every byte-selected destination is between addresses 17 and 272. The explicit byte destination at small base plus 13 is between addresses 13 and 523. These are literal address calculations, not evidence that low memory is accessible or intended as ordinary storage.

### Aliasing

The calculations do not exclude aliasing among explicit accesses, dependency accesses, and the unnamed global.

When the second selector is zero, small base plus 13 equals absolute address 13. The derived-byte store therefore targets address 13, followed by assignment of 192 to the unnamed global.

An exact overwrite relationship requires missing global metadata:

- If the global is one byte wide and mapped to address 13, the assignment replaces that byte with 192.
- If the global is wider and overlaps address 13, the resulting byte depends on its type, assignment conversion, storage layout, and byte order.
- If the global does not overlap address 13, the assignment does not overwrite that byte through its destination.

The global’s spelling in the supplied representation does not establish any of these cases.

### Ownership and failure

No explicit allocation or release is identifiable. No dependency is established to acquire ownership, release storage, preserve passed memory, or preserve globals.

There is no visible rollback, validation branch, error status, retry, or recovery path. If a memory access or call prevents continuation, earlier effects may already have occurred. The mechanism of failure is platform-dependent and unresolved.

## Dependencies

| Opaque IDs | Established interaction |
|---|---|
| F-U01 | Receives the second explicit input; target depends on the ambient call-target origin. |
| F-U02–F-U12 | Produce or consume the initial result and the first scaled dependency result. |
| F-U13–F-U25 | Consume the small base, constants, secondary machine result, and intermediate scaled results. |
| F-U26–F-U35 | Consume loaded words and the expanded result; F-U35 supplies the first split result. |
| F-U36–F-U47 | Consume split-result components and produce indexed-store and late-selector values. |
| F-U48–F-U65 | Interleave further calls with selector rereads and the final zero-byte stores. |

There are 65 visible indirect call sites. Except for F-U01’s ambient-origin calculation, targets are represented through arithmetic involving fixed code-location values. This does not establish valid entry points, stable APIs, or distinct callees.

Dependency implementations, calling conventions, memory effects, termination behavior, and ownership semantics remain **documented_only** unknowns.

## Acceptance examples

These are conditional static expectations, not runtime observations. They assume valid accesses, compatible callable targets, returning dependencies where required, and the apparent bit-vector arithmetic described above.

1. **Initial byte selection.** If the first selector is 5 and the ambient word is 0x89ABCDEF, the first store writes 0xEF at address 22. If the second selector is also 5, the second store writes 0x26 there: shifting the ambient word right by 22 gives 0x226, whose low byte is 0x26.

2. **Independent selector reads.** If the first selector is 5 and the second is 9, the first two stores target addresses 22 and 26. Substituting the first read for the second would not satisfy the visible behavior.

3. **Byte versus word value.** If the initial result is 0x12345678, the derived byte is 0xCF. The later full-word store derived from that result writes 0x02468ACF, not 0x000000CF.

4. **Global assignment without assumed mapping.** If the second selector is zero, the small-base derived-byte store writes at address 13. The following event assigns 192 to the unnamed global. Only if that global is established as one byte at address 13 does the latter event establish a byte overwrite with 0xC0. A wider overlapping global requires its conversion rules and byte order to determine the resulting byte.

5. **Masked argument.** If the loaded address word is 0xFFFFFFFF, F-U26 receives 0xFFFFFFFF, 0xFFFFFFFE, the expanded result, and 0x1FFFFFFF. The fourth argument clears bit 31 before shifting.

6. **Split-result addressing.** If F-U35’s high half is 3, its associated word store targets address 32. If its low half is 7, F-U36’s first three arguments are each 28.

7. **Final selector reread.** If the first final selector read yields 4 and the second yields 8, the zero-byte stores target addresses 21 and 25. The repeated selector-address expression does not permit eliminating the second read.

8. **Partial execution.** If an early dependency does not return, later stores and calls are not required to occur. Earlier completed effects are not visibly rolled back.

## Uncertainties

- **Control-flow validity — documented_only:** invalid-instruction warnings limit confidence in the represented path and its endpoint.
- **Application intent — inferred only:** repeated scaling and fixed offsets may suggest structured manipulation, but no application domain or object model is established.
- **Global metadata — documented_only:** declaration, width, signedness, address mapping, conversion rules, write footprint, and aliasing relationships are unavailable.
- **Calling interface — documented_only:** ambient inputs, secondary results, hidden arguments, and displayed 64-bit returns require independent calling-convention evidence.
- **Arithmetic execution — documented_only:** signed-looking shifts and multiplication do not independently prove machine overflow behavior.
- **Memory model — documented_only:** mapping, alignment support, byte order, memory-mapped effects, and access atomicity are unknown.
- **Dependency behavior — documented_only:** mutation, preservation, allocation, release, errors, and termination are unspecified.
- **Concurrency — documented_only:** no synchronization or concurrency guarantees are established.
- **Final outcome — documented_only:** the visible termination marker does not determine actual continuation, trapping, or process termination.

## Evidence

Established static evidence consists of the supplied representation’s displayed widths, ambient-state references, 65 indirect call sites, arithmetic expressions, eight explicit reads, eight explicit byte stores, four explicit word stores, two assignments to an undeclared global, and invalid-instruction warnings.

The ordering and value relationships above follow that representation conditionally. They do not independently validate original-machine behavior where recovery is incomplete or arithmetic interpretation is uncertain.

No authoritative global declaration, global address mapping, dependency implementation, verified calling convention, memory map, or independent instruction-level validation was supplied. No runtime tests have been performed.
