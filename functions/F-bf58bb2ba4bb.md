<!-- Trusted provenance: F-bf58bb2ba4bb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bf58bb2ba4bb

## Purpose

F-bf58bb2ba4bb writes a caller-supplied destination using an owner reference, configuration fields, temporary geometry, and results from opaque dependencies. It stores an associated address, selects destination geometry using a candidate extent, supplies two four-byte appearance inputs, optionally requests movement of an external object, and performs final owner and temporary-context calls.

Allocation, text measurement, alignment, appearance, and cleanup are inferred roles, not established dependency contracts. In particular, the call to F-8157a180d901 with argument eight does **not** establish a fresh allocation or an eight-byte allocation size. The visible function only uses its returned address for two 32-bit writes and stores that address in the destination.

This contract describes static evidence. No runtime tests have been performed.

## Interface

### Inputs and return value

The function accepts:

1. A destination address, used for direct writes and passed to dependencies.
2. A 32-bit owner reference, initially stored in the destination and later used as an address.
3. A configuration address, read at offsets through `0x4B`.

On the visible normal-return path, it returns the original destination address. No separate success status or error code is produced.

The supplied representation uses 32-bit integers and address values, signed 16-bit configuration components, and individual bytes. Byte order and the complete calling convention are not independently established.

### Destination layout

| Offset | Directly accessed width | Role in this function |
|---|---:|---|
| `0x00` | 32 bits | Owner reference |
| `0x04` | 32 bits | Associated address returned by F-8157a180d901 |
| `0x08` | 32 bits | First geometry component; later used in positioning |
| `0x0C` | 32 bits | Second geometry component |
| `0x10` | 32 bits | Third geometry component |
| `0x14` | 32 bits | Fourth geometry component |
| `0x18` | 32 bits | Value read from configuration `0x14` |
| `0x1C` | 32 bits | Sum of configuration values at `0x14` and `0x18` |
| `0x20` | 8 bits | Mode-like value |
| `0x21` | Dependency-defined | First appearance destination |
| `0x25` | Dependency-defined | Second appearance destination |

The last two addresses are passed to F-35c2e4297234 with four-byte input storage. The function does not directly establish that dependency’s write width. These offsets therefore do not establish the complete required destination capacity.

### Configuration layout

| Offset | Width and interpretation | Use |
|---|---|---|
| `0x0C` | 32 bits | Third argument to F-e365af86ca86 |
| `0x10` | 32 bits | Second argument to F-e365af86ca86 |
| `0x14` | Signed 32 bits | Destination copy and separate addition input |
| `0x18` | Signed 32 bits | Other addition input |
| `0x21`, `0x23`, `0x25`, `0x27` | Signed 16 bits each | First four-component temporary |
| `0x3C` | Unsigned byte in the displayed expression | Mode selection |
| `0x3D` | Character compared with `−1` and `1` | Conditional geometry selection |
| `0x44`, `0x46`, `0x48`, `0x4A` | Signed 16 bits each | Initial working geometry |

The first group of 16-bit fields has odd byte offsets. No alignment check is visible.

### Preconditions, errors, and ownership

Normal completion requires accessible storage for every direct read and write actually reached, plus whatever prerequisites the dependencies impose. The associated address must support the two immediate 32-bit writes. Its contents and accessibility must also support subsequent reached operations.

There are no visible checks for null input addresses, an unusable returned address, record length, geometry validity, arithmetic overflow, or dependency failure. No rollback or explicit failure return is shown.

The caller supplies the destination. The function does not establish who owns the associated address, whether it is fresh, whether it aliases existing storage, or who must release it. Allocation is an inferred purpose of F-8157a180d901 only. An existing destination value at `0x04` is overwritten without a visible release call directed at that prior value.

Input disjointness is not established. If destination, configuration, owner, returned storage, or shared data overlap, earlier writes can affect later reads.

## Behavior

### 1. Initial writes and associated address

The function first writes, in order:

1. The supplied owner reference to destination `0x00`.
2. Zero to destination `0x04`.
3. Zero to destination `0x18`.
4. Zero to destination `0x1C`.
5. A zero byte to destination `0x20`.

It calls F-8157a180d901 with the integer argument `8`. Through the returned address, it writes an opaque global 32-bit value at offset zero, then a zero 32-bit value at offset four. It stores the returned address at destination `0x04`.

These accesses establish an eight-byte direct write footprint through that address, not the size or origin of an allocation.

F-e365af86ca86 then receives:

- The returned address.
- The configuration word at `0x10`.
- The configuration word at `0x0C`.

The argument positions are established; the relative evaluation order of the two configuration reads is not. Initialization or updating of the associated storage is plausible intent, but no such effect is guaranteed by this caller.

### 2. Geometry preparation

The four signed 16-bit fields at configuration offsets `0x21`, `0x23`, `0x25`, and `0x27` are read in that order and sign-extended into four 32-bit temporary components.

A separate bounds-like temporary receives four components:

- Zero.
- Zero.
- Either 240 or 480.
- Either 320 or 640.

Selection uses bit mask `0x04000000` in the 32-bit word at offset `0x48` of the owner currently referenced by destination `0x00`. A clear bit selects `(0, 0, 240, 320)`; a set bit selects `(0, 0, 480, 640)`. The owner reference is read after F-e365af86ca86, not captured permanently from entry.

The signed 16-bit configuration fields at `0x44`, `0x46`, `0x48`, and `0x4A` are then read in order and sign-extended into the working geometry.

F-a0383c04b771 receives the destination and addresses of the working geometry, bounds-like temporary, and first temporary, in that order. F-e57b05520b65 follows, receiving the working geometry as both its first and second arguments and the bounds-like temporary as its third.

Later behavior uses the working geometry after these calls. Neither preservation of the temporaries nor a particular transformation is established. The aliased first and second arguments to F-e57b05520b65 are part of the observable interface.

### 3. Range-like values and mode

The function reads configuration `0x14` and stores it at destination `0x18`. It separately reads configuration `0x14` again and configuration `0x18`, adds those signed 32-bit values, and stores the result at destination `0x1C`. The relative order of the addition’s operand reads is not established.

Mode selection first reads an unsigned byte at configuration `0x3C` and compares it with `0xFFFFFFFF`. Under the displayed unsigned-byte interpretation, that equality cannot hold.

The alternative path reads the byte again:

- If this second read equals `1`, destination `0x20` receives `1`.
- Otherwise, destination `0x20` receives `0`.

The assignment of mode `2` is unreachable under that interpretation. A signed sentinel may have been intended, but is not established. For stable configuration storage, byte `0xFF` therefore selects zero, not two. The two visible byte reads must not be collapsed into one when describing potentially changing shared storage.

### 4. Context, descriptor, and candidate extent

The next calls occur in this order:

1. F-a4c26c2b4842 receives temporary context storage.
2. F-59c1430f85c2 receives temporary descriptor storage and the literal `Helvetica`.
3. F-8e04864cbbaa receives the context and descriptor addresses.
4. F-e84807520bd9 receives the working geometry and numeric arguments `9` and `0`.

The context is represented by a 68-byte local region. That declaration does not bound dependency accesses. No visible caller writes initialize the context before the first call; whether F-a4c26c2b4842 initializes it is unknown. Likewise, descriptor construction and font selection are inferred roles, not guaranteed effects.

After the fourth call, the function calculates the working fourth component minus its second component. It then:

- Reads the destination’s current associated address, reads the 32-bit word at offset four through it, and passes that word to F-d8cbf9aa79a9, obtaining a 16-bit result.
- Reads the destination’s associated address again and passes it to F-ecd1af0519f9, obtaining a 32-bit result.
- Calls F-33ade88db1b2 with the context, the 32-bit result, the 16-bit result, the previously calculated component difference, and zero.

The signed 32-bit result of F-33ade88db1b2 is the candidate extent. Text measurement is inferred from this call sequence and the literal; return-value meanings and error conventions remain unknown. The associated address need not remain unchanged across the calls.

### 5. Destination geometry selection

The candidate extent is compared strictly, as a signed 32-bit value, with working third component minus working first component.

If the candidate is smaller, the destination geometry is written in component order as:

- Working first component.
- Working second component.
- Working first component plus candidate extent.
- Working fourth component.

Only this branch reads configuration `0x3D`.

If the first character read compares equal to `−1`, the destination’s first component becomes working third component minus candidate extent. Its third component is then written using that same subtraction expression plus the candidate extent. This is consistent with right alignment.

Otherwise, the character is read again. If that read equals `1`, F-cca11044abb5 receives the destination geometry address and working geometry address. Its transformation is unknown; centering is not established. If the second read is not `1`, the preceding geometry writes remain the caller’s selected geometry.

If the candidate is equal to or larger than the working span, all four working components are copied to the destination in order. The alignment field is not read.

Negative candidates and negative spans are not rejected. All participate in the same signed comparison and arithmetic.

F-e214f63ae031 then receives the destination and a 32-bit temporary value with no visible direct assignment by this function. Whether an earlier dependency writes that value through supplied temporary storage is unresolved.

### 6. Appearance inputs and global indirection

Four successive byte assignments form the first appearance input:

`00 00 00 CC`

F-35c2e4297234 receives destination address `0x21` and the address of that temporary.

After the call returns, the same temporary is overwritten by four successive indirect reads. Each assignment separately evaluates an opaque global pointer:

1. Evaluate the global pointer and read byte offset zero through that value.
2. Evaluate the global pointer again and read byte offset one.
3. Evaluate it again and read byte offset two.
4. Evaluate it again and read byte offset three.

The caller does not capture a single base address for these four reads. If shared state changes between evaluations, the four bytes can come from different bases. The byte values themselves are also read successively rather than as an atomic four-byte snapshot.

F-35c2e4297234 is then called with destination address `0x25` and the reused temporary.

The first sequence may represent a color, but channel order and interpretation are unknown. Neither call establishes copying, conversion, retention, or a four-byte destination write. If the first call retains the input address, it may observe subsequent overwrites; no preservation promise is supplied.

### 7. Optional external positioning

The function reads the byte at offset `0xA8` of the destination’s current owner. If it is zero, positioning is skipped. Otherwise, F-ffe0d92c57b7 is called; a zero result also skips positioning.

For a nonzero returned address, the function reads signed 32-bit fields at offsets `0x20`, `0x24`, and `0x28`, in that order. It then calls F-7e4f952e879e, followed by F-e10d3ae06e3b.

The proposed first position component is the captured field at `0x24` plus 35. The initial second component is destination geometry component one, read after both calls.

The horizontal margin is zero if F-e10d3ae06e3b returned exactly `1`; otherwise it is ten. The upper limit is always `480 − margin`, independently of the earlier owner flag.

Using the captured width—field `0x28` minus field `0x20`—the function first corrects an excessive far edge, then applies a lower bound:

- If initial second component plus width exceeds the upper limit, subtract the excess from the second component.
- If the resulting component is below the margin, replace it with the margin.

There is no resizing or second upper-bound correction. An oversized width can therefore remain beyond the upper limit.

The current external position at `0x30` is then read. If it differs from the proposed first component, F-a19ace379809 is called without requiring a read at `0x34`. Otherwise, `0x34` is read and compared with the proposed second component. The call is omitted only when both components match.

The proposal storage reuses earlier descriptor storage. No dependency contract establishes that previously passed addresses were not retained.

### 8. Completion

F-92bea7c30bd6 receives the destination’s current owner reference. F-282eebd7c413 then receives the temporary context address.

The function returns the destination address. The last context call suggests cleanup, but release or destruction is not established. No separate descriptor-release operation is visible.

## Side effects

Direct effects include destination writes, two 32-bit writes through the returned associated address, reads of configuration and shared state, and writes to temporary storage.

Observed calls include conditional external positioning and the final owner and context operations. These calls do not by themselves guarantee allocation, initialization, movement, resource release, or preservation of any state.

Later reads must remain later reads: owner references, associated addresses, destination geometry, global appearance-pointer evaluations, and external current-position fields are not entry-time snapshots.

Dependencies may mutate reachable storage or globals, retain passed addresses, or alter resource lifetimes; their contracts are unavailable. Partial state remains possible if execution fails after earlier writes. No transactional behavior is established.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-8157a180d901 | Receives `8`; returned address is written through and stored |
| F-e365af86ca86 | Receives associated address and configuration words `0x10`, `0x0C` |
| F-a0383c04b771 | Receives destination and three temporary addresses |
| F-e57b05520b65 | Receives aliased working-geometry arguments and bounds storage |
| F-a4c26c2b4842 | Receives context storage; initialization is not established |
| F-59c1430f85c2 | Receives descriptor storage and `Helvetica` |
| F-8e04864cbbaa | Receives context and descriptor addresses |
| F-e84807520bd9 | Receives working geometry, `9`, and `0` |
| F-d8cbf9aa79a9 | Receives associated-storage word; supplies a 16-bit result |
| F-ecd1af0519f9 | Receives current associated address; supplies a 32-bit result |
| F-33ade88db1b2 | Supplies the signed candidate extent |
| F-cca11044abb5 | Conditional geometry call for alignment comparison equal to `1` |
| F-e214f63ae031 | Receives destination and unresolved 32-bit temporary value |
| F-35c2e4297234 | Receives appearance destination and four-byte temporary address |
| F-ffe0d92c57b7 | Supplies an optional external address |
| F-7e4f952e879e | Called before margin-result lookup |
| F-e10d3ae06e3b | Result selects margin by equality to `1` |
| F-a19ace379809 | Receives external address and changed two-component proposal |
| F-92bea7c30bd6 | Receives current owner reference |
| F-282eebd7c413 | Receives context storage at completion |

## Acceptance examples

These are conditional static expectations, not executed tests. Numeric examples assume representable arithmetic and the stated dependency results.

1. **Ordinary narrow geometry.** With post-dependency working geometry `(10, 20, 110, 60)`, candidate 30, and alignment reads selecting neither special case, the writes before F-e214f63ae031 produce `(10, 20, 40, 60)`.

2. **Right-aligned narrow geometry.** With the same geometry and candidate, an alignment comparison equal to `−1` produces `(80, 20, 110, 60)` before the subsequent destination call.

3. **Equal candidate and span.** Candidate 100 selects the complete working geometry above. The alignment byte is not read and F-cca11044abb5 is not called.

4. **Unsigned mode boundary.** Stable configuration byte `0xFF` selects mode zero under the displayed unsigned-byte interpretation. Stable byte `0x01` selects mode one. Neither establishes reachability of mode two.

5. **Returned storage need not be fresh.** If F-8157a180d901 returns an already-existing writable address, the visible caller still writes the opaque word and zero there and stores the address. Freshness is not an acceptance requirement established by this body.

6. **Changing appearance pointer.** If successive global-pointer evaluations yield bases A, B, C, and D, the second appearance input receives bytes from A at offset zero, B at offset one, C at offset two, and D at offset three.

7. **Right-edge correction.** With margin ten, captured width 100, and initial horizontal component 450, the proposed horizontal component becomes 370, giving a far edge of 470.

8. **Oversized width.** With margin ten, width 500, and initial horizontal component zero, upper correction produces `−30`; lower correction then produces ten. The far edge becomes 510 without another upper correction.

9. **Unchanged external position.** If both current position fields equal the proposal, F-a19ace379809 is omitted. The final owner and context calls still occur on normal completion.

## Uncertainties

- **documented_only — Allocation and ownership:** freshness, allocation mechanism, capacity, ownership transfer, and release obligations for the associated address are unknown.
- **documented_only — Dependency effects:** initialization, transformations, write footprints, retention, global mutation, failures, and resource release are unavailable.
- **documented_only — Arithmetic overflow:** signed 32-bit additions and subtractions are visible; wrapping, trapping, and other overflow semantics are not established. Algebraic simplifications are unsafe outside representable cases.
- **documented_only — Mode discrepancy:** the visible mode-two assignment conflicts with the unsigned-byte comparison.
- **documented_only — Character interpretation:** the alignment branch compares a character with `−1`; the underlying character signedness and exact byte correspondence require additional evidence.
- **documented_only — Unassigned temporary:** the value passed to F-e214f63ae031 has no visible direct initialization.
- **documented_only — Geometry and appearance meaning:** axis conventions, helper transformations, channel order, and color interpretation remain inferred.
- **documented_only — Aliasing and concurrency:** no synchronization, disjointness, common-base guarantee for repeated pointer reads, or atomic multi-field snapshot is established.

## Evidence

The supplied static representation supports the explicit widths, offsets, access sequences, call arguments, branch conditions, and normal return described here. It establishes the repeated global-pointer evaluations for the second appearance input, but not a single captured base.

No dependency implementation was supplied. Allocation, context initialization, associated-object initialization, measurement, movement, and cleanup therefore remain inferred roles where not directly implemented by this function.

No runtime execution or behavioral testing was performed.
