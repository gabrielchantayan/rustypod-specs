<!-- Trusted provenance: F-389c41c1cbea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-389c41c1cbea

## Purpose

F-389c41c1cbea recognizes a zone-designator-like byte sequence and stores a signed numeric value in a caller-provided record.

Established behavior supports three forms after leading classified bytes:

- An uppercase or lowercase UTC-style marker, `Z` or `z`.
- A signed numeric form delegated to F-d770f78fa23f.
- An empty remainder.

The signed form combines two dependency-produced integers as a first component multiplied by 60, plus a second component, with the leading sign applied to the result. Interpreting this result as an offset in minutes is inferred intent, not an independently established external contract.

The function returns a one-byte status: zero for acceptance and one for rejection. Rejection does not imply that the destination was left unchanged. In particular, a successfully parsed numeric value is stored before the function checks for trailing unwanted content.

## Interface

### Inputs

**Input address.** A pointer to readable bytes. Bytes are interpreted as unsigned values from 0 through 255 when indexing a classification table. The input is expected to have enough accessible storage for all reads performed by this function and by F-d770f78fa23f.

A zero byte serves as the final acceptance condition. It is not an unconditional stopping condition for classification scans: those scans test a classification bit without first checking whether the byte is zero.

**Destination address.** An address identifying a caller-provided record. The function directly writes a four-byte field at byte offset 28 from that address. It does not directly access other record fields.

The supplied representation uses a signed integer expression for the numeric value and a four-byte destination store. No byte-order conversion is visible. Record layout, alignment requirements, address validity, and the interpretation of the record’s other fields are external preconditions.

### Output

The return value is one byte:

| Value | Meaning |
|---|---|
| 0 | The final inspected byte is zero after the applicable recognition and classification steps. |
| 1 | The numeric dependency did not return exactly two, or the final inspected byte is nonzero. |

No separate status distinguishes malformed numeric input from unwanted trailing content.

The input cursor is advanced internally only. The function does not return a consumed length or an ending pointer.

### Required external conditions

For normal completion:

- Both supplied addresses must be valid for the accesses described here.
- F-2a4c402ba617 must return a usable address of a cell containing the classification-table base.
- Each classification-table entry addressed by an inspected input byte must be readable.
- Each classification scan must eventually encounter a byte whose classification entry has its low bit clear.
- If the numeric dependency reports success, both integer outputs must be available for the subsequent arithmetic.

There are no explicit checks for null addresses, inaccessible memory, record size, classification-table validity, or arithmetic overflow.

## Behavior

### 1. Obtain classification state and skip leading classified bytes

The function first calls F-2a4c402ba617. Its result is treated as the address of a cell containing a classification-table base.

For the current input byte, the function:

- Reads that byte as an unsigned value.
- Reads the table base through the returned cell.
- Reads the byte-sized table entry indexed by the input value.
- Advances the input pointer by one if bit zero of that entry is set.

This repeats until bit zero is clear.

Calling these bytes “whitespace” is an interpretation of the table’s likely purpose. The established predicate is specifically the low classification bit. The supplied evidence does not identify which byte values satisfy it.

The address returned by F-2a4c402ba617 is retained. The table base is read through that address during classification checks rather than being guaranteed immutable across the whole call.

### 2. Initialize the destination field

After the leading scan stops, the function writes four zero bytes to the destination field at offset 28.

This write precedes the read used to select the input form. Consequently:

- Ordinary rejection after this point still leaves an explicit zero initialization in the write history.
- A failure during the initial dependency call or leading classification scan may occur before any destination write.
- If the destination field overlaps unread input or classification storage, the zero write may affect subsequent reads.

There is no explicit non-overlap check. The ordinary text-parsing interpretation assumes that the destination does not modify the input or classification structures.

### 3. Select the input form

The function reads the byte at the current input position after initializing the destination.

#### Leading minus sign

For `-`, it selects a negative multiplier and invokes the numeric dependency on the byte immediately following the sign.

#### Leading plus sign

For `+`, it selects a positive multiplier and invokes the same numeric dependency in the same way.

#### Uppercase or lowercase marker

For `Z` or `z`, it advances the current pointer by exactly one byte. It does not invoke the numeric dependency and does not explicitly replace the zero already stored in the destination.

It then performs the trailing classification scan described below.

#### Any other byte

For all other byte values, there is no numeric dependency call and no trailing classification scan. The already-read byte determines the result:

- Zero produces status zero.
- Any nonzero byte produces status one.

This is why empty input, or input reduced to an empty remainder by the leading scan, can be accepted without an explicit marker.

### 4. Delegate signed numeric recognition

For either sign, F-d770f78fa23f receives eleven arguments, in this order:

| Position | Supplied value |
|---|---|
| 1 | Address of the byte immediately after the sign |
| 2 | Integer 2 |
| 3 | Integer 0 |
| 4 | Integer 14 |
| 5 | Integer 58, the byte value for `:` |
| 6 | Address for a first integer result |
| 7 | Integer 2 |
| 8 | Integer 0 |
| 9 | Integer 59 |
| 10 | Integer 0 |
| 11 | Address for a second integer result |

These arguments suggest two numeric components, two-character widths, a colon separator, and bounds of 0–14 and 0–59. Those meanings depend on the unavailable contract of F-d770f78fa23f. They are not independently enforced here.

The dependency’s return value is compared with exactly two:

- Any other result causes immediate status one.
- Exactly two allows processing to continue.

On the immediate rejection path, this function makes no numeric destination write after its earlier zero initialization. Effects performed by the dependency itself are not constrained by the supplied evidence.

### 5. Store the signed combined value

Following a dependency return of exactly two, the input pointer advances by exactly six bytes from the sign position. The next inspected position is therefore the byte after a nominal six-byte signed form such as `+05:30`.

This advancement does not use an ending pointer or a consumed-length result from the dependency.

The function then computes the destination value by:

1. Multiplying the first dependency-produced integer by 60.
2. Adding the second dependency-produced integer.
3. Applying the sign selected from the leading byte.
4. Storing the result in the four-byte field at offset 28.

For ordinary bounded component values, this is a signed offset in minutes. If the dependency enforces the apparent inclusive ranges, the possible combined magnitudes are at most 899. Under that condition, the arithmetic fits comfortably in a signed 32-bit integer.

The evidence does not establish those bounds as guarantees. It also does not establish portable overflow behavior for arbitrary dependency outputs. No explicit overflow check or wider intermediate is visible.

### 6. Skip trailing classified bytes and decide acceptance

After a marker or a numeric dependency success, the function performs another classification scan using the same low-bit predicate as the leading scan.

The scan uses the previously obtained classification-cell address. The function does not call F-2a4c402ba617 again. In particular, the evidence does not guarantee that the numeric dependency leaves the contents of that cell or the classification table unchanged.

After the scan stops, the current byte is read:

- Zero produces status zero.
- Any nonzero value produces status one.

For signed numeric input, the numeric destination write has already occurred before this trailing scan and final decision. There is no rollback on trailing-content rejection.

## Side effects

### Direct memory writes

The function directly performs:

- One four-byte zero write at destination offset 28 after the leading classification scan.
- One additional four-byte numeric write at the same offset if the numeric dependency returns exactly two.

It does not directly clear or update the rest of the destination record.

### Read/write ordering

The consequential ordering is:

1. Call F-2a4c402ba617.
2. Read input and classification state during the leading scan.
3. Zero the destination field.
4. Read the input byte that selects the form.
5. For a sign, call F-d770f78fa23f.
6. On numeric success, write the combined signed value.
7. For a marker or numeric success, read input and classification state during the trailing scan.
8. Make the final zero-byte decision.

This ordering is observable through aliasing, dependency effects, or externally observed memory accesses. It must not be replaced by an assumed transactional “validate, then commit” model.

### Ownership and lifetime

No direct allocation, release, or ownership transfer is visible.

The supplied evidence does not establish whether either dependency allocates memory, releases memory, retains arguments, or mutates shared state. F-d770f78fa23f receives addresses for temporary integer outputs; valid execution requires it to make any outputs needed by this function available before returning success.

No explicit logging, I/O, exception translation, or error-object construction is visible.

## Dependencies

### F-2a4c402ba617

Called once at entry, before the destination write.

Its return value supplies an indirection used to obtain a byte-classification-table base. The table entry’s low bit controls both scans.

Unknowns include the table’s character classes, initialization, lifetime, mutability, and concurrency guarantees.

### F-d770f78fa23f

Called only when the post-leading-scan byte is `+` or `-`.

It receives the argument sequence documented above and supplies two integer results. Only a return value of exactly two is treated as success.

Its lexical rules, width handling, separator requirements, bound semantics, partial-output behavior, memory accesses, and global-state effects are not established here.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not runtime test results.

For the examples below, assume disjoint valid storage, stable classification state, a terminating zero whose classification low bit is clear, and no dependency side effects that alter the described outcome.

| Input or condition | Required outcome |
|---|---|
| Empty byte string | Return zero; destination field is zero. |
| Only bytes whose classification low bit is set, followed by zero | Skip those bytes, return zero, and leave the explicitly initialized destination field at zero. |
| `Z` followed immediately by zero | Return zero; destination field is zero; no numeric dependency call. |
| `z` followed immediately by zero | Same result as uppercase `Z`. |
| `Z`, followed by classified bytes, then zero | Return zero after skipping the trailing classified bytes. |
| `ZQ` followed by zero, where `Q` has its classification low bit clear | Return one; the explicit destination value remains zero. |
| An unrecognized nonzero initial byte whose classification low bit is clear | Return one without calling the numeric dependency. |
| `+05:30`, where the numeric dependency returns exactly two with outputs 5 and 30 | Store 330, inspect trailing content from six bytes after the sign, and return zero if that suffix is accepted. |
| `-05:30`, under the same dependency conditions | Store −330 and apply the same suffix rule. |
| `-00:00`, with dependency outputs zero and zero | Store integer zero; no distinct negative-zero representation is established. |
| A signed input for which the numeric dependency returns one, zero, or a negative value | Return one immediately after the dependency call; no numeric destination write by this function. |
| `+05:30Q`, with successful component outputs and an unclassified `Q` | Store 330, then return one because of the trailing byte. |
| `+14:59`, if the dependency accepts it with outputs 14 and 59 | Store 899 and accept an otherwise valid empty or classified-only suffix. |

Inputs such as `+15:00`, `+05:60`, `+5:30`, or `+05-30` cannot be assigned unconditional numeric acceptance outcomes from this function alone. If the dependency rejects them, this function returns one immediately. If it returns exactly two, this function uses its outputs and checks the suffix at the fixed six-byte displacement.

## Uncertainties

The following are **documented_only**, not resolved assumptions:

- The exact byte set identified by the classification low bit.
- Whether the zero byte is guaranteed to have that bit clear.
- Whether the numeric dependency actually enforces two-digit components, the colon, or the apparent bounds.
- Whether its numeric bounds are inclusive.
- Whether a return value of two guarantees initialization of both integer outputs.
- Whether either dependency mutates classification state or other shared memory.
- Whether either dependency retains pointers or manages memory internally.
- The behavior of out-of-range arithmetic beyond the ordinary bounded case.
- Alignment, byte order, and fault behavior of the target environment.
- Safe behavior for overlapping input, destination, and classification storage.
- Concurrency and reentrancy guarantees.
- Any external requirement that an explicit zone marker be present, despite this function accepting an empty remainder.

The likely intent is recognition of a UTC marker or a signed hours-and-minutes offset with optional surrounding whitespace. That interpretation does not strengthen the directly observed checks.

## Evidence

The supplied static representation establishes the dependency call sites, low-bit classification predicate, four-byte destination offset, explicit zero initialization, sign and marker branches, exact dependency-success comparison, fixed six-byte advancement, signed component combination, trailing scan, and one-byte status outcomes.

It also establishes that final suffix validation occurs after the numeric destination write, and that empty input can reach the successful final-byte condition without a marker.

No dependency implementation, runtime trace, target execution, or runtime tests were supplied or performed. Dependency semantics and environmental guarantees not established by the visible operations remain documented uncertainties.
