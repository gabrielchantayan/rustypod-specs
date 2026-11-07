<!-- Trusted provenance: F-dbd3ab86c6d9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dbd3ab86c6d9

## Purpose

F-dbd3ab86c6d9 examines a mode-selected stream of bytes for a numeric-shaped sequence and returns a 32-bit Boolean result. It can also write a 32-bit status distinguishing an initial classified run from a completed fractional or exponent component.

Under suitable classification-table contents, the recognized shape is an optional leading sign, a required initial run, an optional fractional component, an optional exponent component, and a selected zero terminator. The function does not calculate a numeric value or return a stopping position.

The classification rule is exact: a selected byte belongs to a run when its classification-table entry equals `0x20`. The supplied evidence does not establish that this means “decimal digit,” “space,” or any other named character class. Consequently, the apparent numeric grammar is conditional on the table contents.

The status is not a success flag. Rejection can occur after the function explicitly writes either zero or one. Moreover, the dependency executes before initial classification and may independently modify globally reachable result storage. No explicit output write by this function does **not** imply that result storage remains unchanged across the whole call.

## Interface

### Input address

The first input is an address from which unsigned eight-bit bytes are selected. There is no length argument.

The mode controls the initial position and distance between successive selected positions:

| Mode | Initial selected position | Stride |
|---|---|---:|
| `1` | Supplied input address | 1 byte |
| `3` | Supplied input address plus 1 byte | 2 bytes |
| Any other value | Supplied input address | 2 bytes |

Mode is represented as a signed 32-bit integer. Comparisons use its full value. Negative values and values other than `1` and `3` receive no separate validation.

Mode `3` therefore selects offsets `1, 3, 5, …`; other non-`1` modes select offsets `0, 2, 4, …`. The function does not combine adjacent bytes into multibyte characters or validate an encoding. Bytes between selected positions are not directly inspected by its input traversal.

Every reached input read must address readable storage. There is no established rejection result for a null input address, inaccessible storage, or traversal beyond an available buffer.

### Optional result address

The second input is either null or an address supporting a four-byte write.

A null address suppresses this function’s explicit result writes. It does not prevent dependency effects on other reachable storage. With a nonnull address, the storage must be writable whenever an explicit write is reached.

The explicit values written are:

- Zero after successful initial classification.
- One after fractional-run completion.
- One after exponent-run completion.

There is no unconditional initialization and no rollback on rejection.

These explicit writes must be distinguished from whole-call effects. F-2a4c402ba617 receives no explicit result address, but its unknown effects may include modifying that storage through a global or another reachable alias. Disjointness from input and classification metadata does not establish preservation by the dependency.

### Return value

The return is a four-byte value with only these explicit outcomes:

- `1`: execution reaches the final check with a selected zero byte.
- `0`: an explicit recognition check fails.

Invalid memory access, dependency failure, or dependency nonreturn is not assigned a Boolean outcome by the supplied evidence.

### Classification representation

F-2a4c402ba617 returns an address used to locate a cell containing a table-base address represented as a 32-bit integer.

Each classification expression uses the selected input byte as an unsigned index from `0` through `255`. That index is a byte offset, with no scaling. The addressed eight-bit table entry is compared for equality with `0x20`.

Within the supplied representation, the byte index is widened to an unsigned 32-bit value. Addition to the represented 32-bit table base therefore has unsigned 32-bit arithmetic semantics. Conversion of the resulting value into an address does not establish a portable address-validity or wraparound guarantee. Valid storage for every reached lookup remains a precondition.

## Behavior

### Selection and leading sign

The function first establishes the stride and, for mode `3`, applies the one-byte initial displacement.

It then tests the initial selected byte for `0x2D` (`-`) or `0x2B` (`+`). A recognized sign advances the selected position by one stride. At most one leading sign is consumed.

These sign checks precede the dependency call. The selected byte can be read separately for the two comparisons; no single-read snapshot is established.

The function then calls F-2a4c402ba617 once, without explicit arguments. This occurs even when the selected sequence is empty or cannot satisfy initial classification, provided execution reaches and returns from the call.

Dependency-side mutation can therefore occur after the leading-sign decision but before the first classification. In particular, input inspected after the call need not match its earlier contents.

### Initial classified run

The first classification obtains a table base through the dependency-returned cell and tests the current selected byte.

If its table entry is not `0x20`, the function returns `0` without an explicit result write. This establishes no whole-call preservation guarantee for result storage: the dependency has already executed.

If the first classification succeeds:

- The function writes four-byte zero when a nonnull result address was supplied.
- It then reads the table-base cell again and retains that base.
- It advances by one stride before checking the next selected byte.
- It continues advancing while selected bytes classify as `0x20` through the retained base.

The first successful lookup thus precedes the explicit zero write, while the retained base is obtained after that write. Result storage that overlaps the table-base cell can change which base is retained.

Under stable-memory assumptions, at least one classified byte is required before fractional or exponent recognition is considered. A sequence beginning with a period is not accepted merely because a valid fractional run follows it.

### Fractional component

After the initial run stops, the function tests the current selected byte for `0x2E` (`.`).

If it is not a period, fractional processing is skipped. If it is a period, the function advances by one stride and tests the next selected byte using the retained table base.

A failed classification returns `0`. A successful classification is followed by a run condition that classifies the same selected position again. The function advances while that condition succeeds.

When this run ends, the function writes four-byte one if the result address is nonnull. The write occurs before exponent-marker inspection and before final termination checking.

With stable input and metadata, a period therefore requires at least one following classified byte. The period alone does not cause a write of one.

With changing memory, the initial successful fractional check and the immediately following run-condition check can disagree. The function provides neither a single-read guarantee nor a snapshot guarantee.

### Exponent component

After the initial run or completed fractional processing, the function tests the current selected byte for `0x65` (`e`) or `0x45` (`E`).

If neither matches, exponent processing is skipped. A marker causes an advance by one stride. The function then recognizes at most one exponent sign: `0x2B` (`+`) or `0x2D` (`-`). A recognized sign causes one additional stride advance.

The resulting selected byte must classify as `0x20`; otherwise, the function returns `0`. As with fractional processing, a successful first classification is followed by a run condition that evaluates the current selected position again.

Exponent classification differs from earlier run processing: **each exponent classification expression reads the table-base cell again.** It does not use the base retained for initial-run continuation and fractional processing.

The function advances while exponent classification succeeds. After the run ends, it writes four-byte one if the result address is nonnull.

Under stable-memory assumptions, an exponent requires at least one classified byte after its marker and optional sign. Only one exponent component is processed.

### Final termination check

After optional fractional and exponent processing, the function reads the current selected byte:

- Zero produces return value `1`.
- Any nonzero value produces return value `0`.

There is no trailing-whitespace skip or general suffix handling.

A zero byte terminates recognition only when execution reaches this final check at that position. If zero itself classifies as `0x20`, a run can consume it and continue to later selected positions.

### Classification precedence

Runs consume table-approved bytes before punctuation or termination is considered.

Consequently:

- A period classified as `0x20` can be consumed as an ordinary run member.
- An exponent marker classified as `0x20` can be consumed without initiating exponent processing.
- A classified zero can be consumed rather than terminate recognition.
- Leading signs and signs immediately following a recognized exponent marker are handled before classification at those positions.

The numeric-shaped interpretation must not override these rules.

## Side effects

### Explicit result writes

The function’s explicit writes are limited to the optional four-byte result storage:

| Milestone | Written value |
|---|---:|
| Initial classification succeeds | `0` |
| Fractional processing completes its run | `1` |
| Exponent processing completes its run | `1` |

A call containing both a completed fractional component and a completed exponent component performs both writes of one. The supplied representation does not combine them.

Rejection does not undo earlier writes. Under stable, disjoint-memory conditions:

- Missing fractional content leaves the earlier explicit zero as the last explicit result write.
- A malformed exponent after a completed fraction leaves one as the last explicit result write.
- An invalid suffix after a completed fraction or exponent likewise follows a write of one.
- Initial classification failure produces no explicit result write.

These statements describe this function’s writes, not an unconditional whole-call storage-preservation contract.

### Dependency effects on result storage

F-2a4c402ba617 executes before any initial-classification rejection. Its effects are unknown.

Even when result storage is valid and disjoint from input, the table, and the table-base cell, the dependency may have another route to that storage. It may therefore alter the result before this function decides whether to write it.

For example, if the dependency changes result storage from `7` to `9` and initial classification then fails, this function returns `0` without explicitly writing the result; the whole call nevertheless leaves `9`, assuming no other changes.

A guarantee that rejection preserves the original result requires additional assumptions, including preservation by the dependency.

### Aliasing and repeated reads

No disjointness checks are present. Result writes can overlap selected input, classification entries, or the table-base cell.

The ordering above then matters:

- The explicit zero write can affect the subsequently retained table base or future input reads.
- Fractional completion writes one before exponent recognition.
- Exponent completion writes one before the final zero check.
- Exponent classification reloads the table-base cell, whereas initial-run continuation and fractional classification use a retained base.

The evidence establishes the order of these control-flow milestones. It does not establish an exact machine access sequence between all operands within a classification expression.

No atomicity, synchronization, memory barriers, or safe concurrent-mutation semantics are established.

### Ownership and resources

The function contains no explicit allocation, release, retention, or ownership-transfer operation. It requires readable and writable storage as described, but does not establish ownership of that storage.

This does not constrain the dependency. Its allocations, releases, ownership effects, global mutations, and lifetime requirements remain unknown.

## Dependencies

### F-2a4c402ba617

**Established interaction:** Called once after leading-sign handling and before initial classification, on executions that reach the call. It receives no explicit arguments. Its returned address identifies a cell containing the table-base representation.

**Required usable result:** The returned cell and every reached table entry must be readable. The function performs no explicit null, table-size, or address-validity check.

**Read strategy:** The cell is read for initial classification, again after the optional explicit zero write, and for every exponent classification expression. The second read supplies the retained base used by initial-run continuation and fractional processing.

**documented_only:** The dependency’s actual purpose, table contents, initialization rules, return and failure behavior, effects on globally reachable result storage, other global effects, and ownership behavior are not supplied. No preservation or release guarantee is assumed.

## Acceptance examples

These examples are static deductions, not runtime observations.

Unless an example states otherwise, assume:

- Exactly ASCII bytes `0` through `9` have classification entry `0x20`.
- Signs, punctuation, letters, spaces, and zero do not.
- Input and classification metadata remain stable throughout recognition.
- Result storage is valid, disjoint from input and classification metadata, and initially contains `7`.
- **F-2a4c402ba617 preserves result storage.**
- No other actor changes result storage.
- The dependency returns usable metadata, and all reached accesses are valid.
- Each sequence has an accessible selected zero terminator.

Sequence notation describes selected bytes, not necessarily adjacent physical bytes.

| Selected sequence | Return | Final result | Reason |
|---|---:|---:|---|
| Empty sequence | `0` | `7` | Initial zero fails classification; dependency preservation makes the absence of an explicit write sufficient here. |
| `+` | `0` | `7` | One sign is consumed; the following zero fails initial classification. |
| `123` | `1` | `0` | Initial run ends at zero. |
| `-123` | `1` | `0` | One leading sign is consumed. |
| `.5` | `0` | `7` | Initial classification fails before fractional processing. |
| `1.` | `0` | `0` | The period lacks a classified following byte. |
| `1.5` | `1` | `1` | Fractional processing completes and termination succeeds. |
| `1e2` | `1` | `1` | Exponent processing completes. |
| `1E-2` | `1` | `1` | One exponent sign is consumed. |
| `1e+` | `0` | `0` | No classified byte follows the exponent sign. |
| `1.5e+` | `0` | `1` | Fractional completion wrote one before exponent rejection. |
| `1.5x` | `0` | `1` | Fractional completion does not excuse a nonzero suffix. |
| `12 ` | `0` | `0` | A trailing space is not consumed or accepted as termination. |
| `++1` | `0` | `7` | Only the first leading sign is consumed. |

If the dependency-preservation assumption is removed, the rows with no explicit result write no longer establish final result `7`. For example, dependency mutation to `9` followed by rejection of `.5` permits return `0` with final result `9`.

With null result storage and otherwise equivalent stable input and metadata, the recognition outcomes remain the same, but this function performs no explicit result writes.

For physical bytes `0x31, 0x78, 0x32, 0x78, 0x00`:

- Mode `1` selects `1` followed by `x`, returning `0` with result zero.
- Mode `0` selects `1`, `2`, then zero, returning `1` with result zero.
- Mode `-1` follows the same selection as mode `0`.

For mode `3`, physical bytes `0x78, 0x31, 0x78, 0x32, 0x78, 0x00` select `1`, `2`, then zero and return `1` with result zero.

Under a different table that classifies space, but not decimal digits or zero, a nonempty selected run of spaces followed by zero can return `1` with result zero. A decimal-digit sequence can instead fail initial classification.

## Uncertainties

- **documented_only — Classification meaning:** Equality with `0x20` is established; the character class it represents is not.
- **documented_only — Encoding:** Traversal is byte-based. No encoding, byte order, or wide-character validity rule is established.
- **documented_only — Address interpretation:** The representation establishes 32-bit table-base arithmetic. Machine address conversion, pointer provenance, alignment requirements, and invalid-address behavior are not fully specified.
- **documented_only — Invalid memory:** Unreadable input or metadata and unwritable result storage have no established Boolean failure contract.
- **documented_only — Dependency effects:** Preservation of result storage, input, metadata, globals, ownership, and resource lifetimes is unknown.
- **documented_only — Mutable memory:** Repeated reads and retained-versus-reloaded bases are established. Concurrent execution semantics and precise machine access granularity are not.
- **Inferred intent:** The punctuation and status writes suggest numeric-text validation that distinguishes integer-only forms from fractional or exponential forms. This interpretation does not replace the classification-driven behavior.

## Evidence

Established static evidence includes the mode comparisons, selected-byte displacement and stride, sign and punctuation checks, dependency-call placement, classification equality with `0x20`, table-base read strategy, conditional four-byte writes, and explicit zero-or-one returns.

The absence of an explicit output write on initial classification failure is established. Whole-call preservation of result storage is not: the dependency has already executed, and its effects are unspecified.

Acceptance examples are deductions under their stated assumptions. No runtime tests have been performed. No dependency implementation, concrete classification table, ownership contract, encoding specification, or complete machine-level memory-fault model was supplied.
