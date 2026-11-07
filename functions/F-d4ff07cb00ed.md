<!-- Trusted provenance: F-d4ff07cb00ed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d4ff07cb00ed

## Purpose

F-d4ff07cb00ed is represented as a state-changing routine that performs two byte stores interleaved with twelve indirect calls. It uses its first explicit input both as stored data and as an address-calculation operand. It also depends on ambient machine state not expressed in its three-argument interface.

No application-level purpose is established. In particular, the available material does not establish that the routine initializes an object, operates on a valid table, or implements a particular protocol.

The behavioral contract below separates:

- **Established static evidence:** operations and ordering represented in the supplied listing.
- **Inferred intent:** interpretations that might explain those operations but are not required behavior.
- **Unknowns:** information the listing cannot establish, designated **documented_only**.

The listing has an explicitly incomplete terminal control-flow reconstruction. Consequently, this contract describes the represented path, conditional on each preceding operation completing; it does not establish a reliable executable implementation or a normal return.

## Interface

### Explicit inputs

The represented interface has three inputs and no declared return value.

| Input | Represented width and interpretation | Visible use |
|---|---|---|
| First input | Unsigned 32-bit value | Supplies the first stored byte, contributes to a later byte-read address, and supplies the second stored byte. |
| Second input | Opaque 32-bit value | No explicit use is represented. |
| Third input | Opaque 32-bit value | Passed as the third represented argument to the first indirect call. |

“No explicit use” of the second input does not prove that an indirect callee cannot observe its original register or stack location. Calling conventions and complete machine-level argument use are **documented_only**.

### Ambient inputs

Execution also depends on:

- A global value used in a left-shift calculation.
- An ambient base-register value used to locate a byte.
- An ambient link-register value used to select the first call target.
- Memory containing the bytes read during the routine.
- The availability and behavior of twelve indirect call targets.

The ambient base and link values are not initialized within the represented routine. Treating them as ordinary explicit arguments would change the interface and is not justified.

### Outputs and ownership

There is no represented normal return value. Observable outputs consist of memory changes, dependency invocations, and any effects produced by those dependencies.

No allocation, ownership transfer, retain operation, or release operation is explicitly represented. This does **not** establish that callees avoid such operations. Whether any passed value denotes an owned allocation is **documented_only**.

### Width and address model

The listing represents unsigned arithmetic on the first input at 32-bit width. Its byte reads produce values from 0 through 255; its byte stores retain eight bits.

Address calculations involving the first input therefore use modulo-\(2^{32}\) arithmetic in the represented model. Actual address translation, accessibility, alignment rules, and fault handling are not supplied.

Two left shifts have signedness or reconstruction limitations:

- The initial global-derived value is shifted left by ten.
- A later call result, represented as a signed integer, is shifted left by two.

Their treatment for negative operands, overflow, or discarded high bits is not established as a portable arithmetic contract. Any low-32-bit interpretation of those shifts is conditional on the actual machine semantics.

## Behavior

The following order is established by the represented path. Every later step is conditional on earlier reads, writes, and calls completing.

### Initial state read and first byte store

The routine first reads the global value and forms a value shifted left by ten. It then reads one byte from the address one byte beyond the ambient base-register value.

Call that byte the **initial selector**. Its zero-extended value is between 0 and 255.

The routine writes the low eight bits of the first input to the address:

\[
\text{initial selector} + 17
\]

This is a single-byte destination in the numerical range 17 through 272, inclusive. The represented operation does not add an object base or dereference a pointer table at this stage. The selector byte itself supplies the address contribution.

That distinction is material: interpreting the selector as an index into an ordinary object array would introduce behavior absent from the evidence.

### First group of calls

After the first byte store, the routine invokes the following dependencies in order:

1. **F-000000000001** receives four represented arguments:
   - The zero-extended initial selector.
   - The same selector value again.
   - The third explicit input.
   - The previously computed global-derived shifted value.
2. **F-000000000002** is invoked with no explicit arguments represented. Its opaque 32-bit result is retained.
3. **F-000000000003** receives that retained result as its single represented argument.
4. **F-000000000004** is invoked with no explicit arguments represented.
5. **F-000000000005** is invoked with no explicit arguments represented.
6. **F-000000000006** is invoked with no explicit arguments represented.
7. **F-000000000007** is invoked with no explicit arguments represented. Its integer result is retained for a later call.

The initial selector is read once before this group. The represented path does not reload it between the first byte store and the first call.

The global-derived value is also computed before that store and passed afterward. Thus, under the represented data flow, a write that aliases the global does not retroactively change the already-computed argument.

The callees may change memory or ambient state. No preservation contract is available for them.

### Second byte read and store

After F-000000000007 returns, the routine computes the address:

\[
(2 \times \text{first input} + 1) \bmod 2^{32}
\]

It reads one byte from that address. Call the resulting value the **later selector**.

It then writes one byte to:

\[
\text{later selector} + 17
\]

As with the first store, this destination lies numerically between 17 and 272.

The stored value is:

\[
(\text{first input} \mathbin{\text{logical-right-shift}} 22) \bmod 256
\]

It therefore consists of original input bits 22 through 29. Original bits 30 and 31 are discarded by the final narrowing to eight bits. This is not a store of the input’s most significant byte.

The later selector is read at this point, not before the first group of calls. Consequently, the first byte store or any of the first seven callees can affect the later selector if their writes reach its source address.

There is no represented validation of the calculated read address or either byte-store destination.

### Final group of calls

After the second byte store:

8. **F-000000000008** receives the retained result of F-000000000007 shifted left by two.
9. **F-000000000009** is invoked with no explicit arguments represented.
10. **F-000000000010** is invoked with no explicit arguments represented.
11. **F-000000000011** is invoked with no explicit arguments represented.
12. **F-000000000012** is invoked with no explicit arguments represented.

The second byte store occurs between the production of the retained integer result and its use by F-000000000008. It must not be moved after that call in a behavior-preserving description.

### Terminal behavior

After the twelfth call, the supplied representation ends at an invalid-instruction/control-flow truncation marker rather than a demonstrated ordinary return.

This establishes an evidence boundary, not a dependable runtime error API. Whether actual execution traps, stops, continues through unrecovered instructions, or follows another path is **documented_only**.

## Side effects

### Directly represented memory effects

The routine has three explicit data-memory reads:

1. The global value used for the initial shift.
2. The initial selector byte.
3. The later selector byte.

It has two explicit data-memory writes:

1. Low eight bits of the first input to the initial-selector-derived address.
2. Input bits 22 through 29 to the later-selector-derived address.

Both writes are one byte wide. No multi-byte object layout, structure size, terminator, or length field is established.

### Aliasing and ordering

The two destination addresses may be equal. If so, and execution reaches both stores, the second store occurs later and determines the byte immediately after that store. Later callees may change it again.

The first store can alias the source of the later selector. Earlier callees can also alter that source. The later read must therefore observe memory at its own position in the sequence rather than an assumed entry-time snapshot.

The initial selector’s source may itself be affected by the first store, but the selector already read remains the value used for the first represented call arguments.

### Dependency effects

All other memory changes, I/O, synchronization, resource operations, exceptions, and termination behavior are **documented_only**. Absence of explicit arguments at a call site does not imply a pure or effect-free dependency.

No rollback, cleanup, retry, or error-code handling is represented. If execution stops before the second store, the routine contains no represented mechanism that reverses the first store.

## Dependencies

The following are contract-local opaque function IDs. They distinguish call sites; they are not verified identities supplied by a dependency registry. Distinct IDs do not prove distinct runtime targets.

| Opaque ID | Target-selection evidence | Represented data flow |
|---|---|---|
| F-000000000001 | Ambient link-derived address plus a fixed displacement | Receives the initial selector twice, the third input, and the global-derived shifted value. |
| F-000000000002 | Fixed arithmetic-derived target | Produces an opaque 32-bit value. |
| F-000000000003 | Fixed arithmetic-derived target | Receives the preceding opaque result. |
| F-000000000004 | Symbolic-address-derived target plus a fixed displacement | No explicit arguments or consumed result represented. |
| F-000000000005 | Fixed arithmetic-derived target | No explicit arguments or consumed result represented. |
| F-000000000006 | Fixed arithmetic-derived target | No explicit arguments or consumed result represented. |
| F-000000000007 | Fixed arithmetic-derived target | Produces the integer later shifted left by two. |
| F-000000000008 | Fixed arithmetic-derived target | Receives that shifted integer. |
| F-000000000009 | Fixed arithmetic-derived target | No explicit arguments or consumed result represented. |
| F-000000000010 | Fixed arithmetic-derived target | No explicit arguments or consumed result represented. |
| F-000000000011 | Fixed arithmetic-derived target | No explicit arguments or consumed result represented. |
| F-000000000012 | Fixed arithmetic-derived target | No explicit arguments or consumed result represented. |

The supplied dependency map identifies only F-d4ff07cb00ed itself. Actual dependency identities, signatures, ABI requirements, memory ownership rules, and return guarantees remain **documented_only**.

## Acceptance examples

These examples are conditional static acceptance criteria, not reports of execution. Each assumes that the required memory accesses are valid and every preceding call returns.

### First-store narrowing

Given a first input of `0x123456AB` and an initial selector of `0x20`:

- The first byte store targets address `0x31`.
- It stores `0xAB`.
- F-000000000001 receives `0x20` in both of its first two represented argument positions.

This does not constrain the third input or the global-derived argument.

### Second-store bit selection

Given a first input of `0xFFC00000`:

- The later-selector read address is `0xFF800001` under 32-bit unsigned arithmetic.
- The second stored byte is `0xFF`.

The logical right shift initially produces `0x3FF`; narrowing to one byte discards its upper two bits.

Given a first input of `0xC0000000`, the second stored byte is instead `0x00`. The input’s top two bits alone do not contribute to that byte.

### Address wraparound

Given a first input of `0x80000000`, the later-selector read address is `0x00000001`, not an unbounded integer address above the 32-bit range.

If address one is inaccessible, successful continuation is not established. No fallback read or validation path is represented.

### Shared destination

If both selector reads produce `0x05`, both direct stores target address `0x16`.

Absent intervening or subsequent callee writes to that address, the first store places the input’s low byte there, and the second later replaces it with input bits 22 through 29.

### Callee mutation of the later selector

If F-000000000006 changes the byte at the computed later-selector source address from `0x04` to `0x09`, and no later operation changes it before the read, the second store targets address `0x1A`, not `0x15`.

This follows from the placement of the read after F-000000000007 returns.

### Result shifting

If F-000000000007 returns the represented integer value 3, the second store occurs before F-000000000008 is called with 12.

This example avoids negative operands and overflow; it makes no claim about ambiguous signed-shift cases.

## Uncertainties

The following are **documented_only**:

- Whether the recovered operations faithfully represent executable instructions.
- The actual machine ABI, address-space configuration, and accessibility of the unusually low byte-store addresses.
- The width, signedness, and storage layout of the global before its shift.
- Overflow and negative-operand semantics for the two left shifts.
- The origin and validity of the ambient base and link values.
- All dependency identities and complete calling conventions.
- Callee effects on globals, memory, registers, resources, and control flow.
- Whether any numeric input represents a pointer, handle, encoded field, or unrelated scalar.
- Behavior after the final represented call.
- Fault behavior for inaccessible memory or invalid call targets.
- Concurrency, volatile-access, and memory-ordering requirements.

An initialization or dispatch interpretation is conceivable, but no such intent is adopted as part of this contract.

## Evidence

The sole evidence is the supplied static listing and its accompanying dependency map. The listing represents a three-input routine, two byte stores, twelve indirect calls, and a truncated terminal path. Its reconstruction warnings limit confidence in complete control flow and global layout.

No implementation access, runtime execution, smoke checks, or tests were performed. Acceptance examples are arithmetic and ordering consequences of the represented operations, subject to the stated conditions; they are not observed outcomes.
