<!-- Trusted provenance: F-bfda76581bef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bfda76581bef

## Purpose

F-bfda76581bef obtains an object address from F-60d99bc349ed, writes initial fields using two reads of one global location, derives a mode byte and Boolean from an implicit input, invokes F-ee7c284830cf, performs further initialization, invokes an external operation, and returns the original object address.

The established behavior is a sequence of reads, writes, and dependency calls. “Object initialization” is inferred intent, not proof of allocation, construction, ownership transfer, or a particular object type.

This contract distinguishes:

- **Established static evidence:** operations represented in the supplied function.
- **Inferred intent:** interpretations such as initialization, metadata, arrays, or copying.
- **documented_only:** behavior unresolved by the supplied representation and dependency mapping.

No runtime tests have been performed.

## Interface

### Inputs

The supplied representation declares no explicit parameters, but accesses an entry-stack-associated value as an address. It reads a 32-bit unsigned word at byte offset `0x58` from that address.

The behavioral interface therefore includes an **implicit address input**. Its calling convention, stack placement, and relationship to any source-language parameter are documented_only. The apparent zero-parameter declaration does not establish that no caller-provided state is required.

Other inputs are:

- The object pointer returned by F-60d99bc349ed.
- **Two separate 32-bit reads of the first global location**, both after F-60d99bc349ed returns. The first value is retained for the field at object offset `0x0A4`; the second supplies the field at offset `0x000`.
- One 32-bit read of a second global location, after F-ee7c284830cf and six scalar zero writes.
- Ambient state consumed or modified by dependencies.

There are **three global reads overall**, not two. The two reads of the first location need not be assigned equal values by this contract.

### Output

On ordinary completion, the routine returns the same pointer value received from F-60d99bc349ed.

Return values from later dependencies do not determine the result. There is no explicit success flag, error code, null-return branch, or alternate result.

### Widths and address model

The representation uses 32-bit integer words and byte offsets. The mode field receives a one-byte write.

For the external operation, the object pointer is converted to a 32-bit integer representation before adding `0x0AA`. Interpreting that argument as the ordinary interior address requires a compatible address model and a valid address calculation. Behavior in a wider-pointer environment, or when that conversion loses address information, is documented_only.

### Storage layout and extent

Offsets are relative to the pointer returned by F-60d99bc349ed.

| Byte offset or range | Direct behavior |
|---|---|
| `0x000–0x003` | Receives the second read of the first global |
| `0x0A4–0x0A7` | Receives the retained first read of that global plus `0x124` |
| `0x0A8` | Receives the low byte of the implicit-input word |
| `0x0A9` | No direct write shown |
| Starting at `0x0AA` | Supplied to an external operation with third argument 10 |
| `0x0B4–0x0CF` | Seven 32-bit scalar fields receive zero |
| `0x0D0–0x147` | Thirty consecutive 32-bit fields receive zero |
| `0x148–0x1BF` | Thirty consecutive 32-bit fields receive zero |
| `0x1C0–0x237` | Thirty consecutive 32-bit fields receive zero |

The highest direct write covers bytes `0x234–0x237`. Validity of all direct object writes therefore requires writable storage extending through offset `0x237`: at least `0x238` bytes, or 568 bytes.

This extent does not establish the object’s actual size or cover additional storage accessed by dependencies.

## Behavior

### Obtain the object pointer

The routine first calls F-60d99bc349ed without explicit arguments in the supplied representation.

Its returned pointer is retained for subsequent operations and eventual return. No null, alignment, extent, or lifetime check appears before use.

Because both reads of the first global occur after this call, changes to that global made by F-60d99bc349ed can affect either read. Its pre-call value is not an input snapshot.

### Read the first global twice and write the initial fields

The established order is:

1. Read the first global location as a 32-bit value and retain that value.
2. Read the same global location again.
3. Write the value from the **second read** to object offset `0x000`.
4. Write the value from the **first read**, increased by `0x124`, to object offset `0x0A4`.

No dependency call or object write is shown between the two global reads. Nevertheless, they are separate reads in the supplied representation. The field at `0x0A4` is not derived from the value written at `0x000`, and the routine performs no equality check between the two samples.

The increment `0x124` is decimal 292. The retained value, arithmetic, and destination are represented using 32-bit integers. No range check, saturation, or overflow-handling branch is present.

When the addition is representable under the represented signed arithmetic, its result is established. For values outside that range, portable signed-overflow semantics are not established. Unconditional modulo-\(2^{32}\) wrapping must not be inferred solely from the 32-bit destination.

### Read the implicit input and establish the mode

After both initial field writes, the routine reads one 32-bit unsigned word at offset `0x58` from the implicit address input.

It then writes the low eight bits of that word to object offset `0x0A8`. This is one byte, not a 32-bit store. There is no direct write to the adjacent byte at `0x0A9`.

The same sampled word determines the Boolean supplied to F-ee7c284830cf:

- True if the low eight bits equal 2.
- False otherwise.

The upper 24 bits do not affect the stored byte or Boolean.

The input read follows the writes at `0x000` and `0x0A4`. If the input address aliases storage affected by those writes, the read observes the state at this later point; the routine does not snapshot the input before modifying the object.

### Invoke mode-dependent initialization

F-ee7c284830cf receives:

1. The original object pointer.
2. The Boolean testing whether the sampled low byte equals 2.

This call follows the mode-byte store and precedes all explicit scalar and region zero writes.

No return value is used. Its contract is otherwise unavailable: it must not be assumed to preserve object contents, globals, ownership, or storage lifetime. The preceding stores are established events, not unconditional final-value guarantees.

### Zero the scalar fields and read the second global

After F-ee7c284830cf returns, the routine performs six 32-bit zero writes in this order:

1. `0x0C4`
2. `0x0C8`
3. `0x0CC`
4. `0x0B4`
5. `0x0B8`
6. `0x0BC`

It then reads the second global location as a 32-bit value.

After that read, it writes a 32-bit zero at offset `0x0C0`.

Thus, the third global read overall occurs between the zero writes at `0x0BC` and `0x0C0`. Its value is not sampled at entry and must not be assumed unchanged across either preceding dependency call.

### Invoke the external operation

The routine calls an external dependency, identified here by the local opaque label **D-01**, with:

1. The object pointer’s 32-bit integer representation plus `0x0AA`.
2. The sampled value from the second global.
3. The numeric value 10.

No published opaque function ID was supplied for this dependency. D-01 identifies it only within this contract.

Under a compatible address model, the first argument denotes the interior object address at byte offset `0x0AA`. The call occurs after all seven scalar zero writes and before the ninety region zero writes.

The argument pattern suggests a size-bounded memory operation, possibly copying ten bytes. That interpretation is inferred intent, not established behavior. The unconditional requirement is the call with these argument values.

If an independent dependency contract establishes that D-01 copies exactly ten bytes into its first argument, the destination interval is `0x0AA–0x0B3`, inclusive. That interval excludes byte `0x0A9` and ends immediately before the scalar field at `0x0B4`.

The second argument’s interpretation as an address, handle, encoded value, or other input remains documented_only. No return value from D-01 is used.

### Zero the middle region

After D-01 returns, the routine performs thirty 32-bit zero writes in ascending address order, beginning at offset `0x148`.

The first write covers `0x148–0x14B`; the last covers `0x1BC–0x1BF`. The region contains 120 bytes.

The count is fixed. It does not depend on object contents or a caller-provided length.

### Zero the lower and upper regions in pairs

The routine next performs thirty pairs of 32-bit zero writes.

Within each pair, it writes the lower-region field first and the upper-region field second. The initial pair is at offsets `0x0D0` and `0x1C0`. Each following pair advances both offsets by four bytes. The final pair is at `0x144` and `0x234`.

This is an interleaved write sequence, not a complete pass over one region followed by a complete pass over the other.

Together, the three thirty-word regions occupy `0x0D0–0x237` without gaps. Their initialization order is the middle region first, followed by paired lower-region and upper-region writes.

### Return

After the region writes, the routine returns the original pointer.

There is no visible cleanup, rollback, explicit deallocation, or final validation.

## Side effects

The direct operations comprise:

- Two 32-bit initial-field writes.
- One byte-sized mode write.
- Seven 32-bit scalar zero writes.
- Ninety 32-bit region zero writes.
- One 32-bit read through the implicit input address.
- Three 32-bit global reads: two from the first location and one from the second.
- Three dependency calls.

These are 100 direct object-store operations: 99 word stores and one byte store. Dependency reads and writes are additional and unknown.

No allocation or ownership policy is established. F-60d99bc349ed might return existing storage; returning that pointer does not itself transfer ownership. Absence of an explicit release operation does not prove that dependencies preserve allocation lifetime.

The final ninety stores occur after the last dependency call. On normal return, assuming valid, live storage and no concurrent or external modification, every word in `0x0D0–0x237` is zero.

The seven scalar zeros precede D-01, so their final values depend on that dependency’s effects. The initial fields and mode byte precede both later calls and have still weaker final-value guarantees.

No explicit synchronization, atomic transaction, or memory-ordering mechanism is shown. The stated sequence describes the supplied representation; it does not independently establish a cross-thread observation guarantee.

Invalid addresses, insufficient storage, invalid alignment, arithmetic outside the supported execution model, dependency failure, and nonreturning calls have no explicit handling. Exact fault or exception behavior is documented_only.

## Dependencies

| Opaque identifier | Established interaction | Unresolved contract |
|---|---|---|
| F-60d99bc349ed | Called first; its returned pointer is retained and returned | Allocation, size, initialization, failure signaling, ownership, lifetime |
| F-ee7c284830cf | Receives the pointer and the low-byte-equals-2 predicate | Predicate meaning, object mutations, global effects, ownership and lifetime effects |
| D-01 | Receives the offset-derived address representation, the later global sample, and 10 | Exact operation, argument interpretation, bounds, overlap rules, failure and memory effects |

All operations following a dependency call are conditional on that call returning and leaving the subsequent accesses valid. No dependency is assumed to preserve globals or own, retain, or release memory.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

### Low-byte classification

Given valid input storage and otherwise valid execution:

| Sampled input word | Byte written at `0x0A8` | Boolean passed to F-ee7c284830cf |
|---|---:|---|
| `0x00000002` | `0x02` | True |
| `0xA5B6C702` | `0x02` | True |
| `0x00000100` | `0x00` | False |
| `0x00000003` | `0x03` | False |

These constrain the direct write and call argument, not the mode byte’s final value after dependencies run.

### Distinct first-global samples

If the first read of the first global yields `0x00001000` and the second yields `0x00002000`, the initial writes are:

- `0x00002000` at offset `0x000`.
- `0x00001124` at offset `0x0A4`.

This example is conditional on those two read results being supplied by a valid execution environment. It does not prescribe how the location changes.

### Equal first-global samples

Only under the explicit assumption that both reads yield `0x00001000` must the writes be:

- `0x00001000` at offset `0x000`.
- `0x00001124` at offset `0x0A4`.

An implementation that reads the global once and reuses that value does not reproduce the represented two-read contract merely because the values happen to agree in this example.

### Required ordering

Where the accesses are observable, a conforming trace must preserve:

1. F-60d99bc349ed before both reads of the first global.
2. The retained first read before the second read, followed by the stores at `0x000` and `0x0A4`.
3. Both initial stores before the implicit-input word read.
4. The mode-byte store before F-ee7c284830cf.
5. The second-global read between the scalar zero writes at `0x0BC` and `0x0C0`.
6. D-01 before all ninety region writes.
7. All thirty middle-region writes before the lower/upper paired writes.
8. The lower-region write before the upper-region write in every pair.

Matching only final bytes is insufficient where these access distinctions are observable.

### Return and final region values

If all calls return normally, the storage remains valid, and no external modification intervenes, the result equals the pointer returned by F-60d99bc349ed and all ninety words spanning `0x0D0–0x237` are zero.

### Conditional external-copy behavior

If D-01 is independently established to copy exactly ten bytes to its first argument, and address formation is valid, the destination bytes are `0x0AA–0x0B3`. Without that dependency evidence, a particular copied payload is not an acceptance requirement.

## Uncertainties

The following remain documented_only:

- The exact calling convention and origin of the implicit address.
- The object’s semantic type, full layout, allocation source, and actual extent.
- The meaning of either global and whether their locations can alias other accessed storage.
- Whether the initial fields represent dispatch metadata.
- Overflow semantics for the initial addition and address calculations outside their valid ranges.
- Ownership, release obligations, and dependency effects on lifetime.
- D-01’s exact behavior and argument interpretation.
- Dependency mutations of fields and globals.
- Values of bytes not directly written.
- Fault, exception, concurrency, reentrancy, and cross-thread visibility behavior.

Calling the repeated regions arrays or D-01 a copy operation remains inferred intent.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping.

The representation establishes two distinct reads of the first global: the first is retained for the offset-`0x0A4` computation, while the second supplies the offset-`0x000` store. A later read of the second global brings the total to three global reads.

The represented access widths, four-byte word indexing, fixed iteration bounds, and call sequence establish the offsets, write counts, ordering, call arguments, and returned-pointer identity described above.

Dependency implementations, a complete ABI, memory validity, ownership rules, and boundary arithmetic semantics are not established. No runtime tests, dependency executions, or memory observations have been performed.
