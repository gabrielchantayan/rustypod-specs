<!-- Trusted provenance: F-b3e903cfa74f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b3e903cfa74f

## Purpose

F-b3e903cfa74f classifies an unsigned value obtained from dependency F-a6e77b5dfc20. It returns an unsigned Boolean result: 1 when the value remaining after an unsigned right shift by 16 bits equals one of four accepted values, and 0 otherwise.

The accepted shifted values are:

- 12, hexadecimal 0x000C.
- 16, hexadecimal 0x0010.
- 19, hexadecimal 0x0013.
- 20, hexadecimal 0x0014.

Under the recovered 32-bit unsigned interpretation, this is a membership test on the upper 16 bits of the dependency result. The lower 16 bits do not affect classification.

The supplied material does not establish what these four values signify. Interpreting them as status codes, categories, device types, or protocol fields would be inferred intent rather than an established contract.

## Interface

**Inputs:** No explicit arguments.

**Indirect input:** One unsigned integer returned by F-a6e77b5dfc20. There is no explicit input buffer, pointer, length, or caller-provided output location.

**Output:** An unsigned integer whose normal-return value is exactly 0 or 1. This is a numeric Boolean contract, not a declaration that the interface uses a dedicated Boolean type.

**Width and layout:** The supplied representation identifies the dependency result and the function result as unsigned integers. Under its recovered 32-bit interpretation:

- The dependency supplies a 32-bit unsigned value.
- Bits 31 through 16 determine the result.
- Bits 15 through 0 are ignored.
- The returned value is a 32-bit unsigned 0 or 1.

The unsigned type definition and target ABI are not included. Consequently, the 32-bit width is an interpretation of the supplied representation, not an independently verified platform property. The width-independent operation established by the material is an unsigned right shift by exactly 16 bits followed by equality comparisons against 12, 16, 19, and 20.

The distinction matters if the unsigned type is wider than 32 bits: the function does not establish a mask that discards bits above bit 31. Such bits would remain significant after the shift.

**Preconditions:** The dependency must be callable in the surrounding execution environment. For a normal result to be specified, it must return normally with an unsigned integer value. No additional caller-side precondition, validation rule, or initialization requirement is visible.

No calling convention, register assignment, linkage detail, or binary return layout is established.

## Behavior

### Dependency invocation and classification

Each invocation calls F-a6e77b5dfc20 exactly once before determining the result. The returned value is the sole explicit data input to the classification.

The function shifts that unsigned value right by 16 bit positions. This is a logical shift: vacated upper positions are filled with zero rather than a replicated sign bit.

The shifted value is tested for membership in the set containing 12, 16, 19, and 20:

- Membership produces 1.
- Nonmembership produces 0.

All normal-return inputs are covered by these two cases. The function does not return the original dependency result, the shifted value, or an index identifying which accepted value matched.

The positive and negative conditions shown in the supplied material are complementary. There is no normal-return path on which an unclassified intermediate value escapes as the result.

### Bit significance

Under the 32-bit interpretation, the dependency result consists of an upper 16-bit classification field and a lower 16-bit field irrelevant to this function.

Every value with upper half 0x000C, 0x0010, 0x0013, or 0x0014 returns 1, regardless of its lower half. Every other upper-half value returns 0.

The accepted 32-bit input ranges are therefore:

| Inclusive dependency-result range | Result |
|---|---:|
| 0x000C0000–0x000CFFFF | 1 |
| 0x00100000–0x0010FFFF | 1 |
| 0x00130000–0x0013FFFF | 1 |
| 0x00140000–0x0014FFFF | 1 |

These ranges describe integer values, not a byte-oriented memory format. No byte order is established or required by the visible operation because the function receives an integer rather than decoding bytes from memory.

### Arithmetic and boundary behavior

The classification uses a shift and equality comparisons. No addition, subtraction, multiplication, division, or narrowing conversion is visible.

For a 32-bit unsigned value, shifting by 16 yields a value from 0 through 65535. There is no arithmetic overflow in that operation. An original value with its highest bit set is still treated as unsigned; it does not become a negative category.

The accepted values are discrete. Acceptance of 16 does not imply acceptance of nearby values such as 15 or 17. Acceptance of 19 and 20 does not imply acceptance of values above 20.

### Repeated calls

Each invocation obtains a fresh dependency result. No cache, retained classification, or cross-call state is visible in this function.

Identical dependency results produce identical classification results. Repeated calls need not return the same result if the dependency returns different values.

## Side effects

The only explicit outward action is the single call to F-a6e77b5dfc20. The supplied body shows no direct access to caller-owned memory, no explicit global read or write, no allocation, no release, and no external output operation.

The visible ordering is:

1. Invoke the dependency.
2. Receive its unsigned result if it returns normally.
3. Classify that returned value.
4. Return 0 or 1.

The dependency's side effects, if any, occur before this function returns its classification. The supplied material does not establish whether the dependency reads or changes global state, accesses hardware, performs I/O, allocates memory, or releases resources.

Accordingly, this function must not be treated as globally side-effect-free merely because its own classification requires no explicit memory access. Nothing establishes that it restores state changed by the dependency.

There is no visible ownership transfer through the interface. The returned integer is consumed as a value; it is not visibly dereferenced or released. This does not establish ownership rules for resources the dependency might manipulate internally.

No explicit retry, rollback, cleanup, exception handler, or error-reporting operation is shown.

## Dependencies

### F-a6e77b5dfc20

**Role:** Supplies the unsigned value to classify.

**Invocation count:** Exactly one per invocation of F-b3e903cfa74f along the visible execution path.

**Arguments:** No explicit arguments.

**Return use:** The returned unsigned value is shifted right by 16 bits and compared with the four accepted constants.

**Preservation requirements:** None are established. In particular, there is no evidence that this dependency preserves globals, external state, or resource ownership.

**Failure behavior:** The supplied material provides no dependency failure contract. If the dependency does not return normally, this function has no visible mechanism that produces an alternate Boolean result. The exact effects of nonreturn, trapping, or other abnormal control flow remain documented_only.

A dependency result of zero is classified as 0. That fact does not establish that zero denotes a dependency error.

## Acceptance examples

These are conditional acceptance criteria derived from static inspection, not executed tests. Hexadecimal examples below assume the recovered 32-bit unsigned interpretation and normal dependency return.

| Dependency result | Value after shifting right by 16 | Required result |
|---|---:|---:|
| 0x00000000 | 0 | 0 |
| 0x0000000C | 0 | 0 |
| 0x000BFFFF | 11 | 0 |
| 0x000C0000 | 12 | 1 |
| 0x000CFFFF | 12 | 1 |
| 0x000D0000 | 13 | 0 |
| 0x00100001 | 16 | 1 |
| 0x00110000 | 17 | 0 |
| 0x0012FFFF | 18 | 0 |
| 0x00130000 | 19 | 1 |
| 0x0014FFFF | 20 | 1 |
| 0x00150000 | 21 | 0 |
| 0x80000000 | 32768 | 0 |
| 0xFFFFFFFF | 65535 | 0 |

Additional conditional criteria:

- Changing only the lower 16 bits must not change the result.
- A call must obtain its classification input from one dependency invocation, not multiple samples.
- If successive dependency returns are 0x000C0000 and 0x000D0000, the corresponding results must be 1 and 0.
- A normal return must never expose 12, 16, 19, or 20 as its result; accepted inputs return exactly 1.
- Under a wider unsigned interpretation, high-order bits must not be silently discarded before comparison. For example, a shifted value of 65548 is not accepted merely because its low 16 bits equal 12.

## Uncertainties

The following remain documented_only:

- The unsigned type's exact width and ABI representation.
- The semantic meaning of the dependency result and accepted values.
- Whether the dependency reads or modifies shared or external state.
- Dependency allocation, release, and ownership behavior.
- Dependency errors, abnormal termination, blocking, and latency.
- Reentrancy, thread safety, and synchronization requirements.

The apparent intent is to recognize four categories encoded above the lowest 16 bits. That interpretation is consistent with the operation but does not establish a named field, a protocol, or a validity rule for rejected values.

A return of 0 means only that the shifted value is outside the accepted set. It is not independently established as an error, invalid input, or failed operation.

## Evidence

The supplied static representation establishes one dependency invocation, an unsigned right shift by 16, comparisons against four constants, and complementary outcomes yielding exactly 0 or 1.

The 32-bit range examples are conditional consequences of the recovered 32-bit interpretation. They are not evidence of an independently verified ABI.

No runtime tests have been performed. No dependency implementation or external behavioral documentation was supplied. Side effects beyond the visible call, domain semantics, and abnormal-return behavior remain unknown rather than assumed.
