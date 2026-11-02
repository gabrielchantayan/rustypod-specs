<!-- Trusted provenance: F-c38e978ba2dc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c38e978ba2dc

## Purpose

F-c38e978ba2dc conditionally forwards an object reference and a locally constructed descriptor to F-be41255119f8.

The wrapper admits an object only when all three conditions hold:

- The object reference is non-null.
- The object's first 32-bit word equals a value read from a designated global marker.
- The object's 32-bit word at byte offset 0x44 is nonzero.

If any condition fails, the wrapper returns the 32-bit value 7 without calling F-be41255119f8. If all conditions hold, it constructs a three-word descriptor, calls that dependency, and returns its result unchanged.

The descriptor combines an unmodified 32-bit input, an 8-bit input tagged with 0xAA, and a value derived from one byte of the object with fixed bit patterns. The meanings of the marker, tag, and fixed patterns are not established.

## Interface

### Inputs

The recovered interface has three arguments:

| Argument | Recovered width and form | Use |
|---|---|---|
| Object reference | Pointer to storage accessed as 32-bit words | Supplies validation fields and a descriptor field; forwarded unchanged to the dependency |
| Payload | 32 bits | Copied without interpretation into the descriptor |
| Selector | 8 bits | Placed in the low byte of the descriptor's second word |

These names describe data flow, not an established application-level meaning.

A null object reference is explicitly handled. A non-null reference is not sufficient to make the operation safe: the required object locations must be accessible whenever execution reaches their reads. The source establishes no allocation-size, alignment, mapping, or lifetime validation.

### Object fields accessed by this wrapper

| Byte offset | Width | Interpretation in this wrapper |
|---|---|---|
| 0x00 | 32 bits | Equality comparison against the global marker |
| 0x0C | 32 bits | Low eight bits contribute to the descriptor |
| 0x44 | 32 bits | Zero versus nonzero admission check |

The wrapper does not otherwise inspect the object. F-be41255119f8 may impose additional object requirements that are not visible here.

For an accepted object, the highest directly accessed byte is at offset 0x47. This describes the wrapper's access extent, not a guarantee that a contiguous allocation of that size satisfies the dependency's requirements.

### Output

The result is a 32-bit value:

- Exactly 0x00000007 if a wrapper admission condition fails.
- Exactly the result returned by F-be41255119f8 if that dependency returns normally.

The wrapper does not translate, normalize, or interpret the dependency's result. In particular, a result of 7 alone does not distinguish wrapper rejection from a dependency that also returns 7.

## Behavior

### Admission checks

The checks short-circuit in the following order:

1. Check whether the object reference is null.
2. For a non-null reference, compare the object's word at offset 0x00 with the designated global marker.
3. Only if those values compare equal, read the object's word at offset 0x44 and test it against zero.

The null-reference check does not require an object-memory read. A null reference therefore causes return of 7 without accessing either object field or constructing the descriptor.

If the first-word comparison fails, the wrapper does not read the word at offset 0x44 or the word at offset 0x0C. If the word at offset 0x44 is zero, the wrapper does not read the word at offset 0x0C.

The comparison requires both the first object word and the global marker value. Their relative read order within that comparison is not established by the supplied representation. Neither should be assumed to be read before the other.

The word at offset 0x44 is tested only for zero. Any nonzero 32-bit pattern satisfies this gate; the wrapper does not require the value to equal 1 or to represent a positive signed integer.

### Descriptor layout

On the admitted path, the recovered local layout forms a 12-byte descriptor containing three adjacent 32-bit words:

| Descriptor byte offset | Width | Numeric value |
|---|---|---|
| 0x00 | 32 bits | Payload, unchanged |
| 0x04 | 32 bits | 0x0000AA00 combined with the selector in bits 7–0 |
| 0x08 | 32 bits | 0xEF000000 combined with 0x00A00000 and the object's low byte shifted left by 16 bits |

Let \(b\) denote the low eight bits of the object word read at offset 0x0C. The third word has the numeric value:

\[
0xEF000000 \mathbin{\mathrm{OR}}
\bigl((b \mathbin{\mathrm{OR}} 0xA0) \ll 16\bigr).
\]

Consequently:

- Bits 31–24 are exactly 0xEF.
- Bits 23–16 are \(b \mathbin{\mathrm{OR}} 0xA0\).
- Bits 15–0 are zero.

The fixed 0xA0 pattern is combined by bitwise OR. It does not replace the entire contributing object byte. Bits already set in that byte remain set, while bits selected by 0xA0 are forced on.

Only the low eight bits of the 32-bit object word affect the descriptor. The recovered intermediate narrowing does not preserve a sign extension in the final value. Two object words with the same low byte produce the same third descriptor word.

The second descriptor word is zero-extended to 32 bits. For selector 0xFF, for example, its value is 0x0000AAFF, not a sign-extended negative value.

These are numeric word values. The supplied evidence does not independently establish the target byte order, so it does not justify a universal byte-by-byte serialization.

### Construction and call order

After admission succeeds, the visible construction order is:

1. Write the descriptor's word at offset 0x04 using the selector.
2. Read the object's word at offset 0x0C, derive the tagged value, and write the descriptor's word at offset 0x08.
3. Write the payload into the descriptor's word at offset 0x00.
4. Call F-be41255119f8 with the original object reference and a pointer to the descriptor's first word.
5. If the dependency returns, return its 32-bit result unchanged.

All three descriptor words are initialized before the call. Their construction order differs from their order in memory.

This is the order visible in the recovered representation. It is not evidence of memory barriers, volatile access semantics, or a machine-level ordering guarantee under concurrent observation.

The wrapper performs no visible retry, fallback, post-call validation, or explicit cleanup. It does not visibly reread the global marker or object fields after the call.

## Side effects

### Direct wrapper effects

The wrapper directly:

- Reads object memory on paths that reach the corresponding checks or descriptor construction.
- Reads the designated global marker when the object reference is non-null.
- Writes the local descriptor on the admitted path.
- Calls F-be41255119f8 once on that path.

No direct write to object memory or global storage is visible in the wrapper. Rejected paths do not construct the descriptor or call the dependency.

### Ownership and lifetime

The wrapper visibly performs no allocation or deallocation.

The original object reference is passed directly to F-be41255119f8; no object copy is made. The descriptor is temporary storage belonging to the current invocation. Its lifetime does not extend beyond that invocation.

These facts do not establish the dependency's ownership policy. In particular, the supplied evidence does not show whether F-be41255119f8:

- Mutates, releases, retains, or transfers the object.
- Modifies the descriptor.
- Retains the descriptor pointer or copies its contents.
- Reads or changes global state.

A retained descriptor pointer would not acquire a valid post-return lifetime from this wrapper. Whether retention occurs is unknown.

### Errors and abnormal execution

The only explicit wrapper rejection result is 7. There is no separate visible result for each failed admission condition.

A non-null but inaccessible object reference is not converted into that result by any visible handling. Likewise, an inaccessible global marker or a failure within the dependency has no visible recovery path here.

The wrapper's normal return-value contract applies when the required reads and dependency call complete normally. Dependency non-return, faults, and externally imposed exception behavior remain outside the established contract.

## Dependencies

### F-be41255119f8

Called exactly once if, and only if, all wrapper admission checks succeed.

Its arguments are:

1. The original object reference.
2. A pointer to the beginning of the fully initialized three-word descriptor.

Its result is forwarded unchanged as the wrapper's 32-bit result.

The dependency's signature beyond these observed arguments, descriptor interpretation, side effects, validation rules, and error semantics are not supplied. No preservation guarantee for object storage, descriptor storage, or globals can be inferred from this wrapper.

### Global marker

The wrapper depends on one designated global marker value for its first object-word comparison. The marker's initializer, mutability, ownership, and application meaning are not established.

The comparison uses the value obtained for that invocation. The source does not establish that the marker is immutable or stable across calls.

## Acceptance examples

These examples are conditional deductions from the supplied static representation. They are not runtime test results. Cases involving a dependency return assume that the required accesses are valid and that the dependency returns normally.

| Conditions | Required wrapper behavior |
|---|---|
| Object reference is null | Return 7; do not read object memory; do not call F-be41255119f8 |
| Object reference is non-null, but its first word differs from the marker | Return 7; do not read offsets 0x44 or 0x0C; do not call the dependency |
| First word matches, but the word at offset 0x44 is zero | Return 7; do not read offset 0x0C; do not call the dependency |
| First word matches and the word at offset 0x44 is 0x80000000 | Admission succeeds; the gate treats this as nonzero |
| Admission succeeds and F-be41255119f8 returns 0x12345678 | Return 0x12345678 unchanged |
| Admission succeeds and F-be41255119f8 returns 7 | Return 7; the result is indistinguishable from rejection without observing other behavior |

For descriptor values, assume admission succeeds, the payload is 0x12345678, and the selector is 0x5C:

| Object word at offset 0x0C | Descriptor words at offsets 0x00, 0x04, 0x08 |
|---|---|
| 0x00000000 | 0x12345678, 0x0000AA5C, 0xEFA00000 |
| 0x00000001 | 0x12345678, 0x0000AA5C, 0xEFA10000 |
| 0x0000005F | 0x12345678, 0x0000AA5C, 0xEFFF0000 |
| 0x1234565F | 0x12345678, 0x0000AA5C, 0xEFFF0000 |
| 0xFFFFFFFF | 0x12345678, 0x0000AA5C, 0xEFFF0000 |

The equal results for 0x0000005F and 0x1234565F demonstrate the low-byte boundary. The 0x5F example also demonstrates that OR with 0xA0 preserves other set bits rather than replacing the entire byte.

For selector boundaries:

- Selector 0x00 produces second word 0x0000AA00.
- Selector 0xFF produces second word 0x0000AAFF.

## Uncertainties

The following remain **documented_only**:

- **Application meaning:** No established meaning for the object type, marker, nonzero gate, payload, selector, or descriptor tags.
- **Byte order:** Numeric word values are established; their byte serialization is not independently established.
- **Pointer ABI:** Pointer width and calling-convention details are not supplied.
- **Dependency memory access:** The wrapper's 12-byte descriptor construction does not prove that the dependency restricts itself to those 12 bytes.
- **Ownership:** No established dependency policy for retaining or releasing the object or retaining the temporary descriptor pointer.
- **Concurrency:** No established locking, atomicity, snapshot consistency, or protection against object or marker changes between reads.
- **Fault behavior:** No established platform handling for invalid accesses or dependency failures.
- **Return semantics:** The significance of 7 and other result values is unknown beyond the wrapper's explicit rejection behavior.

Inferred intent is limited to describing this as a guarded forwarding operation with a tagged descriptor. That interpretation does not establish a protocol, command type, or resource-management role.

## Evidence

Established static evidence consists of the supplied recovered function representation and the supplied opaque dependency mapping.

That representation supports:

- Ordered short-circuit admission checks.
- Object reads at offsets 0x00, 0x44, and, conditionally, 0x0C.
- Three initialized 32-bit descriptor words in the recovered local layout.
- Explicit narrowing, concatenation, shifting, and bitwise-OR operations.
- One conditional call to F-be41255119f8.
- Direct forwarding of its result, with 7 returned on rejected paths.
- No visible direct object or global writes, allocation, deallocation, or post-call processing.

No dependency implementation, runtime trace, target ABI documentation, or execution results were supplied. No runtime tests have been performed.
