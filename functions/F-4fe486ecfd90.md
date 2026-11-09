<!-- Trusted provenance: F-4fe486ecfd90; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4fe486ecfd90

## Purpose

F-4fe486ecfd90 performs a fixed sequence of indirect calls interleaved with three 32-bit stores, two byte-selector reads and associated byte stores, a global-result assignment, two zeroing stores, and a final call to address zero.

**Established static evidence:** The supplied representation contains no explicit branch, loop, validation, or local error-recovery path. Its declared interface accepts one integer parameter and returns no value. Each operation is reached only if all preceding calls return and preceding accesses complete.

**Inferred intent:** Repeated scaling by four suggests word-indexed addressing. This does not establish an object layout, protocol, device operation, or resource-management purpose.

**Unknown:** Dependency behavior, the accuracy of the recovered calling convention, and runtime reachability are unresolved. The routine is not established to perform successful initialization, cleanup, serialization, or dispatch.

## Interface

### Explicit input and output

The declared input is one 32-bit integer, called the **input word** here. Its value is multiplied by four for the first store address and the first dependency argument.

There is no declared return value or visible status result. Observable outcomes consist of memory accesses, calls, a global update, and any resulting fault or non-returning control transfer.

### Implicit inputs

The supplied representation also depends on:

- An incoming 32-bit word associated with architectural register r7, used as the first stored value.
- An incoming address-like value associated with architectural register r6, used for a later byte read at offset one.
- The incoming link-register value, used to construct the first call target.
- A global base value, called **G-base**.
- A global destination, called **G-result**.
- An unresolved byte pointer used for a late zero-byte store.
- Memory mappings, callable targets, and other implicit state consumed by dependencies.

These dependencies do not establish a verified public calling convention.

### Widths and arithmetic

The representation distinguishes unsigned 8-bit reads, 8-bit stores, a 16-bit zero store, 32-bit values and word accesses, and two represented 64-bit results split into low and high 32-bit components.

This contract uses **scaled(x)** to denote the low 32 bits of four times a 32-bit word when describing address relationships and arguments. This notation is not an unconditional guarantee of wrapping arithmetic:

- Scaling the unsigned low component of the second 64-bit result has a clear 32-bit unsigned interpretation.
- Several other multiplications and shifts are represented using signed 32-bit values. Overflow, negative operands to signed left shifts, and overflowing signed address additions require machine-level confirmation.
- Examples below avoid those unresolved arithmetic cases.

Addresses are described in the apparent 32-bit address domain. The representation does not establish multi-byte endianness, required alignment, atomicity, or memory-ordering guarantees.

## Behavior

Dependency IDs identify call sites, not proven distinct implementations. All sequencing below is conditional on execution reaching the relevant operation.

### 1. Initial store and first paired result

The first visible memory operation stores the incoming r7-associated word at the address obtained by multiplying the input word by four and adding 20.

F-000000000001 is then invoked with the scaled input word. F-000000000002, F-000000000003, and F-000000000004 follow, in that order.

F-000000000005 produces a represented 64-bit result. Call its low 32-bit component **A** and its high 32-bit component **B**. The routine forms **V**, the represented 32-bit result of shifting A left by two, and stores V as a 32-bit word at **scaled(B) + 20**.

Only after that store does it read G-base. The resulting **captured base** is retained for a later load.

F-000000000006 is then invoked with four explicit arguments, in order: **V, B, V, V**.

An earlier store or dependency may affect the sampled G-base. A later change to G-base does not replace the captured value in the displayed computation.

### 2. Third word store and saved word

F-000000000007, F-000000000008, and F-000000000009 are invoked in order. F-000000000010 follows and supplies a represented 32-bit result, **C**.

The routine stores V as a 32-bit word at **scaled(C) + 20**. It then loads an unsigned 32-bit word from **captured base + 20**, retaining that value as **W**.

The third store precedes this load. If their addresses coincide, the load observes the stored word in the absence of external interference or exceptional memory semantics. The source establishes no disjointness requirement.

F-000000000011 receives scaled(C). F-000000000012, F-000000000013, and F-000000000014 follow in order.

F-000000000015 produces another represented 64-bit result. Call its low 32-bit component **L** and its high 32-bit component **H**.

W is retained for both subsequent **W-derived byte stores**; it is not visibly reloaded. Neither stored byte is guaranteed nonzero. The first uses W’s bits 0–7; the second uses bits 22–29.

### 3. First selected byte store and forwarded result

The routine reads an unsigned byte from **incoming r6-associated address + 1**. It adds 17 to the byte value and uses that sum as the destination address for an 8-bit store of W’s low byte.

The destination therefore lies in **17–272**, inclusive. It is not the incoming address plus 17, nor an address relative to the location from which the selector was read.

F-000000000016 then receives four explicit arguments: **H, H, H, scaled(L)**.

F-000000000017 supplies a represented 32-bit result that is passed directly as the sole explicit argument to F-000000000018. F-000000000019, F-000000000020, and F-000000000021 follow in order.

F-000000000022 then supplies a represented 32-bit result, **D**.

### 4. Second selected byte store

The second selector address is formed by retaining L’s low 30 bits, shifting them logically right by 21, and adding one.

This selects an unsigned byte from an address in **1–512**, inclusive. Equivalently, L’s bits 21–29 choose the offset; bits 30–31 do not affect this address.

The routine adds 17 to the selected byte and uses the sum as an 8-bit store destination. The stored value is the low eight bits of W shifted logically right by 22: exactly **W’s bits 22–29**, not its top byte.

This destination also lies in **17–272**. The two W-derived stores can target the same byte. The second can replace the first, although intervening dependency effects may also modify that location.

F-000000000023 receives scaled(D). F-000000000024, F-000000000025, and F-000000000026 follow in order.

F-000000000027 supplies a represented 32-bit result, **E**. F-000000000028 then receives **scaled(E), scaled(E)**.

### 5. Global update, zero stores, and final call

F-000000000029 is invoked. F-000000000030 then supplies a represented 32-bit result, which is passed as the sole explicit argument to F-000000000031.

F-000000000032 follows. Its represented result is assigned to G-result.

The routine next performs two distinct zeroing operations:

- An 8-bit zero store through the unresolved byte pointer.
- A 16-bit zero store at the address obtained by retaining L’s low 30 bits and shifting logically right by seven.

The halfword destination ranges from **0 to 8,388,607**, inclusive. It can be zero or odd. No visible check enforces non-nullness or two-byte alignment.

Finally, F-000000000033 attempts a call to address zero with three explicit zero arguments. The declared return is reached only if this call returns.

## Side effects

### Direct memory-access order

Ignoring accesses inside dependencies, the visible order is:

1. First 32-bit store, using the input-derived address.
2. Second 32-bit store, using B.
3. Read of G-base.
4. Third 32-bit store, using C.
5. Load of W from the captured base plus 20.
6. First unsigned selector-byte read.
7. First W-derived byte store.
8. Second unsigned selector-byte read.
9. Second W-derived byte store.
10. Assignment to G-result.
11. Zero-byte store through the unresolved pointer.
12. Zero-halfword store derived from L.

The calls interleaved with these accesses occur as specified in Behavior. This order is the order exposed by the supplied representation, not proof of volatile semantics, hardware transaction order, memory barriers, or atomicity.

### Aliasing and timing

No visible condition requires any addresses to be disjoint. Stores may overlap other stores, selector bytes, globals, or dependency-owned memory.

Consequently:

- G-base must be interpreted at its actual sampling point.
- W reflects memory at its later load point, not memory when G-base was sampled.
- Each selector reflects its own read point, not function entry.
- The first W-derived store may affect memory used by the second selector read if their addresses overlap.
- Later writes to the original W location do not visibly cause W to be reloaded.

Dependencies may change globals or memory between these operations. Their preservation of implicit machine state is not established.

### Ownership and failures

No explicit allocation, release, ownership transfer, bounds check, rollback, or cleanup appears. An argument’s use as an address-forming value does not establish ownership or object lifetime.

Access faults, alignment faults, dependency failures, non-returning calls, and invalid call targets have no visible local recovery path. Earlier completed effects are not visibly undone. Whether they remain observable in a particular environment depends on that environment’s fault and memory semantics.

## Dependencies

The following opaque IDs designate the 33 call sites in execution order. “None shown” means no explicit arguments appear; it does not exclude implicit register inputs or an incompletely recovered signature.

| Dependency | Explicit arguments | Visible result use |
|---|---|---|
| F-000000000001 | Scaled input word | None |
| F-000000000002 | None shown | None |
| F-000000000003 | None shown | None |
| F-000000000004 | None shown | None |
| F-000000000005 | None shown | Split into A and B |
| F-000000000006 | V, B, V, V | None |
| F-000000000007 | None shown | None |
| F-000000000008 | None shown | None |
| F-000000000009 | None shown | None |
| F-000000000010 | None shown | Supplies C |
| F-000000000011 | Scaled C | None |
| F-000000000012 | None shown | None |
| F-000000000013 | None shown | None |
| F-000000000014 | None shown | None |
| F-000000000015 | None shown | Split into L and H |
| F-000000000016 | H, H, H, scaled L | None |
| F-000000000017 | None shown | Forwarded to F-000000000018 |
| F-000000000018 | Result of F-000000000017 | None |
| F-000000000019 | None shown | None |
| F-000000000020 | None shown | None |
| F-000000000021 | None shown | None |
| F-000000000022 | None shown | Supplies D |
| F-000000000023 | Scaled D | None |
| F-000000000024 | None shown | None |
| F-000000000025 | None shown | None |
| F-000000000026 | None shown | None |
| F-000000000027 | None shown | Supplies E |
| F-000000000028 | Scaled E, scaled E | None |
| F-000000000029 | None shown | None |
| F-000000000030 | None shown | Forwarded to F-000000000031 |
| F-000000000031 | Result of F-000000000030 | None |
| F-000000000032 | None shown | Assigned to G-result |
| F-000000000033 | Zero, zero, zero | None; final call |

F-000000000001 targets the incoming link-register value plus 4,094. The intermediate targets are constructed from embedded code-location values, generally with an offset of 4,094. F-000000000016 instead uses 4,076, and F-000000000030 uses 176. F-000000000033 targets zero.

These constructions establish indirect control flow, not dependency identity, service semantics, ownership behavior, or a verified calling convention.

## Acceptance examples

These are conditional expectations derived from static evidence. **None has been executed.** Unless stated otherwise, assume prior operations complete, dependencies return, and no unspecified effects or external interference alter the illustrated values.

### Initial and result-derived stores

For input word 3 and incoming stored word hexadecimal 11223344:

- The first store writes that 32-bit value at address 32.
- F-000000000001 receives 12.

If A is 5 and B is 7:

- V is 20.
- The second word store writes 20 at address 48.
- F-000000000006 receives 20, 7, 20, 20.

These values avoid signed-scaling overflow. The example does not prescribe byte order.

### Captured base and later memory contents

If G-base is 4,096 when sampled, but a later dependency changes it to 8,192, W is still loaded from address 4,116.

W is the word present at 4,116 at load time. If the third store targets that same address, it precedes the load and can determine W.

### W-derived bytes, including zero

For W equal to hexadecimal AB123456:

- The first W-derived store writes hexadecimal 56.
- The second writes hexadecimal AC.

For W equal to zero, **both stores write zero**. There is no visible condition that suppresses either store because its data is zero.

If both selector reads return 9, both destinations are address 26. Assuming no intervening modification, the second W-derived store determines the byte there immediately after that store.

### Selector boundaries

For the second selector read:

- L equal to zero selects address 1.
- L equal to hexadecimal 3FFFFFFF selects address 512.
- Changing only L’s bits 30–31 leaves the selector address unchanged.

For either selector, byte value 255 produces destination address 272. The byte is unsigned; it does not produce a negative offset.

### Unaligned and zero halfword destinations

With L equal to 128, the final halfword destination is address 1. The routine requests a 16-bit zero store there without visible alignment adjustment.

With L equal to zero, the destination is address zero. Completion of either access depends on platform mappings and alignment behavior.

### Failure at the final call

If all preceding operations complete but address zero is not callable, the final call may fault or otherwise transfer control according to the platform. No normal return, error code, or rollback is required by the visible representation.

## Uncertainties

The following remain **documented_only**:

- Accuracy of the recovered function boundary, parameter declaration, and calling convention.
- Whether the two represented 64-bit results are genuine wide returns, paired register values, or recovery artifacts.
- Origin, validity, lifetime, and ownership of the unresolved byte pointer. Its apparent association with late call output does not identify a defining dependency.
- Dependency identities, implicit arguments, side effects, preservation rules, ownership obligations, and return behavior.
- Exact behavior of signed scaling, signed shifts, overflowing address arithmetic, and conversion to executable or data addresses.
- Whether low numeric addresses designate ordinary memory, device mappings, privileged mappings, or invalid locations.
- Endianness, alignment requirements, atomicity, concurrency effects, and fault semantics.
- Exact declared width and semantics of G-result; its assigned dependency result is represented as word-sized.
- Whether the zero-target call is intentional, reachable, or accurately reconstructed.
- Any resource-lifetime obligations not exposed by this routine.

These unknowns do not justify assuming callee purity, safe memory mappings, successful completion, implicit cleanup, or fallback behavior.

## Evidence

This contract is based solely on the supplied static representation of F-4fe486ecfd90.

Established observations include the access widths, call order, explicit argument order, two split results, masks and shifts, timing of the G-base capture and W load, reuse of W for both W-derived byte stores, global assignment, zeroing operations, and final zero-target call.

The byte-width stores select W’s bits 0–7 and 22–29. Neither extraction imposes a nonzero condition; W equal to zero produces zero for both.

Word-indexed storage is inferred intent, not an established data model. Dependency semantics, machine-specific arithmetic, implicit-state provenance, and runtime validity remain unresolved.

No implementation access, dependency inspection, runtime tests, or execution traces were available or performed.
