<!-- Trusted provenance: F-8a8be221fcaf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8a8be221fcaf

## Purpose

The established behavior is an ordered sequence of dependency calls, four reads from an entry-state-derived address, four writes through an explicit destination argument, and subsequent operations involving two fixed storage locations. A bounded branch adds two calls, a byte write, a halfword write, and local stack assignments.

A higher-level purpose is not established. The evidence does not justify identifying this function as a loader, decoder, allocator, initializer, or cleanup operation.

Both represented paths end where instruction interpretation becomes incomplete. This contract specifies the supported observable prefix, not successful completion, normal return, or an intentional processor trap.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits, unsigned | Its upper four bits become the explicit argument to F-000000000001. |
| Second argument | 32 bits, opaque | No explicit use is visible. |
| Third argument | 32 bits, address-like | Supplies the destination base for writes at byte offsets `0x94`, `0x98`, `0x9C`, and `0xA0`. |

The absence of a visible use of the second argument does not prove machine-level irrelevance. Dependency argument lists and calling conventions are incomplete.

### Additional entry state

Two incoming register-derived values are effective inputs:

- A 32-bit source-related value whose upper byte supplies a memory address.
- A 32-bit link-derived value used to calculate the first dependency target.

Their provenance, validity constraints, and caller obligations are unknown.

### Memory inputs

The represented execution consumes:

- Four 32-bit words from the derived source address.
- The first 32-bit word of storage beginning at `0x08400240`, used as a base address.
- A 32-bit shared word at `0x08400198`, used for an unsigned comparison and, conditionally, as a stored byte value and address offset.

Dependency results and secondary register state also supply values. None of these memory locations is established as immutable or protected from dependency side effects.

### Outputs and completion

No defined return value is established. Neither represented path reaches a normal return.

Observable outputs consist of memory writes and dependency interactions. Behavior beyond the two incomplete instruction endpoints is **documented_only**.

### Preconditions for executing the represented prefix

Each reached dependency must be callable, and every reached access must be valid when performed. In particular:

- The derived source region must support four 32-bit reads at byte offsets `0x00`, `0x04`, `0x08`, and `0x10`.
- The destination must support four 32-bit writes at offsets `0x94` through `0xA0`.
- Addresses `0x08400240` and `0x08400198` must support 32-bit reads.
- The base obtained from `0x08400240` must support a 32-bit write at offset `0x68`.
- The bounded branch additionally requires a byte write at that base plus `0x0F` and a 16-bit write starting at an address from `0x28` through `0x30`.
- Required stack storage and any target-specific alignment constraints must be satisfied.

No visible validation establishes these conditions before use.

## Behavior

### Widths and address interpretation

The visible operations include 32-bit reads and writes, an 8-bit write, a 16-bit write, and a dependency result represented as 64 bits.

The first argument is logically shifted right by 28, producing a value from 0 through 15. The source-related entry value is logically shifted right by 24, producing an address from 0 through 255. This address is used directly: no further base addition or element-size scaling is visible.

Addresses and address-derived values are represented as 32 bits. Address additions are described in that model, not as portable signed-language arithmetic. Actual overflow behavior, access faults, endianness, and alignment behavior are not established by the supplied representation.

### Initial dependency sequence

The source address is derived before the first dependency call. The function then makes seven calls, in this order:

1. F-000000000001 receives the upper four bits of the first argument.
2. F-000000000002 produces a consumed 32-bit result.
3. F-000000000003 receives that result.
4. F-000000000004 produces another consumed 32-bit result.
5. F-000000000005 receives that result.
6. F-000000000006 produces another consumed 32-bit result.
7. F-000000000007 receives that result.

For calls without listed explicit arguments, the evidence does not establish that no meaningful machine state is passed.

The first target is calculated by clearing the low two bits of the incoming link-derived value and adding `0xDDC`. The following six targets are fixed calculated addresses, identified in the dependency table.

A secondary 32-bit register value available after F-000000000007 is consumed later. It is not established as an ordinary return value or as any particular semantic kind of data.

### Source reads

After all seven calls, four 32-bit words are read in the following represented order:

| Order | Source byte offset | Subsequent use |
|---|---:|---|
| 1 | `0x00` | First explicit argument to F-000000000008 |
| 2 | `0x04` | Fourth explicit argument to F-000000000008 |
| 3 | `0x08` | Destination write at offset `0x9C` |
| 4 | `0x10` | Destination write at offset `0xA0`, and a later local stack assignment |

There is no visible source read at offset `0x0C`.

These are low-address accesses because the derived base is at most `0xFF`. Their physical meaning and validity are unresolved. They must not be replaced in the contract by a conventional full-width pointer interpretation.

Although the source address is derived before the calls, its contents are read afterward. Dependency changes to the source region before these reads can therefore affect the consumed values.

### Destination writes

The destination field address is the third argument plus `0x94`. After all four source reads, the function performs these 32-bit writes in order:

| Destination byte offset | Value written |
|---|---|
| `0x94` | Secondary register value available after F-000000000007 |
| `0x98` | Address of the destination field at offset `0x94` |
| `0x9C` | Captured source word from offset `0x08` |
| `0xA0` | Captured source word from offset `0x10` |

The write at offset `0x98` stores an address, not the contents of the field at offset `0x94`.

All source reads precede all four destination writes. If the regions overlap, these writes still use the already-captured source words. No source reread is visible between the destination writes.

### Fixed-storage reads and state-processing call

After the four destination writes, the function reads the first 32-bit word at `0x08400240`. That word supplies a base used for later writes.

It then calls F-000000000008 with four explicit arguments, in order:

1. Captured source word from offset `0x00`.
2. Secondary register value previously written at destination offset `0x94`.
3. Destination field address: the third argument plus `0x94`.
4. Captured source word from offset `0x04`.

After F-000000000008, the function:

1. Reads the 32-bit shared word at `0x08400198`, capturing it in local stack storage.
2. Writes a secondary 32-bit register value available after that call at offset `0x68` from the previously loaded base.
3. Compares the captured shared word with 8, using an unsigned comparison.

The ordering is material:

- The base read at `0x08400240` follows the destination writes and precedes F-000000000008.
- The shared-word read at `0x08400198` follows F-000000000008.
- The write at base offset `0x68` follows that shared-word read and precedes the comparison.

A change by F-000000000008 to the word at `0x08400240` does not cause a visible base reread. A change by that dependency to `0x08400198` can affect the shared value subsequently captured. These statements describe visible dataflow, not a general guarantee that dependencies preserve globals or memory.

If the captured shared word exceeds 8, the represented path reaches its first incomplete instruction endpoint. The bounded branch’s additional calls and stores are not visible on that path. The write at base offset `0x68` has already occurred.

### Branch for shared values from 0 through 8

For a captured shared word no greater than 8, the represented sequence continues:

1. F-000000000009 is called, producing a result represented as 64 bits.
2. The low eight bits of the captured shared word are written at offset `0x0F` from the previously loaded base.
3. A 16-bit stack-address-derived value is written at the address equal to the captured shared word plus `0x28`.
4. A 32-bit local stack slot receives the destination field address logically shifted right by 14.
5. Another 32-bit local stack slot receives the captured source word from offset `0x10`.
6. F-000000000010 is called.
7. The represented path reaches the second incomplete instruction endpoint.

The halfword value in step 3 is the address of the local slot assigned in step 4, plus `0x398`, reduced to its low 16 bits. This is an address-derived value, not a value derived from that slot’s contents. The halfword write precedes initialization of that slot with the shifted destination address.

The explicit arguments to F-000000000010 are:

1. Low 32 bits of the result from F-000000000009.
2. High 32 bits of that result.
3. The 32-bit constant `0x74`.

The shared storage at `0x08400198` is not visibly reread after F-000000000009. A change to that original shared word alone therefore does not replace the captured branch value. Preservation of the captured stack storage against dependency writes is not independently established.

## Side effects

The established direct effects are:

- Seven initial calls.
- Four source-word reads.
- Four destination-state writes.
- A 32-bit read at `0x08400240`.
- The call to F-000000000008.
- A 32-bit read at `0x08400198` and capture in local stack storage.
- A 32-bit write at the loaded base plus `0x68`.
- Conditionally, two additional calls, a byte write, a halfword write, and two additional local stack assignments.

These are represented ordering relationships. They do not establish hardware transaction granularity, atomicity, synchronization, or all machine-generated stack traffic.

No allocation, deallocation, ownership transfer, lock operation, rollback, or cleanup is established. Passing a destination address to F-000000000008 exposes that address to the dependency; it does not establish whether the dependency retains it, owns it, or releases associated memory.

Dependencies may perform additional reads, writes, or other effects. Their effects are not constrained beyond the supplied evidence.

No visible error result, retry, or recovery path is present. Earlier effects are not visibly undone if a later call fails, an access faults, or execution reaches an incomplete endpoint.

## Dependencies

These IDs are local opaque labels for call sites. They are not independently verified dependency identities. Target addresses identify the visible call behavior without establishing dependency semantics.

| Local ID | Represented target | Established interaction |
|---|---|---|
| F-000000000001 | Incoming link-derived value with low two bits cleared, plus `0xDDC` | Receives the upper four bits of the first argument. |
| F-000000000002 | `0x08400A5C` | Produces a consumed 32-bit result. |
| F-000000000003 | `0x08400A60` | Receives that result. |
| F-000000000004 | `0x08400A64` | Produces a consumed 32-bit result. |
| F-000000000005 | `0x08400A68` | Receives that result. |
| F-000000000006 | `0x08400A6C` | Produces a consumed 32-bit result. |
| F-000000000007 | `0x08400A70` | Receives that result; subsequent behavior consumes secondary register state. |
| F-000000000008 | `0x0840011F` | Receives two captured source words, secondary register state, and the destination field address. |
| F-000000000009 | `0x08E97C84` | Called only in the bounded branch; produces a represented 64-bit result. |
| F-000000000010 | `0x084002AD` | Receives the low and high 32-bit result halves, followed by `0x74`. |

Addresses are recorded as represented, including odd target values. Instruction-set state selection, target normalization, calling conventions, failure behavior, and pointer-lifetime contracts remain unresolved.

## Acceptance examples

These are conditional static expectations, not executed tests. Each assumes that execution reaches the relevant operation, preceding calls return, and required accesses are valid. Where dependencies could alter retained storage, the example also assumes they do not alter that storage.

### Upper-bit extraction

For first argument `0xA1234567`, F-000000000001 receives `0xA`.

Changing only the lower 28 bits leaves that explicit argument unchanged. It does not establish equivalence of the entire execution under an incomplete calling convention.

### Source layout

For source-related entry value `0x80000000`, the derived source address is `0x80`. The four reads occur at `0x80`, `0x84`, `0x88`, and `0x90`, after the seven initial calls.

No read at `0x8C` is established.

### Destination self-reference

For destination base `0x20000000`, the write at `0x20000098` stores `0x20000094`, not the contents of `0x20000094`.

### Fixed-storage timing

Suppose the first word at `0x08400240` contains `0x20001000` when read. If F-000000000008 subsequently changes that word to `0x20002000`, the represented write at base offset `0x68` still targets `0x20001068`.

If that call also changes `0x08400198` to 8 before the shared-word read, the captured comparison value is 8.

### Inclusive branch boundaries

For captured shared value 0, the bounded branch writes byte value 0 and uses halfword destination `0x28`.

For captured shared value 8, it writes byte value 8 and uses halfword destination `0x30`.

Both cases include F-000000000009 and F-000000000010, with `0x74` as the latter’s third explicit argument.

### First excluded value and unsigned comparison

For captured value 9, the write at base offset `0x68` occurs before the first incomplete endpoint. Neither bounded-branch call nor its byte or halfword write is visible.

Captured value `0xFFFFFFFF` also exceeds 8; it is not treated as a negative signed value.

### Captured value versus later shared-storage mutation

If the captured value is 3 and F-000000000009 changes only its original shared storage at `0x08400198` to 7, the later byte value remains 3 and the halfword destination remains `0x2B`.

Whether a halfword access at that address is legal is a separate target-dependent condition.

### Result splitting and halfword truncation

If F-000000000009 produces represented result `0x1122334455667788`, F-000000000010 receives `0x55667788`, `0x11223344`, and `0x74`.

If the relevant local slot’s address has low 16 bits `0xFF00`, the stack-address-derived halfword value is `0x0298` after adding `0x398` and retaining only 16 bits.

## Uncertainties

The following remain **documented_only**:

- **Completion:** The incomplete endpoints do not distinguish an intentional trap from interpretation failure or further executable behavior.
- **Calling conventions:** Complete argument lists, secondary-result conventions, preserved state, and the represented 64-bit return convention are unresolved.
- **Entry-state provenance:** No verified caller contract explains the source-related or link-derived inputs.
- **Low-address memory:** Its mapping, purpose, and accessibility are unknown.
- **Object layout:** Accessed offsets do not establish complete structures, allocation boundaries, or ownership.
- **Address arithmetic and target behavior:** Overflow, endianness, alignment faults, and instruction-set interpretation are not fully established.
- **Aliasing and concurrency:** No disjointness, synchronization, or atomicity guarantee is visible.
- **Dependency effects:** Global, pointed-to-memory, and stack preservation cannot be assumed.
- **Stack-address use:** The halfword’s consumer and useful lifetime are unknown.

Populating state before additional setup is a possible intent, not an established requirement.

## Evidence

This contract is based solely on the supplied static representation: visible dataflow, widths, access offsets, fixed storage addresses, target calculations, call order, unsigned comparison, and incomplete control-flow endpoints.

The fixed reads are specifically the first 32-bit word at `0x08400240` and the later 32-bit word at `0x08400198`. The dependency table records the six fixed targets following the entry-derived call and the three subsequent targets.

No implementation access, dependency inspection, runtime execution, or runtime tests have been performed. Acceptance examples express conditional consequences of static evidence, not observed results.
