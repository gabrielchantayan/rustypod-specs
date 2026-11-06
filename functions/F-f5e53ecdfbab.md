<!-- Trusted provenance: F-f5e53ecdfbab; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f5e53ecdfbab

## Purpose

F-f5e53ecdfbab advances an object-associated sequence by one 16-bit unit, writes the selected unit to a first output, and obtains a separate 8-bit result from F-c13e7403cc30 for a second output.

If the next sequence unit is zero, the function first calls F-d3922c7fe0ec for a 32-bit value. It constructs either a single-unit sequence from that value’s low 16 bits or a two-unit sequence from one of two byte tables. The first output receives a provisional value before sequence construction and is overwritten after sequence advancement.

The function also captures a global argument-base address between updating the sequence pointer and reading the selected unit. The argument passed to F-c13e7403cc30 is read through that captured address only after the final first-output write. This distinction affects overlapping storage.

**Established behavior:** access widths, refill conditions, sequence construction, pointer advancement, base capture, and ordered output writes.

**Inferred intent:** the sequence appears to hold pending text-like units, with selected input values expanded into pairs. No character encoding or textual meaning is established.

## Interface

The function has three explicit inputs and no explicit return value.

| Input | Role | Required access |
|---|---|---|
| Object reference | Supplies dependency state and a mutable sequence-pointer field | Readable object storage and writable pointer field |
| First output address | Receives a 16-bit unit | Writable for two bytes |
| Second output address | Receives an 8-bit dependency result | Writable for one byte |

The supplied representation uses 32-bit addresses and four-byte pointer storage. The object input is represented as a 32-bit integer used as an address.

### Object and sequence layout

| Object-relative byte offset | Established use |
|---|---|
| `+0x004` | Address passed to F-d3922c7fe0ec |
| `+0x408` | Four-byte field holding the current sequence address |

The address at object offset `+0x004` is passed directly. The function does not load an argument from that address before calling F-d3922c7fe0ec.

Relative to the sequence address used for an access:

| Byte offset | Established use |
|---|---|
| `+0x002` | Next 16-bit unit; entry zero check, refill destination, and eventual selected unit |
| `+0x004` | Second expansion unit or direct-path zero terminator |
| `+0x006` | Expansion-path zero terminator |

The function does not read a unit at sequence offset zero.

These offsets describe accesses, not a guaranteed allocation layout. In particular, the function rereads the sequence-pointer field for successive refill stores; interference can make those stores use different sequence addresses.

### External state

The function also accesses:

- A global comparison value and an expression subtracting one from that value.
- A first table containing at least two readable bytes.
- A second table containing at least two readable bytes.
- A global four-byte location supplying an argument-base address.
- A four-byte argument at offset `+0x008` from the captured argument base.

The concrete comparison value, table contents, and argument-base value are not supplied.

### Preconditions

For ordinary, nonoverlapping operation:

- The object’s pointer field must be readable and writable.
- The entry sequence address must permit a 16-bit read at offset `+0x002`.
- Refill destinations must support the required 16-bit writes through offset `+0x004`, or through `+0x006` for expansion.
- Both output regions must be writable at their respective widths.
- A selected table must permit two separate byte reads.
- The global base location must be readable, and the captured base must permit a four-byte read at offset `+0x008`.
- Address calculations must produce valid addresses in the target execution environment.
- Dependencies must satisfy their own, unspecified preconditions.

No explicit null, alignment, bounds, lifetime, or address-overflow checks enforce these conditions.

## Behavior

### 1. Inspect the next sequence unit

The function reads the four-byte sequence address from object offset `+0x408`, then reads the 16-bit unit two bytes beyond that address.

A zero unit triggers refill. Every nonzero 16-bit pattern bypasses refill. Signed interpretation of the loaded unit does not change this zero test.

When refill is bypassed, F-d3922c7fe0ec is not called, the replacement tables are not consulted, and no refill stores occur. Processing proceeds to advancement.

### 2. Obtain and provisionally publish a refill value

On refill, the function calls F-d3922c7fe0ec with the address at object offset `+0x004`.

The returned value is used as an unsigned 32-bit quantity for the visible threshold and equality decisions. Before those decisions, the function writes its low 16 bits to the first output address.

This provisional two-byte write is observable even though a later write normally replaces it. If the first output overlaps relevant state, the provisional write can affect subsequent classification, table contents, or sequence destinations.

The function does not establish that F-d3922c7fe0ec preserves the object, sequence-pointer field, globals, tables, or output storage. Subsequent sequence stores acquire their addresses from the object field again rather than retaining the entry sequence address.

### 3. Select direct storage or expansion

Every unsigned decoded value below `0x81` takes the direct-storage path, without requiring comparison-class checks.

For values at least `0x81`, expansion is selected if the value matches any of:

- The global comparison value.
- The expression representing that comparison value minus one.
- `0xE6`.
- `0xC6`.

A value matching none of these takes the direct-storage path.

The threshold applies to the full 32-bit value, not its low 16 bits. Truncation for output or sequence storage does not change classification.

On the expansion path, the configurable comparison class is checked again to select the first table. If that class does not match, the fixed values `0xE6` and `0xC6` select the second table. Thus the configurable class has priority when stable values make both classes match.

The global comparison value is referenced repeatedly. The function does not establish a single immutable snapshot for both expansion admission and table selection.

If expansion admission succeeds but the later selection checks match neither class, the visible fallback is a null table address. Subsequent processing attempts to read through that address; no explicit error result or recovery branch handles this case.

### 4. Construct the refill sequence

#### Direct storage

Two 16-bit stores occur in order:

1. The decoded value’s low 16 bits are stored at sequence offset `+0x002`.
2. Zero is stored at sequence offset `+0x004`.

The object’s sequence-pointer field is read again for each destination.

The value is truncated, not range-checked. A decoded value above `0xFFFF` is not rejected merely because it cannot fit in one unit.

#### Two-unit expansion

After choosing a table, the function:

1. Reads its first byte and stores that byte’s zero-extended value as a 16-bit unit at sequence offset `+0x002`.
2. Reads its second byte and stores that byte’s zero-extended value as a 16-bit unit at sequence offset `+0x004`.
3. Stores a 16-bit zero at sequence offset `+0x006`.

A table byte of `0xFF` becomes `0x00FF`, not `0xFFFF`.

The byte reads are separate, with the first destination store preceding the second byte read. The selected table address is retained, but its two bytes are not captured as one snapshot. Each destination uses a fresh read of the sequence-pointer field. Consequently, overlapping storage can change the second table byte or a later destination.

### 5. Advance, capture the argument base, and publish the selected unit

After either refill path, or directly after a nonzero entry check, the function performs the following ordered operations:

1. Reads the current sequence address from object offset `+0x408`.
2. Computes an advanced address two bytes beyond it.
3. Stores that advanced address into the object’s sequence-pointer field.
4. Reads the global argument-base location and captures its four-byte address value.
5. Reads the 16-bit unit at the computed advanced address.
6. Writes that unit to the first output address.

The pointer-field update precedes the global base capture. The base capture precedes both the selected-unit read and the final first-output write.

The selected-unit read uses the computed advanced address; it does not reread the pointer field after the update.

Under stable, nonoverlapping conditions:

- A direct refill finally outputs the decoded value’s low 16 bits.
- An expansion finally outputs the first expanded unit.
- A non-refill call outputs the pending next unit.

The pointer advances even when the selected unit is zero. No visible end-of-sequence status, retry, or rollback accompanies zero.

### 6. Read the argument through the captured base and obtain the second output

After the final 16-bit first-output write, the function reads a four-byte value at offset `+0x008` from the argument base captured in the preceding stage.

That loaded value—not the base address itself—is passed to F-c13e7403cc30. The resulting 8-bit value is then written to the second output address.

The global base location is not reread for this argument access. Therefore, a final first-output write that changes the global base location does not redirect the argument read. A final first-output write that changes storage at the captured base plus `0x008` can change the argument value.

F-c13e7403cc30 is called on both refill and non-refill paths if preceding operations complete. It receives neither output address nor the decoded value as an explicit argument.

## Side effects

Directly visible effects on normal completion are:

- One sequence-pointer update.
- One final two-byte write to the first output.
- One additional provisional two-byte first-output write on refill.
- Two two-byte sequence stores for direct refill, or three for expansion.
- One one-byte write to the second output.
- A conditional call to F-d3922c7fe0ec.
- A later call to F-c13e7403cc30.

No allocation, deallocation, ownership transfer, or retained output-address registration is visible in this function. This does not establish the ownership behavior of either dependency.

No transactional protection is visible. Earlier writes are not rolled back if a later access fails or a dependency does not return normally. Neither dependency result is explicitly checked for failure.

### Aliasing and interference

The function contains no disjointness checks. Relevant consequences of overlapping storage include:

- The provisional first-output write can change state used during classification or refill.
- A sequence store can modify the sequence-pointer field, redirecting subsequent refill stores.
- A sequence store can modify a table byte before its later read.
- The pointer-field update can affect the subsequently captured global base if those storage regions overlap.
- The pointer-field update can affect the selected unit if the updated field overlaps the computed advanced address.
- The final first-output write occurs after base capture. It cannot redirect the later argument read merely by changing the global base location.
- That same output write can modify the four-byte argument storage reached through the already captured base.
- F-c13e7403cc30 can have unspecified effects before the final one-byte output store.
- The final one-byte store can overwrite part of the first output when the output regions overlap.

These are ordering consequences, not a guarantee that every overlapping layout is valid. Exact resulting byte patterns can depend on target byte order and valid-access rules.

## Dependencies

| Opaque ID | Invocation condition | Established use | Unresolved behavior |
|---|---|---|---|
| F-d3922c7fe0ec | Entry next-unit check is zero | Receives the address at object offset `+0x004`; supplies the value classified as unsigned 32-bit | State semantics, errors, ownership, memory effects, preservation of object and globals |
| F-c13e7403cc30 | After final first-output write, if execution reaches it | Receives the four-byte value read at captured-base offset `+0x008`; supplies an 8-bit result | Argument meaning, result meaning, errors, ownership, memory effects, preservation of prior outputs |

No particular dependency result is established as an error indication. The function does not restore earlier state if a dependency changes it.

## Acceptance examples

These are conditional expectations derived from static evidence, not runtime test results. Unless an example states otherwise, storage is valid and nonoverlapping, globals remain stable, address calculations are valid, and dependencies do not interfere with relevant memory.

1. **Pending unit without refill**

   With next unit `0x1234`, F-d3922c7fe0ec is not called. The pointer advances by two bytes. The global argument base is captured, the selected unit is read, and the first output receives `0x1234`. The argument is then read through the captured base and passed to F-c13e7403cc30.

2. **Direct refill below the threshold**

   With a zero next unit and decoded value `0x0041`, the first output provisionally receives `0x0041`. Refill stores `0x0041` and zero. Advancement and base capture follow, and the final first-output write again stores `0x0041`.

3. **Threshold boundary**

   A decoded value of `0x80` takes direct storage even if it matches the configured comparison value. A decoded value of `0x81` expands only if it matches the configurable class; otherwise it takes direct storage.

4. **Fixed-class expansion**

   With decoded value `0xE6`, no configurable-class overlap, and second-table bytes `0x61` and `0x65`, refill stores units `0x0061`, `0x0065`, and zero. The first output changes from provisional `0x00E6` to final `0x0061`. With no intervening changes, the next invocation consumes `0x0065` without refill.

5. **Configurable-class priority**

   If the stable comparison value is `0xE6`, decoding `0xE6` selects the first table rather than the fixed-class table.

6. **Full-width classification and truncation**

   A decoded value of `0x00010041` matching neither configurable value takes direct storage and stores `0x0041`. Its small low 16 bits do not make it a below-threshold value.

7. **Zero and byte widening**

   Decoded zero produces a zero first output but still advances the pointer and reaches F-c13e7403cc30. Separately, a selected table byte of `0xFF` produces unit `0x00FF`, not sign extension.

8. **Changing the global base after capture**

   Suppose the final first-output destination overlaps the global base location. Provided the resulting accesses remain valid, the final write can change that global location, but the argument read still uses the base captured before the write.

9. **Changing the argument after base capture**

   Suppose the final first-output destination overlaps argument storage at captured-base offset `+0x008`. The later four-byte argument read observes the storage after the two-byte output write. Its exact value depends on the overlap and byte order.

10. **Changing the base during pointer update**

    Suppose the sequence-pointer field overlaps the global base location. The pointer update occurs first, so the base capture observes the resulting location contents, assuming this layout permits valid execution.

## Uncertainties

The following remain **documented_only**:

- Concrete comparison values, table contents, and global argument-base contents.
- The comparison global’s declared width and signedness, and the arithmetic rules governing its predecessor expression. The decoded operand is unsigned 32-bit, but that alone does not establish how subtraction is evaluated before comparison.
- Exact address-overflow behavior. No bounds check or explicit wrapping contract is visible.
- Full object extent, allocation boundaries, alignment guarantees, and target byte order.
- Semantic interpretation of the sequence and either output.
- Dependency failure conventions, nonlocal exits, allocation or release behavior, retained references, and preservation of memory.
- Whether concurrent mutation is permitted or synchronized. No synchronization is visible.
- The validity and outcome of reaching the null-table fallback after inconsistent repeated classification checks.
- Caller-imposed aliasing restrictions and target-specific behavior for invalid or partially overlapping accesses.

## Evidence

This contract is based solely on the supplied static representation and opaque dependency mapping. Prior prose and reviewer feedback are not independent evidence.

Established evidence includes the two dependency calls, access widths and offsets, zero-triggered refill, unsigned 32-bit threshold, truncating direct stores, zero-extending table stores, repeated sequence-pointer reads, and pointer advancement before selected-unit access.

The ordering relevant to the argument is explicit: update the sequence-pointer field, capture the global base, read and publish the selected 16-bit unit, then read the four-byte argument through the captured base and call F-c13e7403cc30.

The pending-sequence and text-expansion interpretations are inferred intent, not additional requirements.

No runtime tests have been performed.
