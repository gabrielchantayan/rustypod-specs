<!-- Trusted provenance: F-14cc18aad1ea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-14cc18aad1ea

## Purpose

F-14cc18aad1ea selects an integer value from shared state and stores the selected value through a shared destination pointer.

The ordinary result is the value read through a shared source pointer. A replacement applies only when both of these conditions hold:

- The input argument is nonzero.
- The source value read by this invocation equals decimal 30.

When both conditions hold, the selected value is obtained from a separate shared replacement location. Otherwise, the initially read source value is selected.

The routine has no explicit return value. Its successful observable result is the destination write. It does not explicitly change the input argument, update either shared pointer, or modify the replacement location.

These resource names describe roles within this contract; they do not establish that the underlying storage locations are distinct.

## Interface

### Input

The routine accepts one argument represented as a signed integer in the supplied source.

Only its zero-versus-nonzero classification affects selection:

- Zero disables replacement.
- Every nonzero value enables the source-value comparison to determine whether replacement is used.
- Negative values are nonzero and therefore enable replacement in the same way as positive nonzero values.

The argument is not a pointer, length, offset, index, or replacement value. Its magnitude does not participate in arithmetic.

### Shared resources

The routine accesses three logical resources:

| Resource | Role | Access |
|---|---|---|
| Source pointer storage and source referent | Identify and supply the initial integer value | Read |
| Replacement location | Supplies the alternate value when the condition qualifies | Conditional read |
| Destination pointer storage and destination referent | Identify the location receiving the selected integer | Pointer read and referent write |

The source and destination each involve indirection: the routine obtains an address from shared storage and accesses the integer at that address. The replacement is used as a shared value; the supplied body does not dereference it as a pointer.

The declarations of these shared resources are not supplied. Their exact declared types, qualifiers, storage durations, alignment guarantees, and relationships to one another are therefore not established.

### Output

The routine returns without an explicit result value.

On normal completion, it has performed one destination assignment. The assignment occurs even if the selected value is equal to the destination’s existing value. There is no destination-value comparison or write-elision condition in the supplied body.

### Integer widths and representation

The argument and the working value are represented using the source language’s signed `int` type. The supplied material does not independently establish the width of that type or of pointers.

Consequently:

- A 32-bit signed interpretation is a plausible platform assumption, not an established interface guarantee.
- Decimal 30 is the exact comparison value. It is representable by the signed integer type independently of whether that type is 16, 32, or more bits wide.
- There is no addition, subtraction, multiplication, shift, or address-offset calculation in the visible selection logic.
- No arithmetic-overflow behavior is required to explain the routine.
- The shared-resource declarations are needed to determine whether reads or the final assignment involve any additional integer conversions.

The contract does not impose a byte order or a fixed four-byte layout without those declarations.

## Behavior

### Initial source read

Every invocation begins by reading the integer identified by the shared source pointer.

This read is unconditional. In particular, a zero argument does not bypass the source access. A caller cannot use a zero argument to make an invalid source pointer harmless.

The value obtained by this read is retained as the initial selected value. The supplied body does not reread the source referent during the condition check or before the destination assignment.

### Replacement decision

After the initial source read, the routine evaluates whether replacement applies.

Replacement requires the conjunction of a nonzero argument and an initial source value equal to decimal 30. A nonzero argument alone is insufficient. A source value of 30 alone is also insufficient when the argument is zero.

The condition has short-circuit semantics: when the argument is zero, the source-value comparison does not determine the branch. This does not remove the earlier unconditional source read.

If the condition qualifies, the routine reads the shared replacement value and uses it instead of the initial source value.

If the condition does not qualify, the replacement read is absent from the supplied execution path. The initially read source value remains selected.

### Destination assignment

After selection, the routine obtains the destination address from shared destination-pointer storage and assigns the selected value to the destination referent.

Thus, at the source-level semantic level, the relevant access order is:

1. Obtain and read the source referent.
2. Read the replacement location, but only if replacement qualifies.
3. Obtain the destination address and write the selected value.

The destination address is not explicitly captured before selection. This distinction can matter if the shared resources overlap or have externally observable access semantics.

There is no subsequent readback, verification, retry, or second destination write in the supplied body.

### Aliasing and changing shared state

The body contains no requirement that source and destination referents be disjoint.

For ordinary, stable integer storage where both pointers identify the same integer:

- The original value is read before the destination write.
- A nonqualifying invocation writes that original value back.
- A qualifying invocation writes the replacement value over it.

The routine does not establish a transaction across its reads. If another actor can change shared resources, the body supplies no locking, explicit atomic operations, or consistency protocol. Whether concurrent access is valid, and what observations it permits, depends on declarations and execution-environment rules not supplied here.

Aliasing involving pointer-storage objects, differently typed storage, or device registers cannot be characterized safely from the body alone.

## Side effects

The explicit successful side effect is one assignment through the shared destination pointer.

The routine also performs the source read on every invocation and the replacement read on qualifying invocations. If those resources have volatile, device-backed, or otherwise externally observable access behavior, their reads may themselves have side effects. Such properties are not established by the supplied material.

No allocation, deallocation, ownership transfer, resource release, logging, callback, or explicit synchronization appears in the body.

Ownership remains unspecified. In particular, the body provides no evidence that the routine owns any referenced storage or may retain it beyond the call. The caller or surrounding environment must provide storage suitable for the accesses that actually occur.

### Preconditions and failure behavior

For normal completion:

- The source pointer storage must be readable.
- Its stored address must permit the initial integer read.
- The destination pointer storage must be readable.
- Its stored address must permit the final integer assignment.
- On the qualifying path, the replacement location must permit its value to be read.

The routine performs no visible null check, bounds check, access validation, or error reporting.

An invalid source access can prevent selection from completing. A failure during a qualifying replacement read can prevent the destination write. An invalid destination can prevent normal completion after the selected value has already been obtained.

The supplied body does not establish a particular exception, signal, error code, or recovery action for invalid accesses. It also does not establish hardware-level atomicity or partial-write behavior.

## Dependencies

No function calls appear in the supplied body. There are therefore no called-function dependencies requiring opaque function IDs.

The routine depends on shared storage and the execution environment for:

- Pointer and integer representations.
- Validity, alignment, lifetime, and accessibility of referenced storage.
- Any qualifiers governing access behavior.
- Any concurrency or device-memory semantics.
- Any conversions implied by the omitted shared-resource declarations.

No unknown callee is assumed to preserve globals, allocate memory, or release memory; there is no visible callee on which to base such assumptions.

## Acceptance examples

The following are conditional acceptance examples, not executed test results. Unless otherwise stated, they assume ordinary stable integer storage, valid accesses, and types that preserve the illustrated values during assignment.

| Argument | Initial source value | Replacement value | Required destination value | Replacement read |
|---|---:|---:|---:|---|
| Zero | 30 | 91 | 30 | No |
| Positive nonzero | 30 | 91 | 91 | Yes |
| Negative nonzero | 30 | 91 | 91 | Yes |
| Positive nonzero | 29 | 91 | 29 | No |
| Positive nonzero | 31 | 91 | 31 | No |
| Zero | −7 | 91 | −7 | No |
| Nonzero | 30 | 0 | 0 | Yes |
| Nonzero | 30 | 30 | 30 | Yes |

Additional acceptance conditions:

- **Nonzero magnitude:** Arguments of 1 and −1 produce the same selection when shared inputs are otherwise identical.
- **Replacement read despite unchanged result:** When the source and replacement values are both 30 and the argument is nonzero, the replacement path still reads the replacement location.
- **Write despite unchanged destination:** If the destination already contains the selected value, normal execution still includes the destination assignment.
- **Source/destination alias:** If both pointers identify the same ordinary integer initially containing 30, a nonzero argument and replacement value of 91 leave that integer containing 91.
- **Unused replacement:** An inaccessible replacement location does not affect a nonqualifying path, provided it has no overlapping-storage complication and all accesses actually performed are valid.
- **Zero does not bypass source access:** A zero argument does not establish normal completion when the source address is invalid. No specific failure mechanism is prescribed.

## Uncertainties

The following remain **documented_only**:

- Exact integer width, pointer width, byte order, and ABI.
- Full shared-resource declarations and any assignment conversions.
- Whether any resource is volatile, atomic, device-backed, or externally mutable.
- Whether logical resources overlap and, if so, whether those accesses are valid.
- Ownership, lifetime management, and synchronization obligations outside the routine.
- Fault delivery and recovery behavior.
- Compiler-generated instruction count and physical memory-access ordering beyond the source-level semantics described above.

**Inferred intent:** The routine appears to copy a shared value while optionally replacing the distinguished value 30. The meaning of 30 and the reason for replacement are unknown. No domain meaning is assigned to either.

## Evidence

**Established static evidence:** The supplied body shows one unconditional indirect source read, a replacement branch requiring a nonzero argument and source value 30, one shared-value read within that branch, and one final indirect destination assignment. It shows no function calls, loops, explicit validation, allocation, or explicit return value.

**Evidence boundary:** This contract is based solely on the supplied function body. Omitted declarations and surrounding behavior have not been supplied or independently verified.

**Runtime status:** No runtime tests have been performed. The acceptance examples specify conditional expected behavior; they are not reports of observed execution.
