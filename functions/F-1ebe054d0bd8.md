<!-- Trusted provenance: F-1ebe054d0bd8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1ebe054d0bd8

## Purpose

F-1ebe054d0bd8 accepts a selector, performs preliminary state and control-word updates, and either rejects the selector or completes a configuration update through two dependencies.

**Established static behavior:** selectors zero and one are accepted. They select different configuration **sources**, different numeric extents, and different control-bit operations. The supplied representation does not establish that the two configuration sources contain unequal values. Every other selector returns a failure bit pattern after performing preliminary writes; rejection is not side-effect-free.

**Inferred intent:** the accepted selectors appear to identify operating modes. The meanings of the modes, configuration data, control words, and extents are not established. The constants 480 and 576 do not establish units or a display-related purpose.

The function reads the global state-base source twice before validation. The first reading supplies a retained base for later 32-bit state-field accesses. The second supplies the address for the selector-byte store. Those readings are not established to produce the same address.

## Interface

The represented interface has two inputs and a 32-bit result.

| Item | Width | Established use |
|---|---:|---|
| First input | 32 bits | Not referenced by the supplied body. It does not directly determine branches, explicit stores, or dependency arguments. |
| Selector | 32 bits | Compared at full width against zero and one; its low eight bits are stored before validation. |
| Result | 32 bits | Zero on a normally completed accepted path; `0xFFFFFFFF` on a normally completed rejected path. |

The failure result is an all-ones 32-bit pattern. Its interpretation as signed negative one depends on the caller’s type convention. Equality with zero or one does not depend on signed interpretation.

### Global inputs and address roles

The body uses four global sources:

- A control-region base source.
- A state-record base source, read twice initially.
- A 32-bit configuration source selected by selector zero.
- A separate 32-bit configuration source selected by selector one.

For this contract:

- **Retained control base** means the value obtained from the control-base source at entry.
- **Retained state base** means the value obtained by the first reading of the state-base source.
- **Selector-store base** means the value obtained by the second reading of the state-base source.
- **Configuration zero** and **configuration one** identify the two configuration sources, not necessarily different stored values.

The selector-store base is used for the byte store only. Subsequent explicit state-field accesses use the retained state base.

| Address relative to the stated base | Access width | Use |
|---|---:|---|
| Selector-store base +2 | 8 bits | Truncated selector store |
| Retained state base +24 | 32 bits | Selected configuration; later dependency argument |
| Retained state base +28 | 32 bits | Primary extent; later copy source |
| Retained state base +32 | 32 bits | Extent copy destination |
| Retained state base +36 | 32 bits | Dependency-result destination |
| Retained control base +8 | 32 bits | First control word |
| Retained control base −1,048,572 | 32 bits | Second control word |

The negative control displacement is `−0xFFFFC`, not `−0x100000`. The two control offsets differ by 1,048,580 bytes, or `0x100004`, before any address-wrap considerations.

The represented base values occupy 32-bit integer-sized storage. The evidence does not establish a portable rule for address overflow, pointer provenance, or alignment handling.

### Preconditions for normal completion

Every reached global read and addressed memory access must be supported by the execution environment. In particular:

- The base sources must be readable in the stated sequence, including both state-base readings.
- The selector-store destination must permit an eight-bit write.
- Reached control and state fields must permit the specified 32-bit accesses.
- The selected configuration source must be readable on an accepted path.
- Both dependencies must return normally for an accepted path to reach its normal return.
- The retained state storage must remain usable for the accesses performed after the calls.

No pointer validation, bounds check, alignment recovery, or access-fault handling is shown. Disjoint storage is not established.

## Behavior

### Initial base reads and unconditional updates

The initial sequence is:

1. Read the control-base source and retain the resulting base.
2. Read the state-base source and retain the resulting base.
3. Read the state-base source again to obtain the selector-store base.
4. Store the selector’s low eight bits at selector-store base +2.
5. Read the first 32-bit control word, clear bits 0 through 3 with mask `0xFFFFFFF0`, and write the result to that word.
6. Read the second 32-bit control word, clear bit 2 with mask `0xFFFFFFFB`, and write the result to that word.
7. Evaluate the selector for branch selection.

These are three distinct base-source readings: one control-base reading followed by two state-base readings. The byte-store address is not derived from the retained state base in the supplied representation.

If the state-base source changes between its two readings, or has access-sensitive behavior, the selector byte can be written into a different region from the later state fields. Equality of the two state bases is therefore a conditional storage assumption, not a general guarantee.

Each control update uses the value read for that particular update. No atomic operation spanning a read/write pair, the two control words, or the complete function is established.

Selector validation uses the original full-width input. The byte truncation does not turn another selector into an accepted selector.

### Selector zero

After the unconditional updates, selector zero performs these operations in order:

1. Read the first control word again and write the value read back to the same address.
2. Read the second control word again and write the value read back to the same address.
3. Read configuration zero and store that 32-bit value at retained state base +24.
4. Store 480, or `0x000001E0`, at retained state base +28.
5. Perform the shared accepted-path completion sequence.

The same-value control writes are part of the represented access sequence. They must not be treated as absent merely because they would leave ordinary stable memory numerically unchanged. Reads or writes may have access-triggered effects in an unspecified execution environment.

With ordinary, stable, non-aliased storage and no intervening modification, this phase leaves the first control word’s low four bits clear and the second control word’s bit 2 clear. Other bits retain their original values under those assumptions.

### Selector one

After the unconditional updates, selector one performs these operations in order:

1. Read the first control word again, set bit 1 with mask `0x00000002`, and write the result back.
2. Read the second control word again, set bit 2 with mask `0x00000004`, and write the result back.
3. Read configuration one and store that 32-bit value at retained state base +24.
4. Store 576, or `0x00000240`, at retained state base +28.
5. Perform the shared accepted-path completion sequence.

With ordinary, stable, non-aliased storage and no intervening modification, the first control word’s low nibble becomes `0x2`, and the second control word has bit 2 set. Other bits retain their original values under those assumptions.

The operational contract remains the ordered reads and writes. Replacing them with a single final-value update would omit represented accesses.

### Rejected selectors

Any full-width selector other than zero or one returns `0xFFFFFFFF` after the unconditional updates.

Before that return:

- The low selector byte has been stored using the separately obtained selector-store base.
- The first control word has undergone its clearing read/write pair.
- The second control word has undergone its clearing read/write pair.

Neither dependency is called. No explicit access is made to either configuration source, and no explicit store is made to retained state offsets +24, +28, +32, or +36.

These are access-level statements. They do not guarantee that those fields retain their previous values: aliasing or access-sensitive storage could cause the earlier writes to affect them indirectly.

No rollback occurs.

### Shared accepted-path completion

After the selected configuration and primary extent have been stored, both accepted paths continue in this order:

1. Invoke F-5a3d0a15a663 with no explicit arguments.
2. After it returns, read the 32-bit value at retained state base +24.
3. Pass that value as the single explicit argument to F-eb34c4d75211.
4. After that dependency returns, store its 32-bit result at retained state base +36.
5. Read the 32-bit value at retained state base +28.
6. Store that value at retained state base +32.
7. Return zero.

The post-call reads are significant:

- The argument to F-eb34c4d75211 is whatever occupies the configuration field after F-5a3d0a15a663 returns. It need not equal the value originally selected.
- The copied extent is read after F-eb34c4d75211 returns and after its result is stored. It need not remain 480 or 576.
- Neither field is restored to its originally selected value.

The result from F-eb34c4d75211 is not interpreted as success or failure. Any normally returned 32-bit pattern is stored. If the remaining accesses complete, F-1ebe054d0bd8 returns zero.

## Side effects

### Partial updates and access counts

A rejected request still performs the selector-byte store and both preliminary control updates. Accepted requests expose additional intermediate states before and between dependency calls.

For the function’s own explicit accesses:

- The control-base source is read once.
- The state-base source is read twice, both times before the selector-byte store.
- Each rejected path reads and writes each control word once.
- Each accepted path reads and writes each control word twice.
- Only the selected configuration source is explicitly read on an accepted path.
- The selector byte is not explicitly rewritten after validation.

Dependency accesses are additional and unknown.

No transaction, lock, rollback, or all-or-nothing update is shown.

### Address retention and aliasing

Later state-field accesses use the retained state base, not the selector-store base and not a freshly loaded global base. A dependency’s replacement of the global state-base value does not redirect those explicit later accesses.

This does not imply that the retained storage remains valid. If a dependency releases, invalidates, or repurposes that storage, the body shows no recovery.

No non-aliasing guarantees are supplied. Global sources, control locations, selector-byte storage, and state fields may overlap unless an external contract rules that out. Consequently, general guarantees are expressed as ordered operations rather than unconditional final field values.

### Ownership and concurrency

No explicit allocation or release appears in this body. The configuration argument and stored dependency result are unclassified 32-bit values. The evidence establishes neither resource ownership nor a borrowing, transfer, retention, or destruction obligation.

The body establishes no synchronization or atomicity. Concurrent modifications can affect the values consumed by later reads, and read-modify-write sequences can overwrite intervening changes. Hardware ordering, memory-mapped access semantics, and compiler-level volatility guarantees are not established.

## Dependencies

| Opaque ID | Invocation | Established handling |
|---|---|---|
| F-5a3d0a15a663 | Once per accepted selector, after configuration and primary-extent stores; no explicit arguments | No result is consumed. The configuration field is read after the call returns. |
| F-eb34c4d75211 | Once after F-5a3d0a15a663, with one 32-bit argument read from retained state base +24 | Its 32-bit result is stored at retained state base +36 without interpretation. |

Neither dependency is called on a rejected path.

Their effects on globals, captured storage, ownership, resource lifetime, error reporting, or termination are unknown. No preservation guarantee may be inferred merely because the caller subsequently uses retained addresses.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Unless an example states otherwise, assume valid ordinary storage, stable base sources, no relevant aliasing or concurrent changes, and normally returning dependencies.

### Selector zero with preserved fields

Assume both state-base readings produce the same address. Let the initial first and second control words be `0x1234567F` and `0xABCDEF07`. Assume the dependencies preserve the relevant fields, and F-eb34c4d75211 returns a value R.

Required observations:

- The selector destination receives byte zero.
- The first control word is written as `0x12345670`.
- The second control word is written as `0xABCDEF03`.
- The first and then second control words each undergo an additional same-value read/write-back.
- Configuration zero is read and stored at retained state offset +24.
- Offset +28 receives 480.
- F-5a3d0a15a663 runs before F-eb34c4d75211.
- F-eb34c4d75211 receives the stored configuration value.
- Offset +36 receives R, then offset +32 receives 480.
- The function returns zero.

### Selector one with preserved fields

Under the same initial control values and preservation assumptions:

- The selector destination receives byte one.
- The preliminary control writes produce `0x12345670` and `0xABCDEF03`, in that order.
- The branch-specific writes then produce `0x12345672` and `0xABCDEF07`, in that order.
- Configuration one is read and stored at retained state offset +24.
- Offset +28 receives 576.
- After the calls and result store, offset +32 receives 576.
- The function returns zero.

### Equal configuration-source contents

Suppose the two configuration sources both contain the same 32-bit value Q and the dependencies preserve that field.

Selector zero and selector one must each store Q and later pass Q to F-eb34c4d75211. Their control operations and primary extents still differ. Equal source contents do not contradict either accepted path.

### Different state-base readings

Suppose the first state-base reading produces address A and the second produces a distinct address B. Assume both regions are valid and non-aliased.

For selector one:

- The selector byte is written at B +2.
- The configuration and 576 stores target A +24 and A +28.
- The post-call configuration read, result store, and extent copy also use A.
- No explicit selector-byte store at A +2 occurs merely because A was retained.

If a dependency subsequently changes the global state-base source to a third address, the caller’s remaining state accesses still use A.

### Rejection despite a valid-looking byte

Selector 256 stores byte zero but returns `0xFFFFFFFF`, because the full-width selector is not zero or one. Selector 257 similarly stores byte one and is rejected. The all-ones selector pattern stores byte `0xFF` and is rejected.

Each case performs both preliminary control updates and calls neither dependency.

### Dependency-modified fields and unusual results

If F-5a3d0a15a663 replaces retained state offset +24, F-eb34c4d75211 receives that replacement value.

If F-eb34c4d75211 changes offset +28, the later extent copy uses the value read after the result store at +36, not a cached branch constant.

If F-eb34c4d75211 returns `0xFFFFFFFF` normally, that pattern is stored at +36. The enclosing function still returns zero if its remaining accesses complete.

## Uncertainties

The following remain **documented_only**:

- Concrete addresses and initial global values.
- Equality or inequality of the two state-base readings.
- Values of the two configuration sources, including whether they are equal.
- Ordinary-memory versus access-sensitive behavior of the global sources and addressed locations.
- Endianness, alignment requirements, and effective-address overflow behavior.
- Aliasing among sources, destinations, and dependency-accessible storage.
- Semantic names and units for configuration and extent fields.
- Dependency behavior, including global mutations, ownership effects, and storage lifetime.
- Behavior after faults, nonlocal exits, or calls that do not return.
- Exact correspondence between the represented interface and the original calling convention or source-level types.

These unknowns do not remove the established distinction between the two state-base readings, the full-width selector checks, or the displayed access widths, masks, offsets, dependency order, and return patterns.

## Evidence

This contract is based solely on the supplied static representation and opaque dependency mapping.

The representation supports the ordered control-base read and two state-base reads; the separately addressed selector-byte store; the preliminary control updates before rejection; the two accepted branches; distinct configuration sources without a value-inequality guarantee; numeric extents; retained-base state accesses; dependency order; post-call reads; and return patterns.

Operating-mode terminology is inferred intent only. No runtime tests, hardware observations, dependency inspection, or external interface validation have been performed.
