<!-- Trusted provenance: F-f6ea3d15d23d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f6ea3d15d23d

## Purpose

F-f6ea3d15d23d performs staged preparation of a caller-supplied destination record using a configuration record, a separately supplied selector byte, and two implicit 32-bit seed values. It validates configuration fields, selects a base word, invokes an inspection dependency, and conditionally populates derived fields.

Normal success returns the 32-bit value `0x00000000`. Explicit rejection returns `0xFFFFFFFF`. Rejection is not transactional: preceding writes remain observable.

“Preparation” and “inspection” describe the visible data flow. The application-level purpose, field meanings, and meaning of the final dependency call are not established.

## Interface

### Inputs and incoming state

The visible interface supplies an unused 32-bit argument and a pointer from which one selector byte is read.

Additional incoming state is required:

| Contract name | Role |
|---|---|
| Destination | Base address of mutable destination storage. |
| Configuration | Base address of configuration storage. |
| Seed A | Incoming 32-bit value used for word assignments and low-byte assignments. |
| Seed B | Incoming 32-bit value used for word assignments and low-byte assignments. |

**Byte A** and **Byte B** mean the low eight bits of Seed A and Seed B. Neither seed has an established numeric value; seed assignments must not be described as zeroing or Boolean initialization.

The mechanism supplying these implicit inputs is **documented_only**. The visible interface alone does not establish a complete standalone calling convention.

### Widths and storage requirements

All record offsets below are byte offsets:

- Byte accesses are 8 bits.
- Word accesses are 32 bits.
- Direct destination accesses extend through byte `0x8D`.
- Direct configuration accesses extend through the word occupying `0x50`–`0x53`.
- F-6d8c85350b66 receives the configuration address at offset `0x40`; its access extent is unknown.

The function also accesses opaque global storage: two alternative base-value slots, a slot supplying the counter address, the counter itself, and a slot supplying the final call’s first argument. Base values and counter contents are 32 bits. The representation widths of the counter-address slot and final-argument slot are not independently established.

Byte order, alignment requirements, and address validity are not independently established. Normal completion requires every executed access to be valid and both invoked dependencies to return normally.

### Results and failure boundaries

| Return value | Established meaning |
|---|---|
| `0x00000000` | Execution reached the final return after both dependency calls. |
| `0xFFFFFFFF` | A range check, one-or-two check, or configuration-kind dispatch rejected the input. |

There are no visible null checks, bounds checks, allocation operations, rollback, or exception recovery. Invalid accesses and dependency failures have no established conversion into these return values.

## Behavior

### Ordered interpretation

The following phases occur in order. Reads are performed when their phase reaches them, not as an entry-time snapshot. Each copy reads its source before its corresponding destination write. Multiple copies listed together occur in the stated order, without implying a bulk transfer.

The selector retained from the initial read is distinct from the destination selector byte read later. Likewise, retained arithmetic operands are distinct from destination fields subsequently reread.

### Initial destination preparation

Before any explicit rejection, the function performs these ordered effects:

| Order | Effect |
|---|---|
| 1 | Store Byte B at destination `0x49`, then `0x47`; store Byte A at `0x48`. |
| 2 | Read the selector byte once, retain it for the first dispatch, and store it at destination `0x58`. |
| 3 | Copy configuration words `0x04`, `0x18`, `0x14`, and `0x28` to destination `0x50`, `0x64`, `0x54`, and `0x74`, respectively. |
| 4 | Store Seed A at destination `0x88`, then `0x84`. |
| 5 | Store Byte B at destination `0x8D`. |

These writes occur even when a later validation rejects the configuration.

### Selector-dependent preparation

The initially retained selector controls the following additional effects:

| Selector | Ordered effects |
|---|---|
| `0` or `7` | Store Byte A at destination `0x42`, then `0x43`; copy configuration bytes `0x30`–`0x33` individually, in ascending order, to destination `0x5D`–`0x60`. |
| `1`, `2`, or `5` | Store Byte A at destination `0x42`, then `0x43`. |
| `3` | Store Byte A at destination `0x42`, then `0x43`; copy configuration bytes `0x2C`–`0x2F` individually, in ascending order, to destination `0x59`–`0x5C`. |
| `4` | Copy configuration byte `0x10` to destination `0x42`, then configuration byte `0x11` to destination `0x43`. |
| Any other byte, including `6` | No selector-specific writes. |

This dispatch never explicitly rejects a selector. Unmentioned fields retain their prior contents unless overlap or another effect changes them.

### Range-related validation

The function reads configuration word `0x08` as unsigned, then writes Byte A to destination `0x40`.

If the retained value is at most 8, this phase does not write destination word `0x3C`.

If it exceeds 8, the function reads configuration word `0x08` again:

- A second value greater than 16 causes immediate rejection.
- Any second value at most 16 causes Seed B to be written to destination `0x3C`.

For stable configuration storage, values 0–8 preserve the prior destination word, values 9–16 replace it with Seed B, and larger unsigned values reject.

The reads are distinct, with the destination-byte write between them. An initial value above 8 followed by a second value below 9 still takes the Seed B assignment.

### One-or-two validation

Configuration word `0x0C` is first read and compared with 1.

- If equal, Byte B is written to destination `0x41`.
- Otherwise, Byte A is written there, followed by a second read of configuration `0x0C` and comparison with 2.
  - A mismatch rejects immediately.
  - A match causes destination word `0x3C` to be read, incremented by one, and written back.

Thus stable configuration values 1 and 2 pass; all others reject. The equality comparisons concern exact 32-bit values.

The increment uses the current destination word. When the preceding range phase left that word untouched, its prior contents are required input state.

After either accepted branch, configuration word `0x0C` is read again and copied to destination `0x14`. This copied value need not equal a previously validated value if storage changes between reads.

### Common assignments and configuration-kind dispatch

After validation, the ordered assignments are:

| Order | Assignment |
|---|---|
| 1 | Seed A to destination words `0x08`, then `0x0C`. |
| 2 | Byte A to destination bytes `0x05`, then `0x04`. |
| 3 | Seed A to destination word `0x4C`. |
| 4 | Byte B to destination byte `0x45`. |
| 5 | Byte A to destination byte `0x46`. |

The function then reads configuration byte `0x00` to select a base-word action:

| Configuration kind | Action |
|---|---|
| `0`, `3`, or `5` | Write Seed A to destination `0x78`. |
| `1` | Read a 32-bit value from the first opaque global base slot, then write that value to destination `0x78`. |
| `7` | Read a 32-bit value from a distinct opaque global base slot, then write that value to destination `0x78`. |
| Any other byte | Reject. |

The kind-1 and kind-7 actions are **global-value reads**, not established constant selections. Each occurs after the preceding destination writes and before the write to destination `0x78`. The slots are distinct source roles; their current values may be numerically equal.

Unsupported kinds reject after all preceding assignments. Neither dependency is called on that path.

### Inspection and conditional field derivation

For a supported kind, F-6d8c85350b66 is called once with the address of configuration offset `0x40`. Its result is tested only for equality with zero.

**Zero result.** The function writes Byte A to destination `0x28`, then Seed A to destination words `0x2C`, `0x30`, `0x34`, `0x38`, `0x7C`, and `0x80`, in that order. These are seed assignments, not established zero assignments.

**Nonzero result.** The ordered effects are:

1. Write Byte B to destination `0x28`.
2. Copy configuration word `0x40` to destination `0x2C`.
3. Read configuration word `0x44`, retain its value, and write it to destination `0x30`.
4. Read configuration word `0x48`, retain its value, and write it to destination `0x34`.
5. Read configuration word `0x50`, retain its value, and write it to destination `0x38`.
6. Read destination byte `0x58`. If it equals 7, overwrite destination words `0x30`, `0x34`, and `0x38`, in that order, with their corresponding retained source values shifted left by one bit.
7. Read destination word `0x78` as an unsigned base and retain it.
8. Read destination word `0x30`, add the retained base, and write the result to destination `0x7C`.
9. Read destination word `0x34` as unsigned for comparison with the retained base.
10. If that comparison value exceeds the base, read destination word `0x34` **again** and subtract the retained base from this second value. Otherwise, select numeric zero without that second read.
11. Write the selected result to destination `0x80`.

The final two reads of destination `0x34` must remain separate. The comparison uses the first value; subtraction uses the second. The retained base is not reread between them.

The selector read in this phase comes from destination storage, not from the initial retained selector. Inspection-dependency effects or overlapping storage may therefore alter whether shifting occurs. Similarly, the arithmetic base is read from destination storage after that dependency returns; it is not necessarily equal to the earlier seed or global value written there.

### Successful completion and global access order

After either inspection-result branch, successful completion has this order:

1. Read the opaque global slot supplying the counter address, and retain that address.
2. Write Byte A to destination `0x8C`.
3. Read the 32-bit counter at the retained address, increment it, and write it back to that address.
4. Read the opaque global value supplying the final call’s first argument.
5. Invoke F-266a29fc8dc5 with that freshly read value and a zero-valued second argument.
6. Return `0x00000000`.

The final-argument read occurs **after the counter write**, not before it or at function entry. Consequently, overlap with the counter or preceding destination writes can affect the argument read. The counter address, by contrast, is obtained before the destination `0x8C` write and retained for the counter update.

The final dependency’s result, if any, does not determine the return value. Normal return from that dependency is still necessary to reach the success return.

### Arithmetic boundaries

All stored arithmetic results occupy 32 bits.

- The addition used for destination `0x7C` has an unsigned 32-bit operand and retains the low 32 bits.
- The subtraction used for destination `0x80` is unsigned 32-bit subtraction. A second-read value below the retained base wraps modulo \(2^{32}\).
- The successful comparison alone does **not** establish that subtraction cannot underflow: it constrains the first read, not the second.
- If destination `0x34` remains unchanged between those reads, the successful strict comparison establishes a positive, non-underflowing difference.
- Destination `0x3C` increment, counter increment, and left shifts have no visible overflow checks.

Exact exceptional or overflow behavior for the signed-form increments and shifts is **documented_only**. The supplied representation does not independently establish target instruction behavior for those cases. Unbounded mathematical arithmetic must not be substituted.

## Side effects

Direct effects include destination mutation, reads of opaque global values, and a counter read-modify-write on the successful path.

There is no visible allocation, release, or ownership transfer. This does not establish dependency ownership behavior. In particular, passing the interior configuration address to F-6d8c85350b66 does not prove that the dependency merely reads it, preserves it, or refrains from retaining or releasing associated storage.

Every explicit rejection preserves earlier effects. No rollback is visible.

No synchronization or atomic counter operation is established. Thread safety, reentrancy, and behavior under unsynchronized concurrent mutation are not guaranteed.

Disjoint and stable storage gives the straightforward field interpretation, but non-overlap is not checked or established as a precondition. Where overlapping storage is valid, the specified read/write order matters. Where storage can change between accesses, distinct reads must not be merged; this observation does not itself establish that concurrent mutation is valid on the target.

Neither dependency is assumed to preserve globals, destination fields, configuration fields, or reachable memory. In particular, the final dependency executes after all direct destination writes and may change the state observed at return.

## Dependencies

| Opaque function ID | Invocation and result use | Unresolved contract |
|---|---|---|
| F-6d8c85350b66 | Once after a supported kind, with configuration address `+0x40`. Zero selects seed assignments; any nonzero result selects copying and derivation. | Access extent, mutations, global effects, retained addresses, ownership, failures, and termination. |
| F-266a29fc8dc5 | Once after counter update and final-argument global read, with that argument and numeric zero. Its result is unused. | Argument meaning and width, memory effects, ownership, failure signaling, and termination. |

The opaque global slots and counter are additional state dependencies, not function calls. Their locations, values, and relationships to other storage are unknown.

## Acceptance examples

These are conditional static examples, not executed tests. Unless an example explicitly varies storage, assume valid, disjoint, stable storage and normally returning dependencies that do not mutate relevant state. Success examples also require all earlier checks to pass and a supported configuration kind.

1. **Range rejection retains preparation.**  
   Configuration `0x08` equal to 17 produces `0xFFFFFFFF` after the destination `0x40` write. Initial and selector-specific writes have occurred. Neither dependency runs.

2. **One-or-two rejection retains its marker.**  
   Configuration `0x08` equal to 9 and `0x0C` equal to 3 cause destination `0x3C` to receive Seed B and destination `0x41` to receive Byte A before rejection.

3. **Low range preserves an input dependency.**  
   Configuration `0x08` equal to 8, `0x0C` equal to 2, and initial destination `0x3C` equal to 10 yield 11 at that destination word.

4. **Selector and kind are independent.**  
   Selector 4 copies configuration bytes `0x10` and `0x11` into destination `0x42` and `0x43`. Configuration kind 4 nevertheless rejects later.

5. **Unknown selectors need not reject.**  
   Selector 255 performs no selector-specific writes but can reach success when later checks pass.

6. **A zero inspection result uses seeds.**  
   The destination marker at `0x28` receives Byte A, and the six specified words receive Seed A. Counter update and the final call still occur.

7. **Selector 7 shifts before deriving.**  
   With destination `0x58` still 7, nonzero inspection result, configuration words 10, 3, and 9 at `0x44`, `0x48`, and `0x50`, and retained base 8, destination `0x30`, `0x34`, and `0x38` become 20, 6, and 18. Destination `0x7C` becomes 28; destination `0x80` becomes zero.

8. **Stable subtraction uses a strict comparison.**  
   Base 8 and both reads of destination `0x34` equal to 13 produce 5 at `0x80`. A comparison value equal to 8 selects zero and performs no subtraction read.

9. **Changing subtraction input can wrap.**  
   Conditional on a valid execution observing base 8, comparison value 13, and second-read value 3, destination `0x80` receives `0xFFFFFFFB`. The first read permits subtraction; it does not constrain the second value.

10. **Base selection observes current global state.**  
    For kind 1, if its global base slot contains 12 when read, the direct write to destination `0x78` is 12. An earlier value of that slot is irrelevant. Kind 7 reads its distinct slot instead.

11. **Final argument observes the counter update.**  
    Conditional on the final-argument slot aliasing the counter word and containing 4 before its non-overflowing increment, the final dependency receives 5 as its first argument. The argument is read after the update.

## Uncertainties

The following remain **documented_only**:

- Complete calling convention and implicit-input mechanism.
- Application meanings, units, and intended lifecycle of record fields.
- Actual seed values and opaque global values or addresses.
- Representation widths of the counter-address and final-argument slots.
- Dependency preservation, ownership, retention, failure, and termination contracts.
- Endianness, alignment behavior, and signed-form overflow or shift behavior.
- Whether overlap or changing storage occurs in valid callers.
- Whether the overall operation is initialization, registration, activation, or something else.

These unknowns do not erase established access ordering. Distinct reads, retained operands, and explicit writes remain part of the static contract.

## Evidence

This contract uses only the supplied static representation of F-f6ea3d15d23d.

Established evidence includes access widths and offsets, ordered direct reads and writes, repeated reads, validation branches, explicit return values, dependency arguments, and use of dependency results. It specifically establishes separate comparison and subtraction reads at destination `0x34`, conditional global base-value reads, the counter-address read before destination `0x8C`, and the final-argument global read after counter update.

Application intent, dependency preservation, ownership, runtime validity, and unresolved target arithmetic behavior are not established. No runtime tests have been performed.
