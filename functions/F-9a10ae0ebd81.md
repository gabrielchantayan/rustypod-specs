<!-- Trusted provenance: F-9a10ae0ebd81; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9a10ae0ebd81

## Purpose

F-9a10ae0ebd81 reports whether an address-based record satisfies either of two conditions:

- The integer field at byte offset `0x24` is nonzero.
- If that integer is zero, the pointer field at byte offset `0x28` is non-null.

The result is Boolean. The first condition takes precedence: a nonzero integer produces true without requiring a read of the pointer field.

This is a field-presence predicate, not a record-validity check. A true result does not establish that the record is well formed, that a non-null pointer identifies accessible storage, or that any resource is owned or available. Those interpretations are not established by the supplied evidence.

## Interface

### Input

The function accepts one integer argument used as the base address of a record. It accesses fields relative to that address; it does not treat the argument merely as an integer value to classify.

The relevant record layout is:

| Byte offset from base | Interpretation | Access condition |
|---|---|---|
| `0x24` — decimal 36 | Integer discriminator | Always read during a normally completing invocation |
| `0x28` — decimal 40 | Pointer value | Read only when the discriminator equals zero |

No other record fields are accessed in the supplied body. The layout, size, and meaning of the rest of the record are unknown.

The source representation uses an ordinary integer for both the argument and the discriminator. A 32-bit integer model is consistent with this representation, but the supplied fragment does not independently establish the target ABI, pointer width, byte order, or alignment requirements. Consequently:

- The discriminator must be readable using the target’s integer representation.
- The second field must be readable using the target’s pointer representation when its branch is taken.
- The required extent of the second field depends on pointer width.
- Neither a 64-bit address interface nor a specific pointer encoding is established.

### Output

The function returns a Boolean value:

- **True** when the discriminator is nonzero.
- **True** when the discriminator is zero and the pointer field is non-null.
- **False** when the discriminator is zero and the pointer field is null.

There are no output parameters. The function does not return either field’s value or the address of either field.

### Preconditions

For defined, normally completing behavior, the caller must supply an address for which the required address calculations and memory reads are valid under the target execution model.

At minimum, the discriminator field must be accessible. If its value is zero, the pointer field must also be accessible. Any target-required alignment and object-representation constraints apply.

There is no explicit rejection of a zero base address, a negative integer argument, an out-of-range address, or an incorrectly aligned record. Such inputs are not assigned a safe false result by this contract.

## Behavior

### Decision and read order

The function first obtains the integer stored at base plus `0x24`.

If that value is nonzero, the function returns true. The value need not be positive or equal to one. Every representable nonzero discriminator satisfies this branch, including negative values under a signed interpretation.

If the discriminator is zero, the function obtains the pointer stored at base plus `0x28`. It compares that pointer with null:

- A non-null pointer produces true.
- A null pointer produces false.

The pointer’s target is not accessed. A non-null pointer is sufficient for the predicate, without checking the target’s contents, permissions, lifetime, alignment, or relationship to the record.

### Conditional access

The distinction between reading the pointer field and dereferencing its stored pointer is important:

- Reading the pointer field requires access to storage within the record.
- Dereferencing the stored pointer would require access to the pointed-to storage.
- The supplied function performs the former only when the discriminator is zero and never performs the latter.

When the discriminator is nonzero, the pointer field is not needed by the supplied control flow. Its value does not influence the return value.

This describes source-level accesses and decisions. It does not establish a particular machine-instruction sequence, physical memory transaction count, or speculative-execution behavior.

### Arithmetic and representation boundaries

The offsets are byte displacements of 36 and 40. They are not array indices scaled by the size of an integer or pointer.

The supplied representation forms addresses using integer addition before interpreting them as addresses. It contains no explicit overflow check, address-range check, or normalization.

Defined behavior therefore requires that these address calculations be valid under the intended target semantics. The fragment alone does not justify promising modular wraparound or specifying a result when an address calculation exceeds the supported integer range.

Null is tested as a pointer value. The fragment does not establish the byte encoding of a null pointer, so “null” must not be replaced with an unconditional claim about all-zero storage.

### Repeated calls and concurrent changes

The function does not cache a result or maintain a visible history. Each invocation obtains the discriminator anew and conditionally obtains the pointer field.

The discriminator used for the decision is the value obtained by the initial read. There is no second discriminator read after obtaining the pointer.

No synchronization, atomic access, locking, or consistency protocol is present in the supplied body. A concurrent modification does not receive a defined snapshot guarantee from this function. Whether such modification is permitted, and what results remain defined, depends on the target memory model and surrounding program.

## Side effects

The supplied body contains no explicit writes to the record, the pointer target, global storage, or externally supplied output storage.

It also contains no calls that could independently allocate, release, transfer, or mutate resources.

Ownership is unchanged by the visible operations:

- The function does not take ownership of the record.
- It does not take ownership of the stored pointer.
- It does not release either resource.
- It does not retain either address in persistent storage.

These statements describe the supplied body, not a guarantee that arbitrary reads are harmless. If the supplied address refers to special-purpose memory, a read may have platform-specific effects. No such memory model is established here.

The function provides no explicit error-reporting mechanism. Invalid accesses or invalid address calculations are not converted into a Boolean failure, diagnostic, retry, or recovery action.

## Dependencies

No opaque function dependencies are present. The supplied body contains no calls to other functions.

Its environmental dependencies are:

- The calling convention used to supply the integer argument and receive a Boolean result.
- The integer representation and address-conversion rules.
- The pointer representation and null comparison semantics.
- The accessibility and alignment of the conditionally read fields.
- The surrounding memory model.

There is no unknown callee whose preservation of globals or memory ownership must be assumed.

## Acceptance examples

These examples are conditional contracts, not reports of executed tests. Unless stated otherwise, assume a valid base address, valid field representations, valid arithmetic, and no concurrent mutation.

| Discriminator at `0x24` | Pointer at `0x28` | Required result | Required source-level access behavior |
|---|---|---|---|
| Zero | Null | False | Read discriminator, then pointer field |
| Zero | Non-null | True | Read discriminator, then pointer field; do not dereference the pointer target |
| Positive nonzero | Null | True | Read discriminator; pointer field is not read |
| Negative nonzero | Null | True | Read discriminator; pointer field is not read |
| Nonzero | Non-null | True | Read discriminator; pointer field does not affect the result |

Additional boundary cases:

1. **Nonzero discriminator with inaccessible pointer-field storage.**  
   Provided the discriminator read and required address arithmetic are valid, the supplied branch does not require access to the pointer field. Such inaccessibility alone does not change the specified true result.

2. **Zero discriminator with inaccessible pointer-field storage.**  
   The function attempts to read that storage. No normal return value is guaranteed, and the contract does not prescribe false as a fallback.

3. **Zero discriminator with a representable non-null pointer to inaccessible target storage.**  
   If reading the pointer value itself is valid, the required result is true. The target storage is not accessed.

4. **Inaccessible discriminator storage.**  
   The initial required read cannot be completed under the stated preconditions. There is no prescribed Boolean result.

5. **Changing the pointer while leaving the discriminator nonzero.**  
   For otherwise valid invocations, the result remains true. The pointer field is outside the taken decision path.

6. **Both fields representing absence.**  
   Zero in the discriminator and null in the pointer field are jointly necessary for a normally completing false result.

## Uncertainties

### Documented only

The following cannot be resolved from the supplied fragment:

- Exact integer width, pointer width, byte order, and ABI.
- Valid address range and address-conversion behavior.
- Alignment requirements and enclosing record size.
- The domain meaning of either field.
- Whether the pointer is owning, borrowed, or merely a marker.
- Caller-provided lifetime and synchronization guarantees.
- Platform behavior following an invalid access or arithmetic operation.
- Any effects associated with reads from special-purpose memory.

### Inferred intent

The two-field decision suggests a compact presence or availability check. That interpretation is not needed for the contract and does not justify assigning semantic names, ownership rules, or validity guarantees to the fields.

## Evidence

Established static evidence consists of the supplied function body:

- One integer argument is used as an address base.
- The integer at byte offset `0x24` is read first.
- A zero discriminator causes a pointer read at byte offset `0x28`.
- The result is true for a nonzero discriminator or, on the zero branch, a non-null pointer.
- The stored pointer is never dereferenced.
- No explicit stores or function calls appear.

No runtime tests have been performed. The acceptance examples state consequences of the supplied static control flow; they are not execution observations.
