<!-- Trusted provenance: F-be069eb566cb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-be069eb566cb

## Purpose

F-be069eb566cb evaluates whether a caller-supplied bit mask contains any bit absent from a mask read from caller-addressed storage.

The established result is a Boolean subset predicate:

- **True:** every bit set in the supplied mask is also set in the stored mask.
- **False:** at least one bit set in the supplied mask is clear in the stored mask.

Bits present only in the stored mask do not affect the result. The function does not require equality between the two masks.

Interpreting the masks as requested and available permissions, enabled features, or supported capabilities is plausible intent, not established behavior. The contract assigns no application-specific meaning to individual bits.

## Interface

### Inputs

The function receives two scalar arguments:

1. **Base-address argument.** An integer used to form the address of the stored mask. The supplied representation declares this argument as a signed integer rather than as a pointer.
2. **Candidate-mask argument.** An unsigned integer whose set bits are checked against the stored mask.

The stored-mask address is formed by adding hexadecimal `0x20`, or decimal 32, to the base-address argument and then interpreting the result as the address of an unsigned integer.

This is an integer address calculation, not an array-element offset. Under the conventional byte-addressed interpretation, the stored mask begins 32 bytes after the supplied base address. The function does not establish the layout, size, or meaning of any surrounding object.

### Widths and layout

The candidate mask and stored mask have the same declared unsigned-integer type. Let **W** be the number of value bits in that type. The predicate operates over those W bits.

The supplied evidence does not independently establish the target ABI or the widths of its integer and pointer types. A 32-bit unsigned interpretation is consistent with the representation, but exact four-byte storage and 32-bit mask semantics remain conditional on that interpretation. The acceptance examples explicitly state when they assume W = 32.

The relevant layout is therefore:

| Location | Interpretation |
|---|---|
| Base address | Starting value used for address formation; not itself dereferenced |
| Base address plus `0x20` | Beginning of one unsigned-integer stored mask |
| Other locations | No explicit access established |

The code does not define byte order, alignment requirements, packed storage, or a portable serialization format.

### Output

The function returns a Boolean value. The logical distinction between true and false is established; the physical representation, register convention, or serialized width of the Boolean is not.

No output buffer, updated mask, error object, or ownership token is returned.

### Preconditions

For the defined normal behavior described here:

- Adding `0x20` to the signed base-address argument must be valid under the applicable integer model.
- Conversion of the resulting integer to an address must identify readable storage.
- The addressed storage must support the unsigned-integer read, including any applicable alignment, object-lifetime, and access requirements.
- The read must have well-defined behavior under the applicable memory and concurrency model.

The function supplies no explicit validation of these conditions. A candidate mask of zero does not remove the stored-mask read from the expressed computation and must not be treated as permission to provide an invalid address.

## Behavior

### Address formation and stored-mask access

The function forms one address from the base-address argument and the fixed displacement `0x20`. It then reads one unsigned-integer value at that address.

There is one explicit stored-mask read in the supplied expression. No second mask read, retry, range check, or alternate location is established.

The source-level read is not declared volatile or atomic. Consequently, this contract does not promise one particular machine instruction, a transaction visible to another device, or a synchronization event. Compiler and target-memory rules may affect the physical access.

### Predicate

Let:

- **C** be the candidate-mask value;
- **S** be the value obtained from the stored-mask read;
- **U** be the mask with all W value bits set.

The result is true exactly when:

**C AND (U XOR S) = 0**

The complement applies across the width of the unsigned mask type. It is not an unbounded mathematical complement and does not introduce meaningful bits beyond that width.

Equivalently, the result is true exactly when **C AND S = C**. This equivalence describes the result; it does not claim that the function performs another stored-mask read or uses a different implementation.

The predicate has these consequences:

- A zero candidate mask produces true after a valid stored-mask read.
- A zero stored mask accepts only a zero candidate mask.
- An all-ones stored mask accepts every candidate representable in the same width.
- Extra set bits in the stored mask are harmless.
- One missing candidate bit is sufficient for false.
- If one candidate bit is missing while other candidate bits are present, the result remains false.
- The highest representable mask bit participates normally; it is not treated as a sign bit of the candidate mask.

### Ordering and completion

The stored-mask value is required to determine the expressed comparison. The Boolean result follows evaluation of that comparison.

There are no explicit function calls, writes, or other reads with which the stored-mask access must be ordered. No stronger memory-ordering or cross-thread visibility guarantee is established.

The function returns false for a failed bit-subset check. False is not an error report for unreadable storage, arithmetic overflow, or an invalid address.

## Side effects

No explicit memory write, allocation, deallocation, ownership transfer, logging, callback, or persistent-state update appears in the supplied body.

The base-address argument is used to locate storage; receiving it does not establish that the function owns that storage. No requirement to release the storage follows from the call. The stored mask is consumed as an input value, not modified.

No explicit global-variable access is established. Nevertheless, caller-addressed storage could itself reside in global memory, shared memory, or another mapped region. The function does not identify or distinguish these cases.

If reading the selected location has platform-specific effects—for example, because the location is a device register—those effects are outside the established ordinary-memory contract. The supplied representation does not establish suitability for such memory.

Concurrent mutation is likewise not handled explicitly. No lock, atomic operation, snapshot protocol, or retry protects the read. Defined concurrent behavior requires guarantees outside this function.

## Dependencies

No dependency on another callable function is established by the supplied body. There are therefore no additional opaque function IDs to list.

The behavior relies on the execution environment for:

- Integer widths and signed-arithmetic rules.
- Integer-to-address conversion.
- Address validity and alignment.
- Unsigned-integer storage interpretation.
- Memory-access and concurrency semantics.
- Boolean return conventions.

These are environmental prerequisites, not evidence of hidden helper calls. No assumption about an unknown callee preserving global state or owning memory is needed.

## Acceptance examples

The following are conditional acceptance cases, not executed tests. All assume valid address formation and a defined stored-mask read. Hexadecimal examples assume W = 32 and a matching stored unsigned-integer representation.

| Stored mask | Candidate mask | Required result | Reason |
|---|---|---|---|
| `0x00000000` | `0x00000000` | True | The candidate contains no set bits. |
| `0x00000000` | `0x00000001` | False | Candidate bit 0 is absent from storage. |
| `0x0000000F` | `0x00000005` | True | Both candidate bits are present. |
| `0x00000005` | `0x0000000F` | False | Some candidate bits are absent. |
| `0x00000005` | `0x00000005` | True | Equal masks satisfy the subset predicate. |
| `0xFFFFFFFF` | `0x80000001` | True | An all-ones stored mask contains both candidate bits. |
| `0x7FFFFFFF` | `0x80000000` | False | The highest candidate bit is absent. |
| `0x80000000` | `0x80000000` | True | The highest bit participates as an unsigned mask bit. |
| `0xFFFFFFFF` | `0xFFFFFFFF` | True | Every candidate bit is present. |

Additional boundary conditions:

- **Offset selection:** If otherwise valid storage contains different masks at the base address and at base plus `0x20`, the result is determined by the latter location. The value at the base address is not an input to the established predicate.
- **Irrelevant stored bits:** Changing only stored bits that are clear in the candidate cannot change the result, provided the read remains defined.
- **Zero candidate with invalid storage:** No successful return is guaranteed. The expressed computation includes the stored-mask read.
- **Address-arithmetic overflow:** No wrapped-address result is specified. The supplied body does not establish a wrapping arithmetic contract.

## Uncertainties

The following remain **documented_only**:

- Exact signed-integer, unsigned-integer, pointer, and Boolean ABI widths.
- Whether the intended deployment uses the apparent 32-bit mask model.
- Concrete object type, allocation size, lifetime, and ownership surrounding the stored mask.
- Required alignment and target byte order.
- Valid ranges or restrictions on the base-address argument.
- Behavior when signed address addition is outside its representable range.
- Behavior when integer-to-address conversion does not yield valid readable storage.
- Fault delivery or recovery for inaccessible memory.
- Concurrency guarantees supplied by callers or the surrounding application.
- Whether the addressed location can be volatile, atomic, or device-backed.
- Application-specific meanings assigned to the mask bits.

Inferred intent is limited to a containment check between a candidate mask and an available mask. Permission enforcement, capability negotiation, and feature gating are possible uses, not established requirements.

## Evidence

The supplied static representation establishes a signed integer base-address argument, an unsigned candidate-mask argument, a fixed `0x20` address displacement, one unsigned stored-mask read, a bitwise-complement-and-intersection predicate, and a Boolean comparison against zero.

The representation contains no explicit call, write, allocation, release, validation branch, or error-handling path. These observations describe the supplied body, not verified machine-level behavior under every compiler or environment.

No runtime tests have been performed. The acceptance examples are consequences of the established bit-mask predicate under their stated preconditions, not observed execution results.
