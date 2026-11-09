<!-- Trusted provenance: F-808b7cbef44d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-808b7cbef44d

## Purpose

F-808b7cbef44d processes a counted byte sequence consisting of a leading selector, a separator, and decimal fields. It either emits a compact encoding into caller-provided storage or computes the encoded length without writing that storage.

**Established behavior:** decimal fields are accumulated as unsigned 32-bit values and encoded in most-significant-group-first base-128 form. The first field receives a selector-dependent adjustment. Period and space are interchangeable separators; empty fields are possible; decimal overflow wraps; some short inputs produce a zero result without an error report.

**Inferred intent:** the selector adjustment and encoding resemble an identifier encoding. This resemblance does not establish compliance with an external identifier standard. In particular, the visible selector and first-field checks must not be replaced with stricter standard-derived rules.

This contract describes static evidence from the supplied representation. It does not establish dependency internals, physical memory-access behavior after compilation, or runtime results.

## Interface

### Arguments

The function has four ordered arguments:

| Position | Meaning | Represented type and interpretation |
|---|---|---|
| 1 | Destination address | Integer-typed address. Zero selects length-only operation; nonzero selects destination-writing operation. |
| 2 | Destination capacity | Signed 32-bit integer, measured in bytes. Consulted only in destination-writing operation. |
| 3 | Source address | Address used to access unsigned 8-bit source elements. |
| 4 | Source count | Signed 32-bit integer. Zero returns immediately; exactly −1 requests a replacement count from F-1cb585febd84; other values are used directly. |

The representation uses signed 32-bit integers for counts, encoded lengths, and destination offsets, and unsigned 32-bit integers for decimal field values. The callable ABI, address width, and interpretation of integer-to-address conversions are **documented_only**.

### Result

The represented return type is a signed 32-bit integer.

- Ordinary completion returns the accumulated encoded byte count.
- Length-only operation returns the same count without writing the destination.
- Every explicit error path invokes F-4fc89ec41a6d and, if that dependency returns normally, returns zero.
- An original source count of zero returns zero without either dependency being invoked.

A zero result is ambiguous: it can mean no fields were emitted, an ignored two-byte input, or a reported error.

No terminator is appended. No source-consumption count or end address is returned.

### Preconditions and boundaries

For the memory accesses described here to be valid:

- Each source byte reached by execution must be readable when accessed.
- Destination storage must support the writes admitted by the capacity check.
- Signed count, length, and address calculations must remain within the supported range.
- Dependencies must return normally for subsequent behavior to occur.

A supplied positive count does not validate either address. Negative effective counts do not prevent the initial source access.

Stable, non-overlapping source and destination storage is a sufficient assumption for the ordinary parsing examples below. Overlap is not checked or handled by first copying the complete source. An output write can therefore alter a source byte that has not yet been read.

Neither dependency is assumed to preserve source memory, destination memory, global state, or ownership merely because it receives no visible buffer argument.

## Behavior

### Entry and replacement count

The accumulated encoded length starts at zero.

If the original source count is zero, the function returns zero immediately. It performs no source access, destination write, or dependency call.

If the original count is exactly −1, the function invokes F-1cb585febd84 with no visible arguments and uses its return value as the effective source count. There is no second zero-count guard. A replacement count of zero therefore still leads to a first-byte access.

The replacement-count call precedes all source-byte evaluations. Any memory changes it causes can affect subsequent parsing.

### Selector evaluation and minimum length

The first source byte is evaluated before the function checks whether the effective count is sufficient.

Selector admission subtracts 0x30 from the unsigned byte after promotion to a signed integer and checks whether the result is less than three. Consequently, the accepted byte range is **0x00 through 0x32 inclusive**, not just ASCII `0`, `1`, and `2`.

A byte of 0x33 or greater selects error reason **0x7A**.

After admission, the representation evaluates the first source byte again to compute forty times its difference from 0x30. For stable source storage:

| Selector byte | Contribution |
|---|---:|
| ASCII `0` | 0 |
| ASCII `1` | 40 |
| ASCII `2` | 80 |
| Below ASCII `0` | A negative multiple of 40 |

For accepted, stable bytes, the contribution ranges from −1920 through 80.

The repeated source expression is relevant when describing the representation, but it does not establish two distinct physical loads in a compiled implementation. Mutation between evaluations is not assigned deterministic behavior by this contract.

After calculating the contribution, the function checks whether the effective count minus one is less than one. Subject to non-overflowing signed arithmetic, effective counts of one or less select error reason **0x8A**.

Thus selector rejection takes precedence over the short-count error.

If this check succeeds, the function reads the second source byte, advances the source position past the first two bytes, and reduces the remaining count by two.

### Outer iteration and separator validation

Each outer iteration first examines the remaining count.

- If fewer than one byte remains, parsing completes and returns the accumulated encoded length.
- Otherwise, the previously read byte must be period, 0x2E, or space, 0x20.
- Any other previously read byte selects error reason **0x83**.

The count check precedes separator validation. As a result, an accepted selector with an effective count of exactly two causes the second byte to be read but not validated. The return value is zero regardless of that second byte’s value.

Period and space have the same separator role. Tabs, newlines, and other whitespace bytes have no special acceptance.

### Decimal field scanning

After accepting a preceding separator, the function initializes a new unsigned 32-bit field value to zero. At least one counted byte remains at this point.

It then reads source bytes in increasing address order. Each read advances the source position and decrements the remaining count.

- Period or space terminates the field immediately.
- ASCII digits `0` through `9` update the value as ten times the previous value plus the digit.
- Any other byte selects error reason **0x82**.

The digit-range calculation uses unsigned arithmetic. Bytes below ASCII `0`, including NUL, are invalid within a field rather than being interpreted as digits or terminators.

Decimal accumulation wraps modulo \(2^{32}\). There is no decimal-overflow error.

A separator encountered as the first scanned byte produces an empty field whose numeric value is zero. Consecutive separators are not collapsed.

If the final counted byte is a digit, that digit is incorporated and the field is encoded without requiring a terminator. If it is a separator, it terminates the current field; it creates no additional field unless another counted byte remains.

### First-field adjustment

Before encoding each field, the function tests whether the accumulated encoded length is zero. Within non-overflowing ordinary operation, this identifies the first field because every emitted field occupies at least one byte.

For this first field:

1. If the selector contribution is less than two and the accumulated field value exceeds 39, error reason **0x93** is selected.
2. Otherwise, the selector contribution is added to the unsigned field value modulo \(2^{32}\).

For stable ASCII selectors, the range restriction applies to `0`, but not to `1` or `2`. The visible behavior therefore permits `1.40`.

The restriction also applies to admitted selector bytes below ASCII `0`, whose contributions are negative. Adding a negative contribution can produce a large unsigned value.

The check is applied to the already-wrapped decimal accumulator. It does not constrain the mathematical value of an arbitrarily long decimal string before wrapping.

Subsequent fields receive neither this adjustment nor this range check.

### Encoded byte layout

Each adjusted or unadjusted field value occupies one through five bytes.

- The value is divided into 7-bit groups.
- Groups appear most significant first.
- Every nonfinal byte has its high bit set.
- The final byte has its high bit clear.
- No unnecessary leading zero group is included.
- Zero occupies one byte, `00`.

| Unsigned field value | Encoded bytes |
|---:|---|
| 0 | `00` |
| 127 | `7F` |
| 128 | `81 00` |
| 16383 | `FF 7F` |
| 16384 | `81 80 00` |
| 4294967295 | `8F FF FF FF 7F` |

The representation uses local temporary storage to arrange the groups before output. Its 24-byte capacity exceeds the five bytes required for an unsigned 32-bit value.

### Length-only operation

With a zero destination address, each completed field’s encoded size is added to the accumulated length. Destination capacity is not checked.

Length-only operation still performs selector validation, field parsing, decimal arithmetic, first-field checks, and error reporting. It is not an unchecked size estimate.

### Destination-writing operation

With a nonzero destination address, the function computes the next complete field’s encoded size before writing it.

It compares the signed capacity with the accumulated length plus that size:

- A smaller capacity selects error reason **0x6B**.
- An equal or larger capacity admits the field.

Subject to representable arithmetic, a negative capacity cannot admit a field because its required end offset is positive.

After a successful check, bytes are written in increasing destination-offset order. The complete field has already been scanned and validated before these writes begin.

A failed check causes **no direct write of that field by this function**. Earlier fields may already have been written before F-4fc89ec41a6d is invoked. The function performs no explicit rollback or clearing.

**Destination contents after the error dependency runs are unspecified without an additional preservation assumption.** The dependency could modify destination storage through an alias despite receiving no visible destination argument.

## Side effects

### Source accesses

The visible ordering is:

1. Optional replacement-count dependency call.
2. First-byte selector admission evaluation.
3. Accepted-path first-byte evaluation for the selector contribution.
4. Minimum-count check.
5. Second-byte read.
6. Further forward reads while fields are scanned.

On ordinary positive-count paths, the representation does not explicitly read beyond the counted source extent. The initial access remains possible for zero replacement counts and negative effective counts.

Destination writes occur between field scans. Consequently, overlap can change later parsing.

### Destination effects

The function directly writes only in destination-writing operation, after a complete field passes all applicable checks. It appends no terminator and performs no explicit destination clearing.

On an explicit error:

- Some earlier fields may have been written before the dependency call.
- The failing field is not directly emitted.
- The function itself performs no rollback after the call.
- No final destination-content guarantee follows unless F-4fc89ec41a6d is additionally assumed to preserve that storage.

Likewise, an error before any direct output write does not prove that the dependency leaves the destination unchanged.

### Ownership and external state

No direct allocation, deallocation, ownership transfer, or address retention is visible in this function. These observations do not establish the dependencies’ ownership behavior.

Immediately before each explicit error report, a local four-byte location is set to zero. Its address is not visibly supplied to the dependency; no external meaning for this store is established.

Dependency logging, global-state changes, indirect memory access, retention, release, and nonlocal control flow remain unknown.

## Dependencies

| Opaque function ID | Invocation | Contract boundary |
|---|---|---|
| F-1cb585febd84 | Once when the original source count is exactly −1; no visible arguments | Returned value replaces the count. How it obtains that value and what state it changes are unknown. |
| F-4fc89ec41a6d | On explicit errors, with ordered arguments 13, 100, and the selected reason | Return value is ignored. If it returns normally, this function returns zero. Memory preservation and other effects are unknown. |

Error reasons:

| Reason | Condition |
|---|---|
| 0x7A | Initial selector evaluation fails admission. |
| 0x8A | Accepted selector followed by insufficient effective count. |
| 0x83 | Required preceding separator is neither period nor space. |
| 0x82 | Scanned field byte is neither a digit nor an accepted separator. |
| 0x93 | First-field value exceeds 39 while the selector contribution is less than two. |
| 0x6B | Destination capacity is smaller than the next complete field’s required end offset. |

Checks occur in the order described above, not as a whole-input validation pass. In particular, field syntax and first-field range checks precede that field’s capacity check.

## Acceptance examples

These are **conditional static expectations, not executed tests**. Unless stated otherwise, counts equal the displayed ASCII lengths, storage is valid, source and destination do not overlap, source bytes remain stable, signed arithmetic is representable, and invoked dependencies return normally.

For error cases, output observations describe this function’s direct actions **before** F-4fc89ec41a6d runs. Final destination contents remain unspecified unless dependency preservation is explicitly added.

| Input or condition | Expected behavior |
|---|---|
| Original count zero | Returns zero without source access, destination writes, or dependency calls. |
| `1.2`, capacity 1 | Writes `2A`; returns 1. |
| `2.999`, capacity 2 | Combines 999 with 80; writes `88 37`; returns 2. |
| `0.39`, capacity 1 | Writes `27`; returns 1. |
| `0.40` | Reports 0x93 and returns zero; this function emits no field before reporting. |
| `1.40`, capacity 1 | Writes `50`; returns 1. |
| `1 2`, capacity 1 | Writes `2A`; returns 1. |
| `1..2`, capacity 2 | Empty first field becomes 40; writes `28 02`; returns 2. |
| `1.2.`, capacity 1 | Writes `2A`; returns 1. The final separator creates no additional field. |
| `1.`, any capacity | Reads the second byte and returns zero without an error call. |
| `1x`, any capacity | Reads but does not validate `x`; returns zero without an error call. |
| `1x2` | Reports 0x83; returns zero. |
| `1.2x` | Reports 0x82 before directly emitting that field; returns zero. |
| `1.2.128`, capacity 2 | Writes `2A` before discovering that the next two-byte field cannot fit. Reports 0x6B and returns zero. Contents after reporting are unspecified. |
| Same capacity-error case, additionally assuming the error dependency preserves destination storage | Returns zero with the earlier `2A` still present; the failing field was not emitted. |
| `1.2.128`, zero destination, any capacity | Returns 3 without direct destination writes or capacity checking. |
| `1.2`, nonzero destination, capacity −1 | Reports 0x6B before directly emitting the first field; returns zero. |
| `2.4294967296`, capacity 1 | Decimal accumulation wraps to zero; adjustment produces 80. Writes `50`; returns 1. |
| `/ .`, capacity 5 | `/` contributes −40; space precedes an empty field ended by period. Writes `8F FF FF FF 58`; returns 5. |

If F-1cb585febd84 returns zero, a readable first source byte is still required. An admitted selector leads to reason 0x8A; a rejected selector leads to 0x7A first. These expectations concern source memory as it exists after that dependency returns.

## Uncertainties

- **documented_only — ABI and addresses:** Exact calling convention, address width, integer-address conversion behavior, and externally declared signature are not independently established.
- **documented_only — signed overflow:** Extreme count subtraction, accumulated-length addition, and destination-address arithmetic are not assigned portable results. In particular, sufficiently negative counts can overflow the minimum-length expression.
- **documented_only — dependencies:** Neither dependency’s memory effects, ownership effects, global-state preservation, or normal return is established.
- **documented_only — post-error storage:** A normal return from F-4fc89ec41a6d does not imply preservation of earlier output.
- **documented_only — mutation and overlap:** No synchronization, alias protection, or stable-snapshot guarantee is visible. The repeated first-byte expression does not establish a physical-load count.
- **documented_only — count discovery:** F-1cb585febd84 receives no visible source argument. Calling it a conventional source-string-length routine would be unsupported.
- **Inferred intent:** Identifier-like encoding is a useful interpretation, not an additional validation requirement.

## Evidence

This contract is grounded in the supplied static representation: argument uses, represented integer widths, branch order, byte expressions, unsigned arithmetic, output stores, dependency calls, and return paths.

Static evidence establishes selector admission, modulo-\(2^{32}\) decimal accumulation, separator-consumption order, first-field adjustment, base-128 layout, whole-field capacity checks, and direct output writes preceding some later error calls.

Static evidence does **not** establish destination preservation across the error dependency. It establishes only that earlier writes occur before that call and that this function performs no explicit rollback.

No runtime tests, dependency inspection, ABI verification, or external-standard conformance checks have been performed.
