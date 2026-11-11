<!-- Trusted provenance: F-cb437ed258ac; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cb437ed258ac

## Purpose

F-cb437ed258ac partially initializes a caller-supplied state region using a descriptor, a selector, and additional effective entry-state inputs. It checks two descriptor fields, selects a state value from a separate descriptor tag, and invokes F-6d8c85350b66 to choose between two initialization paths. On normal completion, it increments a shared counter, invokes F-266a29fc8dc5, and returns a 32-bit zero.

Initialization is not transactional. All explicit rejection paths occur after state writes, and no rollback is visible.

Configuration or setup is an inferred purpose. The application meanings of the fields, selectors, external values, and counter are not established. In particular, the effective entry-state values must not be assumed to be zero or one.

## Interface

### Explicit and effective inputs

The visible interface has two 32-bit arguments:

| Input | Established use |
|---|---|
| First argument | No use is visible in the supplied body. |
| Selector | Its full 32-bit value selects an early initialization branch; its low eight bits are also stored in state. |

Four additional values are consumed without their origins being established:

| Effective input | Role |
|---|---|
| State base | Address of the destination state region. |
| Descriptor base | Address of the source descriptor region. |
| Seed A | A 32-bit value used for full-word writes and low-byte writes. |
| Seed B | A 32-bit value used for one conditional full-word write and several low-byte writes. |

These are effective inputs, not established ordinary arguments. Their calling-convention placement and any preceding setup remain **documented_only**.

Throughout this contract, a **word** is 32 bits, a **byte** is eight bits, and offsets are byte offsets from the indicated base. **A-byte** and **B-byte** mean the low eight bits of Seed A and Seed B.

### Return values

| 32-bit result | Meaning |
|---|---|
| `0x00000000` | Execution completed the success-path mutations and final dependency call normally. |
| `0xffffffff` | Execution reached an explicit width, multiplicity, or descriptor-tag rejection. |

The result of F-266a29fc8dc5 is not inspected. A zero return from this routine therefore does not establish success according to that dependency’s own semantics.

Faults, non-returning dependencies, and behavior outside established arithmetic conditions are not assigned additional return values.

### Memory and ownership requirements

Every executed access requires appropriately readable or writable memory. Direct state accesses extend through State+`0x8d`; direct descriptor accesses extend through Descriptor+`0x53`. These are access bounds, not evidence of allocation sizes or contiguous ownership. F-6d8c85350b66 receives Descriptor+`0x40`, but its access footprint is unknown.

Preexisting state can be significant. In particular, an accepted multiplicity-two path reads and increments State+`0x3c`, which the width stage may have left unchanged.

No allocation, release, or ownership-transfer operation is explicit in this body. Neither dependency may be assumed to borrow, retain, own, or release the memory it can access.

## Behavior

### Entry writes and selector-dependent fields

Before validation, the routine stores the selector’s low byte at State+`0x58`. It then copies descriptor words in this order:

| Source | Destination |
|---|---|
| Descriptor+`0x04` | State+`0x50` |
| Descriptor+`0x18` | State+`0x64` |
| Descriptor+`0x14` | State+`0x54` |
| Descriptor+`0x28` | State+`0x74` |

Each source word is read for its corresponding copy; no descriptor snapshot is established. These copies are followed by word writes of Seed A to State+`0x88`, then State+`0x84`, and a byte write of B-byte to State+`0x8d`.

The full selector then determines the following additional byte writes:

| Selector | Ordered effects |
|---|---|
| 0 or 7 | Write A-byte to State+`0x42`, then `0x43`; copy Descriptor+`0x30` through `0x33` to State+`0x5d` through `0x60` in increasing offset order. |
| 1, 2, or 5 | Write A-byte to State+`0x42`, then `0x43`. |
| 3 | Write A-byte to State+`0x42`, then `0x43`; copy Descriptor+`0x2c` through `0x2f` to State+`0x59` through `0x5c` in increasing offset order. |
| 4 | Copy the byte at Descriptor+`0x10` to State+`0x42`, then the byte at Descriptor+`0x11` to State+`0x43`. |
| 6 or any other value | No additional writes at this stage. |

An unsupported selector is not itself a rejection condition. Fields omitted by the selected branch retain their prior contents unless other accesses alias them or external mutation occurs.

### Width validation

The routine reads Descriptor+`0x08` as an unsigned word, then writes A-byte to State+`0x40`.

If the captured width is at most eight, State+`0x3c` is not written by this stage. Otherwise, Descriptor+`0x08` is read again:

- A second value greater than sixteen causes immediate rejection.
- Any other second value causes Seed B to be written as a word to State+`0x3c`.

There is no visible lower-bound check. A stable width of zero passes.

The second read is conditional and distinct from the first. For mutable descriptor memory, a first value of nine followed by zero takes the Seed B assignment. The second value need not still exceed eight.

### Multiplicity validation

The routine reads Descriptor+`0x0c` as a word and compares it with one.

If it equals one, State+`0x41` receives B-byte. Otherwise, State+`0x41` first receives A-byte, and Descriptor+`0x0c` is read again. A second value other than two causes immediate rejection. A second value of two causes the current word at State+`0x3c` to be incremented and written back.

That increment is expressed as signed 32-bit addition. Its behavior on signed overflow is not established.

After either accepted branch, Descriptor+`0x0c` is read again and copied as a word to State+`0x14`. This final read is not validated. Consequently, mutable memory can produce a stored multiplicity different from either value used for validation.

### Common initialization and descriptor tag

After both validation stages pass, the routine writes the following fields in the listed order:

| Destination | Width | Value |
|---|---|---|
| State+`0x08` | Word | Seed A |
| State+`0x0c` | Word | Seed A |
| State+`0x05` | Byte | A-byte |
| State+`0x04` | Byte | A-byte |
| State+`0x4c` | Word | Seed A |
| State+`0x45` | Byte | B-byte |
| State+`0x46` | Byte | A-byte |

It then reads the descriptor tag at Descriptor+`0x00` as a byte. This tag is independent of the selector.

| Tag | Effect |
|---|---|
| 0, 3, or 5 | Write Seed A to State+`0x78` and continue. |
| 1 | Read external value K1, write it as a word to State+`0x78`, and continue. |
| 7 | Read external value K7, write it as a word to State+`0x78`, and continue. |
| Any other byte | Reject with `0xffffffff`. |

K1 and K7 are distinct external sources; neither their values nor whether those values differ is established. A rejected tag does not cause an assignment to State+`0x78` by this dispatch.

### Dependency-selected initialization

For every accepted tag, the routine invokes F-6d8c85350b66 with Descriptor+`0x40`. Its result is tested only for equality with zero. Negative and positive nonzero results select the same branch.

No preservation guarantee is established for this call. Subsequent descriptor, state, and external-memory reads can therefore observe changes made by the dependency.

**Zero result.** State+`0x28` receives A-byte. Word writes of Seed A follow at State+`0x2c`, `0x30`, `0x34`, `0x38`, `0x7c`, and `0x80`, in that order. These are not necessarily clearing operations.

**Nonzero result.** State+`0x28` receives B-byte. The routine then copies words from Descriptor+`0x40`, `0x44`, `0x48`, and `0x50` to State+`0x2c`, `0x30`, `0x34`, and `0x38`, respectively, in that order. The values read from the last three descriptor locations are also retained for the immediately following conditional transformation. Descriptor+`0x4c` is not directly copied.

The routine next reads the current byte at State+`0x58`. If that byte equals seven, it rewrites State+`0x30`, `0x34`, and `0x38`, in order, using the corresponding retained descriptor values shifted left by one bit. These transformations do not reread the descriptor or the destination words.

The shifts are expressed on signed 32-bit values. Behavior for negative operands or unrepresentable doubled values is not established as portable signed arithmetic.

The condition uses the stored byte, not the original full selector. Thus, selector `0x00000107` skips the selector-seven branch at entry but initially stores seven at State+`0x58` and can enable this later transformation.

### Nonzero-result derived words

After the conditional transformation, the routine reads State+`0x78` as an unsigned word and retains that value for both derived calculations.

For State+`0x7c`, it reads the current word at State+`0x30`, adds the retained State+`0x78` value using unsigned 32-bit arithmetic, and writes the resulting word. The sum is reduced modulo \(2^{32}\).

For State+`0x80`, the read sequence is significant:

- The routine reads State+`0x34` as an unsigned word for comparison with the retained State+`0x78` value.
- If the comparison value is no greater than the retained value, it selects zero and does not perform the subtraction read.
- If the comparison value is greater, it reads State+`0x34` again and subtracts the retained State+`0x78` value from this second read using unsigned 32-bit arithmetic.
- The selected result passes through a signed 32-bit intermediate before the word write to State+`0x80`.

The second State+`0x34` read controls the subtraction even if it no longer satisfies the comparison. Unsigned subtraction can therefore wrap when that second value is smaller than the retained State+`0x78` value. The comparison is not repeated.

Only when State+`0x34` remains unchanged between the comparison and subtraction reads does the unsigned calculation implement a clamped difference:

\[
\max(\operatorname{uint32}(\text{State+0x34})-\text{retained base},\,0).
\]

That formula is not an unconditional contract under mutable memory. Conversion of an unsigned result exceeding the signed 32-bit range into the signed intermediate remains target-dependent; the unsigned arithmetic itself is modulo \(2^{32}\).

### Success epilogue

Both dependency-result branches converge on the same epilogue. The routine obtains a counter address from an external pointer slot, writes A-byte to State+`0x8c`, then reads, increments, and writes the addressed 32-bit counter.

The counter increment is expressed as signed addition. Signed-overflow behavior and atomicity are not established.

After the counter write, the routine invokes F-266a29fc8dc5 with an externally supplied first argument and a second argument of zero. Its result is ignored. Normal return from that call is followed by this routine returning zero.

## Side effects

All mutations are incremental. An explicit failure leaves earlier writes in place:

| Rejection | Mutations already reached |
|---|---|
| Width | Entry writes, selected selector-specific writes, and State+`0x40`. |
| Multiplicity | The preceding writes, any width-dependent State+`0x3c` assignment, and State+`0x41`. |
| Descriptor tag | Accepted multiplicity handling, State+`0x14`, and common initialization. |

No explicit rejection path reaches either dependency call or the explicit shared-counter increment.

No non-aliasing or synchronization guarantee is visible. State writes may affect later descriptor reads if the regions overlap. External values, pointer storage, or the counter may also alias accessed memory. Read ordering is therefore part of the contract; neither descriptor fields nor derived state operands may generally be treated as stable snapshots.

F-6d8c85350b66 can affect values read after its call. F-266a29fc8dc5 runs after the explicit state and counter updates and can affect their eventual observed values. The writes specified here do not guarantee those values remain unchanged until the caller observes them.

## Dependencies

| Opaque function ID | Established interaction | Unresolved contract |
|---|---|---|
| F-6d8c85350b66 | Called once after an accepted tag, with Descriptor+`0x40`; zero versus nonzero selects initialization behavior. | Data interpretation, access extent, mutation, global preservation, ownership, failure signaling, and termination. |
| F-266a29fc8dc5 | Called once after the counter increment, with an external first argument and zero; return value ignored. | Argument meaning, mutation, global preservation, ownership, failure signaling, and termination. |

Additional external dependencies are K1, K7, the counter pointer slot, its addressed counter, and the first argument supplied to F-266a29fc8dc5. Their application meanings and validity requirements beyond the visible accesses are **documented_only**.

## Acceptance examples

These are conditional acceptance scenarios, not executed tests. Unless explicitly varied, memory is stable and non-overlapping, required accesses are valid, dependencies return normally without changing illustrated fields, and signed arithmetic stays within its established range.

1. **Width rejection retains entry mutations.** With selector three and stable width seventeen, the result is `0xffffffff`. The entry writes, A-byte writes at State+`0x42` and `0x43`, four-byte copy into State+`0x59` through `0x5c`, and State+`0x40` assignment have occurred. Multiplicity handling and dependency calls have not.

2. **Repeated width reads can change the accepted path.** If the first width read returns nine and the second returns zero, State+`0x3c` receives Seed B. Width validation does not reject this sequence.

3. **Multiplicity rejection writes its status byte first.** With width eight and both multiplicity validation reads returning zero, State+`0x41` receives A-byte before rejection. State+`0x3c` is not incremented, and State+`0x14` is not assigned.

4. **Narrow width preserves the increment base.** With width eight, stable multiplicity two, and initial State+`0x3c` equal to ten, that word becomes eleven. With width nine instead, it receives Seed B before being incremented.

5. **An unsupported full selector can succeed.** Selector `0x00000107`, accepted validation fields, and an accepted tag can reach success. No selector-specific byte group is initialized, but the stored selector byte is seven and can enable the later shifts.

6. **Stable derived-state calculation.** Let Seed A be zero, selector seven, tag zero, and F-6d8c85350b66 return nonzero. Descriptor words three, five, and seven at offsets `0x44`, `0x48`, and `0x50` produce transformed state words six, ten, and fourteen. With State+`0x78` remaining zero, State+`0x7c` becomes six and State+`0x80` becomes ten.

7. **The subtraction uses the second read.** Suppose the retained State+`0x78` value is ten. If the comparison reads twelve from State+`0x34` and the subtraction reads eleven, State+`0x80` receives one, not two.

8. **A successful comparison does not prevent unsigned wrap.** With retained base ten, a comparison read of twelve followed by a subtraction read of three produces the unsigned intermediate `0xfffffff9`. The path does not clamp this to zero. Its subsequent conversion to the signed intermediate and stored representation remain subject to target semantics.

9. **A failed comparison skips the reread.** With retained base ten and comparison value ten, zero is selected for State+`0x80`; no second State+`0x34` read is required by this branch.

10. **Dependency zero is not rejection.** For accepted preceding inputs, a zero result from F-6d8c85350b66 selects the Seed A assignments, followed by the counter increment and final call. Normal completion returns zero.

11. **Unsupported tags reject late.** With valid width and multiplicity but tag two, common initialization has occurred before rejection. Neither dependency call nor the explicit counter increment is reached.

## Uncertainties

The following remain **documented_only**:

- Complete calling convention, origins of effective inputs, and whether the supplied body is independently callable.
- Application meanings, units, field names, selector meanings, and ranges beyond the visible checks.
- Seed values, external values, pointer validity, byte order, alignment, and fault behavior.
- Signed-overflow behavior, signed-shift behavior, and out-of-range unsigned-to-signed conversion.
- Aliasing, concurrency, interrupts, access atomicity, and actual execution guarantees for mutable memory.
- Dependency ownership, mutation, global preservation, error, and termination contracts.

Resetting state, setting flags, or registering an object are plausible intentions, not established behavior.

## Evidence

This contract uses only the supplied static body and opaque dependency mapping. Established observations include access widths and offsets, conditional read repetition, write order, explicit rejection points, arithmetic types, call arguments, and normal return patterns.

In particular, the State+`0x80` calculation contains a comparison read of State+`0x34` and a separate subtraction read on the successful-comparison branch. The subtraction uses the latter value without revalidation; the clamped-difference interpretation consequently requires stability between those reads.

No runtime tests, execution traces, or independent implementation inspection have been performed.
