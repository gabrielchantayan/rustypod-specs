<!-- Trusted provenance: F-a26ad5dec204; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a26ad5dec204

## Purpose

F-a26ad5dec204 executes an ordered sequence of indirect calls and memory accesses. Its directly visible effects include five calculated 32-bit stores, two byte-routing operations, a global capture followed later by an indirect 32-bit read, an assignment to another global, and two final zero writes. It then attempts an indirect call to address zero.

No application-level purpose is established. The supplied evidence does not establish allocation, initialization, serialization, dispatch, or cleanup semantics.

This contract distinguishes:

- **Established static evidence:** operations, represented widths, argument relationships, and ordering visible in the supplied body.
- **Inferred intent:** possible interpretations not established by those operations.
- **`documented_only`:** behavior that remains unresolved because declarations, dependency implementations, or execution-environment information are absent.

Every described operation is conditional on execution reaching it. There are no visible conditional branches, but a dependency or unsupported memory access can prevent subsequent operations.

## Interface

### Explicit inputs

The represented interface accepts three 32-bit arguments and has no explicit return value.

| Input | Directly established use |
|---|---|
| First argument | No direct use in the supplied body. |
| Second argument | No direct use in the supplied body. |
| Third argument | An unsigned value whose bottom two bits are cleared before adding `0x14` to obtain a store destination. |

Lack of direct use does not establish that the first two arguments are irrelevant to the complete calling convention or to implicitly communicated dependency state.

### Additional incoming state

The body also depends on:

- An incoming signed 32-bit control-address value used to calculate the first call target.
- An inherited unsigned 32-bit **address word**, used for extraction and as a byte-read address basis.
- An inherited 32-bit **initial payload**.
- An **address-bearing global**, whose value is assigned to a signed 32-bit local called the **captured base**.
- A **result global**, assigned the result of a late indirect call.
- An **unresolved output pointer**, used for a final byte write.
- Executable dependency targets and their required implicit state.

The declarations of both globals are absent. Consequently:

- The address-bearing global’s access width, declared type, and conversion into the captured base are unresolved. The receiving local is 32 bits; this does **not** establish a 32-bit global read.
- The result global’s destination width and declared type are unresolved. Its assignment does **not** establish the producing call’s result width or any conversion performed by the assignment.

These role names establish neither object layout nor ownership.

### Outputs and completion

There is no explicit returned value. Observable outputs are dependency invocations and memory effects.

After the final writes, the function attempts a call to address zero with three zero-valued arguments. Reaching the syntactic return requires that this address be callable and that the call return. Neither condition is established.

### Widths and layout

Explicitly represented non-global accesses include 8-bit reads and writes, a 16-bit zero write, and 32-bit word reads and stores. Two dependency results are represented as 64-bit quantities and divided into 32-bit halves.

Several word destinations are four times a signed 32-bit dependency result, plus `0x14`. This arithmetic does not establish an object header, array layout, allocation size, or bounds guarantee.

Endianness is unspecified. Numeric word values and individual byte accesses are described separately; no serialized byte order is inferred.

## Behavior

### Dependency naming and execution order

F-d001 through F-d048 identify the 48 call sites in execution order. They do not imply distinct targets.

“No explicit arguments” means none are shown at that site. It does not establish a complete ABI signature or the absence of implicit inputs.

### Initial stores and forwarding

F-d001 is called without explicit arguments. Its target is the incoming control-address value plus `0xFFE`.

Its represented signed 32-bit result determines the first store destination: four times that result plus `0x14`. The inherited initial payload is stored there as 32 bits. F-d002 is then called with four times the same result.

F-d003, F-d004, and F-d005 follow without explicit arguments.

F-d006 produces a signed 32-bit result that is shifted left by two. The resulting 32-bit value is stored at the destination obtained by clearing the third argument’s bottom two bits and adding `0x14`. F-d007 subsequently receives the shifted value twice.

The destination calculation uses unsigned 32-bit arithmetic, including wraparound in its final addition. This does not provide an unsigned-wraparound guarantee for the separate signed shift producing the payload.

The next calls forward independent 32-bit values:

| Producer | Consumer | Explicit consumer argument |
|---|---|---|
| F-d008 | F-d009 | The represented 32-bit result of F-d008 |
| F-d010 | F-d011 | The represented 32-bit result of F-d010 |

F-d012, F-d013, and F-d014 then execute without explicit arguments. No result from F-d009 or F-d011 is explicitly consumed.

### Extraction from inherited state

F-d015 supplies a signed 32-bit result used to calculate another destination: four times that result plus `0x14`.

At that destination, the body stores the inherited address word shifted right logically by 23. The store is 32 bits wide, and its payload lies between 0 and 511 inclusive.

F-d016 then receives four times the F-d015 result. F-d017, F-d018, and F-d019 follow without explicit arguments.

### Split result and global capture

F-d020 produces a quantity represented as 64 bits.

- Its low half is interpreted as signed 32-bit and shifted left by two, producing the **retained scaled payload**.
- Its high half is interpreted as signed 32-bit and supplies a destination basis.

The body stores the retained scaled payload at four times the high half plus `0x14`.

Only after that store does it evaluate the address-bearing global and assign its value to the signed 32-bit captured base. The source global’s width and the assignment’s conversion semantics remain unresolved.

F-d021 then receives four arguments, in order:

1. The retained scaled payload.
2. The high half from F-d020.
3. The retained scaled payload.
4. The retained scaled payload.

F-d022, F-d023, and F-d024 follow without explicit arguments.

F-d025 supplies a signed 32-bit destination basis. The body stores the retained scaled payload again, at four times this result plus `0x14`. Immediately afterward, it reads a 32-bit unsigned word at the captured base plus `0x14`. This is the **captured data word**.

F-d026 receives four times the F-d025 result. F-d027, F-d028, and F-d029 follow without explicit arguments.

The global capture and indirect word read are distinct observations separated by calls and a store. The body does not reload the address-bearing global for the indirect read. Changes to that global do not themselves replace the retained captured base. Changes to the referenced memory before the indirect read remain observable.

The store after F-d025 can affect that read if their locations overlap. The retained scaled payload is stored exactly twice by the visible body.

### First byte-routing operation

F-d030 produces another quantity represented as 64 bits. Its low half is retained as an unsigned 32-bit **routing word**; its high half supplies later call arguments.

After F-d030, the body:

- Reads an unsigned byte at the inherited address word plus one.
- Adds `0x11` to that byte’s numeric value.
- Uses the sum as an absolute destination address.
- Writes the bottom eight bits of the captured data word there.

The address-word increment uses unsigned 32-bit arithmetic. The routed destination lies between `0x11` and `0x110`, inclusive; it is not relative to the inherited address.

F-d031 follows the write. Its first three arguments are each the high half from F-d030. Its fourth argument is the routing word shifted left by two using unsigned 32-bit arithmetic.

F-d032 supplies a represented 32-bit result passed to F-d033. F-d034, F-d035, and F-d036 follow without explicit arguments.

### Second byte-routing operation

F-d037 supplies a signed 32-bit result retained for the next call.

Before consuming that result, the body derives another routing-byte source from the routing word: clear its top two bits, shift right logically by 21, and add one. This source address lies between 1 and 512 inclusive.

The unsigned byte read there, plus `0x11`, supplies another absolute destination between `0x11` and `0x110`. The body writes bits 22 through 29 of the captured data word to that destination. This is not the captured data word’s top byte.

F-d038 then receives the F-d037 result shifted left by two. F-d039, F-d040, and F-d041 follow without explicit arguments.

The second routing read occurs after the first routed write and intervening calls. It can observe changes caused by them. Neither byte payload requires rereading the captured data word.

### Final calls and writes

F-d042 supplies a signed 32-bit result. F-d043 receives two equal arguments, each that result shifted left by two.

F-d044 follows without explicit arguments. F-d045 supplies a represented 32-bit result passed to F-d046.

F-d047 then executes without explicit arguments. Its result is assigned to the result global. The assignment and its position in the sequence are established; the call-result width, global destination width, and assignment conversion are not.

The body next performs, in order:

1. An 8-bit zero write through the unresolved output pointer.
2. A 16-bit zero write at the routing word with its top two bits cleared, shifted right logically by seven.
3. F-d048: an indirect call to address zero with three zero-valued arguments.

The calculated 16-bit destination ranges from zero through `0x7FFFFF`, inclusive. It need not be even-aligned.

## Side effects

### Body-visible memory accesses

Counts exclude dependency-internal activity and assume all listed operations are reached.

| Access | Count | Established width |
|---|---:|---|
| Calculated word stores | 5 | 32 bits each |
| Address-bearing global evaluation | 1 | Unresolved; value assigned to a signed 32-bit local |
| Read through captured base plus `0x14` | 1 | 32 bits |
| Routing-byte reads | 2 | 8 bits each |
| Routed data-byte writes | 2 | 8 bits each |
| Assignment to result global | 1 | Unresolved |
| Zero write through unresolved output pointer | 1 | 8 bits |
| Calculated final zero write | 1 | 16 bits |

The five calculated 32-bit stores contain:

1. The inherited initial payload.
2. The shifted F-d006 result.
3. The inherited address word shifted right by 23.
4. The retained scaled payload after F-d020.
5. The same retained scaled payload after F-d025.

The result-global assignment is separate and is not classified as a 32-bit store.

### Aliasing and order

No disjointness is established among destinations, globals, routing sources, or dependency-accessible storage.

A calculated store can affect a later global evaluation or indirect read. The first routed write can affect the second routing read. Both routed destinations can coincide. Final writes can overlap earlier effects.

If the routed destinations coincide, the second routed write occurs later and determines that byte immediately after its write, absent concurrent modification. Later dependencies or writes may change it again.

The supplied evidence warns that some global symbols overlap smaller symbols at the same address. It does not establish exact overlap relationships or either global’s width.

The described order is the represented body order. No volatile-access, atomicity, synchronization, or concurrent-memory guarantee is established.

### Ownership and failures

No explicit allocation, release, ownership transfer, or reference-count change is identified. Dependency behavior remains unknown.

Retaining an address does not establish continued storage lifetime. No dependency is assumed to preserve global values, referenced memory, or ownership.

There are no visible validation branches, bounds checks, null checks, alignment checks, retries, or rollback operations. Reached accesses require storage supporting the indicated operation; reached calls require compatible executable targets. The body defines no error result or recovery path. Earlier completed effects have no visible rollback.

## Dependencies

| Call sites | Explicitly consumed result |
|---|---|
| F-d001, F-d006, F-d015, F-d025, F-d037, F-d042 | Represented signed 32-bit value used in calculation or forwarding |
| F-d008, F-d010, F-d032, F-d045 | Represented 32-bit value forwarded to the next indicated consumer |
| F-d020, F-d030 | Represented 64-bit quantity split into 32-bit halves |
| F-d047 | Result assigned directly to the result global; result width unresolved |
| Remaining sites | No explicit result consumed by the body |

F-d001 has an incoming-state-dependent target. F-d048 targets zero. Intermediate targets are expressed through fixed address calculations.

Complete signatures, implicit inputs, memory effects, ownership behavior, and termination are `documented_only`. The represented split results do not establish whether the underlying ABI returns scalar 64-bit values or paired outputs.

The unresolved output pointer indicates incompletely represented incoming or register-derived state. It does not establish a documented pointer return from any particular dependency.

## Acceptance examples

These are conditional static examples, not executed tests. They assume execution reaches the operations and that required calls and memory accesses are supported.

1. **Third-argument destination.** With third argument `0x1003` and F-d006 result 3, the body stores 12 at `0x1014`. F-d007 then receives 12 twice.

2. **Unsigned destination wraparound.** Third argument `0xFFFFFFFF` becomes `0xFFFFFFFC` after clearing its bottom bits. Adding `0x14` produces destination `0x10` in unsigned 32-bit arithmetic. Writability is not implied.

3. **Inherited extraction.** An inherited address word of `0xFFFFFFFF` produces payload `0x000001FF` after F-d015. The payload is stored as 32 bits.

4. **Repeated scaled payload.** If F-d020 has low half 5 and high half 7, the body stores 20 at `0x30`, then captures the address-bearing global’s value into the 32-bit local. F-d021 receives 20, 7, 20, 20. If F-d025 returns 9, the second store of 20 is at `0x38`.

5. **Captured base versus changed global.** If the global assignment produces captured base `0x2000`, a later change to the global does not itself redirect the indirect read from `0x2014`. This example constrains the captured local, not the width or raw representation of the global. Earlier changes to memory at `0x2014` remain observable.

6. **First routed byte.** With captured data word `0xA5B6C7D8` and first routing byte `0x20`, the first routed write places the 8-bit payload `0xD8` at absolute address `0x31`.

7. **Second routed byte.** With routing word `0x00400000`, the second routing-byte source is address 3. If that byte is `0x7F` and the captured data word is `0xA5B6C7D8`, the second routed write places payload `0x96` at `0x90`.

8. **Coincident destinations.** If both routing reads yield `0x20`, both writes target `0x31`. The second extracted payload is written later. Distinct operations do not imply distinct destinations.

9. **Final ordering without a global-width assumption.** With routing word `0x100`, the final 16-bit zero destination is address 2. F-d047’s result is assigned to the result global before the unresolved-pointer byte write, which precedes the 16-bit write. No particular result-global width or stored numeric representation follows from this example.

10. **Interrupted execution.** If F-d031 does not return, the first routed write has occurred, but the second routing operation, result-global assignment, and final zero writes are not reached.

## Uncertainties

The following remain `documented_only`:

- Application-level purpose and complete ABI.
- Origins and validity of inherited state and the unresolved output pointer.
- Dependency implementations, complete signatures, effects, and termination.
- Both global declarations, access widths, layouts, and overlap relationships.
- Conversion from the address-bearing global into the signed 32-bit captured base.
- F-d047’s result width and conversion into the result global.
- Endianness, address mapping, alignment requirements, and execution permissions.
- Whether address zero is callable.
- Machine-level treatment of signed arithmetic outside source-language-defined ranges.

Arithmetic is not uniformly unresolved. Unsigned masks and logical shifts establish the stated extraction ranges. The third-argument destination addition, inherited-address increment, and unsigned shift for F-d031 have represented unsigned 32-bit behavior.

Other expressions involve signed 32-bit multiplication, addition, or left shift. These include several calculated word destinations, the captured-base-plus-`0x14` expression, and multiple scaled call arguments. Negative left-shift operands or overflow do not justify an unconditional source-language wraparound contract.

A low-32-bit machine-word interpretation is plausible but remains **inferred**, not established for those cases. Exact source-language treatment of out-of-range signed conversions and signed-character narrowing is also unresolved; routed payload descriptions identify the represented eight-bit patterns.

## Evidence

This contract uses only the supplied static body. No dependency implementations, global declarations, ABI specification, memory map, or runtime observations were supplied.

Explicit access types establish five calculated 32-bit stores, one indirect 32-bit data read, two routing-byte reads, two routed byte writes, and final 8-bit and 16-bit zero writes. The body also establishes one address-bearing global evaluation and one result-global assignment, with the ordering described above.

The address-bearing global feeds a signed 32-bit local; that local’s width does not establish the global read width. The result-global assignment establishes neither its destination width nor F-d047’s result width.

The masks, shifts, retained values, call arguments, and body order support the remaining operational relationships. Incomplete inherited state and the global-overlap warning limit higher-level interpretation. No runtime tests have been performed.
