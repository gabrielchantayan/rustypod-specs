<!-- Trusted provenance: F-7c3c6b4b9df3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7c3c6b4b9df3

## Purpose

F-7c3c6b4b9df3 updates a selector-dependent portion of a caller-provided state region from caller-provided integer storage.

Three recognized selectors update one 32-bit state field each. A fourth recognized selector updates three consecutive 32-bit fields and a separate Boolean indicator. An unrecognized selector conditionally records error value `0x500` in the first state field.

The three-component operation computes its indicator from fresh reads of the input, after storing the components. The indicator is false precisely when those later reads satisfy the tuple condition `(0x00010000, 0, 0)`. With overlapping storage or intervening changes, the indicator need not describe the three values stored earlier.

The field names below describe their observable roles, not an established external API or domain interpretation.

## Interface

### Inputs

The function receives three arguments:

1. **State address.** A pointer to mutable storage containing 32-bit fields and a byte-sized Boolean location at the offsets listed below.
2. **Selector.** An integer compared against four constants: `0x8126`, `0x8127`, `0x8128`, and `0x8129`.
3. **Input address.** A pointer to integer storage. The first 32-bit input word is read on every path, including an unrecognized-selector path. Selector `0x8129` also reads the next two words.

### Output and return

There is no return value. Observable results are memory reads and writes through the supplied addresses.

No allocation, deallocation, pointer retention, or ownership transfer is shown. The caller remains responsible for the lifetime and accessibility of the referenced storage.

### State layout

The following byte offsets use the 32-bit integer layout represented by the supplied listing.

| Role | Byte offset from state address | Access width | Relevant selector |
|---|---:|---:|---|
| Error/status field | `0x000` | 32 bits | Any unrecognized selector |
| Scalar field A | `0x8E4` | 32 bits | `0x8126` |
| Scalar field B | `0x8E8` | 32 bits | `0x8127` |
| Scalar field C | `0x8EC` | 32 bits | `0x8128` |
| Component 0 | `0x8F0` | 32 bits | `0x8129` |
| Component 1 | `0x8F4` | 32 bits | `0x8129` |
| Component 2 | `0x8F8` | 32 bits | `0x8129` |
| Non-default indicator | `0x8FC` | Boolean, modeled as one byte | `0x8129` |

The input words begin at byte offsets `0`, `4`, and `8` from the input address.

The Boolean access is distinct from the surrounding 32-bit accesses. It must not be expanded into a four-byte write on the basis of the supplied evidence. The precise target Boolean representation and machine-level access widths remain subject to the limitations in **Uncertainties**.

### Preconditions

For normal completion:

- The first input word must be readable regardless of selector.
- Selector `0x8129` requires all three input words to be readable and its three component destinations and indicator destination to be writable.
- Each recognized scalar selector requires its corresponding destination to be writable.
- An unrecognized selector requires the error/status field to be readable. If that field is zero, it must also be writable.
- Addresses must satisfy the target platform’s applicable alignment and memory-access requirements.

No explicit null-pointer, bounds, alignment, or selector-range validation is shown. Inaccessible or otherwise invalid storage is outside the established normal-completion contract.

## Behavior

### Common initial read

Every invocation first reads the 32-bit word at input offset `0`.

This read precedes selector-dependent state changes. In particular, an unrecognized selector does not bypass input access merely because its eventual state update does not use the input value.

The initially read word supplies any scalar update and the first component store of a three-component update.

### Scalar selectors

The scalar cases have the following effects:

| Selector | Destination |
|---|---|
| `0x8126` | Scalar field A |
| `0x8127` | Scalar field B |
| `0x8128` | Scalar field C |

Each case stores the initially read 32-bit input word into its destination and then returns.

These cases do not inspect or update the error/status field, the other scalar fields, the component fields, or the indicator. They do not read input offsets `4` or `8`.

The operation is a direct word transfer. There is no demonstrated scaling, saturation, range check, sign-dependent behavior, or numerical conversion.

### Three-component selector

For selector `0x8129`, the relevant access order is:

1. Store the initially read input word into component 0.
2. Read the input word at offset `4`, then store that value into component 1.
3. Read the input word at offset `8`, then store that value into component 2.
4. Read the input word at offset `0` again.
5. If that new first-word value equals `0x00010000`, read the input word at offset `4` again.
6. If the new first-word value equals `0x00010000` and the subsequently read second-word value equals zero, read the input word at offset `8` again.
7. Write the Boolean indicator, then return.

The final Boolean is false only when the conditionally observed values satisfy all three comparisons:

- First word equals `0x00010000`.
- Second word equals zero.
- Third word equals zero.

Otherwise, the indicator is true.

The conditional reads are significant. After the three component stores:

- A first-word value other than `0x00010000` makes the indicator true without further input reads.
- A first-word value of `0x00010000` followed by a nonzero second word makes the indicator true without another third-word read.
- Only a matching first word and zero second word cause the additional third-word read.

The component stores finish before this comparison sequence begins. The comparison uses newly read input values, not necessarily the values previously stored.

### Unrecognized selectors

For every selector other than the four recognized constants:

- Read the 32-bit error/status field at state offset `0`.
- If it is zero, write `0x00000500` to that field.
- If it is nonzero, leave it unchanged.
- Return.

The initial input read still occurs before this error/status handling.

This behavior preserves an existing nonzero status value. It does not merge values, overwrite an existing nonzero value with `0x500`, or clear the status.

Recognized selectors do not consult the error/status field. A pre-existing nonzero status therefore does not prevent their state updates.

### Widths and arithmetic

All component and scalar transfers preserve the supplied 32-bit word value. The comparisons use exact equality with zero or `0x00010000`; they do not depend on whether other input bit patterns are interpreted as positive or negative.

There is no data-dependent arithmetic on input values. The layout uses fixed integer-element offsets converted to byte offsets under a four-byte integer model.

### Aliasing and intermediate state

The supplied listing imposes no explicit non-overlap requirement.

If the input region overlaps the state region, an earlier store can change a value read later during the same invocation. Consequently:

- The three component stores are not a snapshot copy.
- Components 1 and 2 may receive values changed by earlier stores.
- The indicator can reflect input values changed by the component stores.
- Replacing the operation with a bulk copy followed by a comparison of cached values would not generally preserve the listed behavior.

No rollback or transactional boundary is shown. The normal sequence exposes ordered intermediate writes rather than a single all-or-nothing state replacement. Exact failure behavior for invalid memory is not established.

## Side effects

The normal-completion write set is selector-dependent:

- `0x8126`: one 32-bit write to scalar field A.
- `0x8127`: one 32-bit write to scalar field B.
- `0x8128`: one 32-bit write to scalar field C.
- `0x8129`: three 32-bit component writes followed by one Boolean write.
- Any other selector: at most one 32-bit write to the error/status field.

The recognized cases perform their writes even if the destination already contains the value being stored. The unrecognized case is different: a nonzero status field is read but not written.

These write sets describe addresses relative to the state argument. If the caller aliases that storage with another object or with the input region, those writes also affect the overlapping view.

No function calls, direct global-state accesses, external I/O, synchronization, allocation, or release operations are shown. This does not establish atomicity, thread safety, or special behavior for memory-mapped storage.

## Dependencies

No called-function dependency is present in the supplied listing. There are therefore no additional opaque function IDs to reference.

F-7c3c6b4b9df3 depends on caller-supplied memory and the target platform’s integer, Boolean, pointer, and memory-access conventions. No callee behavior is assumed, including preservation of globals or ownership of memory.

## Acceptance examples

These examples are conditional expectations derived from static evidence, not runtime results. Unless stated otherwise, assume valid, ordinarily accessible memory; the stated layout; disjoint input and state regions; and no intervening modification.

### 1. Scalar update with an existing status

Given selector `0x8127`, a first input word of `0xFFFFFFFF`, and an existing status of `0x123`:

- Scalar field B receives the 32-bit word `0xFFFFFFFF`.
- Status remains `0x123`.
- No other state field is written.
- Only the first input word is required.

The existing status does not block the recognized operation.

### 2. Default three-component value

Given selector `0x8129` and input words `(0x00010000, 0, 0)`:

- Components 0, 1, and 2 receive those respective words.
- All three input positions are read again during the conditional comparison.
- The indicator becomes false.
- Status and scalar fields remain unchanged.

### 3. First comparison differs

Given selector `0x8129` and input words `(0, 0, 0)`:

- All three zeros are stored.
- The first input word is read again.
- The second and third input words are not reread during the comparison.
- The indicator becomes true.

The original reads of the second and third words still occur to supply their component stores.

### 4. Second comparison differs

Given selector `0x8129` and input words `(0x00010000, 7, 0)`:

- The three words are stored in their corresponding components.
- The first and second input words are reread.
- The third input word is not reread during the comparison.
- The indicator becomes true.

### 5. Third comparison differs

Given selector `0x8129` and input words `(0x00010000, 0, 1)`:

- All three words are stored.
- All three input words are reread during the comparison.
- The indicator becomes true.

### 6. Unrecognized selector

Given selector `0x812A` and a readable first input word:

- An initial status of zero becomes `0x500`.
- An initial status of `0x501` remains `0x501`.
- No scalar, component, or indicator destination is written.

An unreadable first input word does not meet this example’s preconditions, even though its value is not used for status handling.

### 7. Overlapping input propagates a value

Assume overlapping input and state are supported by the target memory model. Let the input begin at state offset `0x8EC`, immediately before component 0. Initially, the three input words are `(0x00010000, 0, 0)`.

With selector `0x8129`:

- The first component store overwrites the second input word with `0x00010000`.
- The second component store reads that changed value and overwrites the third input word with it.
- The third component store consequently also receives `0x00010000`.
- The comparison rereads a first word of `0x00010000` and a second word of `0x00010000`.
- The indicator becomes true, without another third-word read.

This distinguishes the listed access sequence from a snapshot-copy interpretation.

## Uncertainties

### Established static evidence

The supplied listing establishes the selector constants, field indices, common initial input read, conditional status update, component access sequence, short-circuit comparisons, and final Boolean assignment. It shows no calls or return value.

### Inferred intent

The three-component operation appears to recognize a distinguished default tuple and mark other tuples as non-default. The unknown-selector path appears to preserve an existing error while recording `0x500` when no status is present.

Those interpretations are descriptive only. No domain-specific meaning, fixed-point format, or external error-name assignment is established.

### documented_only

The following remain unknown:

- Independent confirmation of target integer size, endianness, Boolean representation, and machine instruction widths.
- Whether callers guarantee non-overlap or stable input storage.
- Alignment requirements and behavior for invalid, protected, or device-backed memory.
- Concurrent-access semantics, visibility, and atomicity.
- The enclosing object’s total size and unrelated field meanings.
- Any external API definition of the selectors or status value.
- Whether the supplied reconstruction omits behavior not recoverable from this listing.

## Evidence

This contract is based solely on the supplied static function listing. The byte layout uses its four-byte integer interpretation; the Boolean destination is separately identified rather than treated as a full integer field.

No implementation was executed. No runtime tests, instrumentation, platform documentation, or independent binary-level verification were available. The acceptance examples are conditional deductions from the listed accesses and comparisons, not observed outcomes.
