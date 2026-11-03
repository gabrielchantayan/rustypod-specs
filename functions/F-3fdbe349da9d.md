<!-- Trusted provenance: F-3fdbe349da9d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3fdbe349da9d

## Purpose

F-3fdbe349da9d scans a byte-addressed region and returns a 32-bit success or failure indicator. It stops successfully when a zero byte is encountered at the current scan position. Some bytes with their high bit set cause the function to inspect and consume a two- or three-byte group.

The established behavior is narrower, and less strict, than general UTF-8 validation:

- Nonzero bytes below `0x80` are accepted individually.
- Bytes from `0xC0` through `0xDF` require one following continuation-shaped byte.
- Bytes from `0xE0` through `0xEF` require two following continuation-shaped bytes.
- Every other byte with its high bit set consumes a three-byte group without validating the two following bytes.

A continuation-shaped byte is any byte from `0x80` through `0xBF`, inclusive.

**Inferred intent:** the function resembles a zero-terminated multibyte-text checker. That interpretation does not establish UTF-8 conformance, Unicode scalar-value validity, or safe handling of an ordinary zero-terminated string. In particular, some paths read beyond an embedded zero rather than stopping at it.

## Interface

### Input

The function receives one pointer to bytes.

The input has no accompanying length, capacity, encoding designation, or ownership flag. Each inspected element is an 8-bit byte. Byte comparisons and bit classifications use values in the range `0` through `255`.

The required readable extent depends on the values encountered during scanning. A zero byte somewhere in the region is not, by itself, sufficient to make the operation memory-safe: the function recognizes zero as a terminator only when that byte is read at a scan position, not when it is read as a following byte within a group.

For defined completion, the caller must provide:

- A valid readable address for every byte the function accesses.
- Valid pointer advancement through the accessed region.
- Stable enough input for the observed reads to have meaningful values.
- A scan path that eventually reaches either a recognized terminator or a rejecting group, unless nontermination is acceptable.

The supplied material does not establish the address width, pointer-overflow behavior, or a mechanism for enforcing these preconditions.

### Output

The return value is 32 bits wide:

| Value | Meaning |
|---|---|
| `1` | A zero byte was encountered at the current scan position before any rejecting check. |
| `0` | A required continuation-shape check failed. |

For executions that return normally under the visible behavior, no other return value is produced.

A return value of `1` does not certify that the bytes form valid UTF-8, that every nonzero byte was validated, or that the first zero in address order terminated the scan.

### Ownership and mutation

The function does not visibly write to the input region, allocate memory, release memory, or transfer ownership. The caller retains responsibility for the storage and its lifetime.

The scan position is internal. No updated pointer or consumed-length value is returned.

## Behavior

### Scan-position processing

At each scan position, the function first reads exactly one byte.

If that byte is zero, the function immediately returns `1`. No following byte is read on that iteration.

If the byte is nonzero and below `0x80`, the function advances the scan position by one byte and repeats. All values from `0x01` through `0x7F` receive the same treatment; there is no additional distinction for control characters, whitespace, or printable characters.

A byte from `0x80` through `0xFF` selects grouped processing. In every such case, the function next reads the byte one address after the scan position. This read occurs before the group is accepted or rejected.

### Two-byte groups

A scan-position byte from `0xC0` through `0xDF`, inclusive, selects a two-byte group.

The following byte must be in `0x80` through `0xBF`. If it is outside that range, the function returns `0`. If it is inside that range, scanning resumes two bytes after the original scan position.

Only those two bytes are read while processing this group. The function does not inspect a third byte until the next scan iteration.

The first-byte range includes `0xC0` and `0xC1`. No check excludes those values, and no decoded character value is calculated. Consequently, a two-byte group can pass this check even when a UTF-8 interpretation would classify it as an overlong encoding.

A zero following byte is not a successful terminator on this path. It fails the continuation-shape check and produces `0`.

### Three-byte groups with validation

A scan-position byte from `0xE0` through `0xEF`, inclusive, selects a three-byte group with validation.

The read order is significant:

1. The scan-position byte is read.
2. The byte at the next address is read.
3. The byte two addresses after the scan position is read.
4. The second byte is checked for continuation shape.
5. If that check succeeds, the third byte is checked for continuation shape.

Thus, the third byte is read even when the second byte will cause rejection.

If either checked byte is outside `0x80` through `0xBF`, the function returns `0`. If both satisfy that range, scanning resumes three bytes after the original scan position.

There are no additional restrictions specific to `0xE0`, `0xED`, or any other first byte within this class. The function does not reject overlong forms or surrogate-range encodings when their following bytes satisfy the continuation-shape tests.

A zero in either following position is not interpreted as a terminator. It causes failure of the corresponding continuation-shape check, provided all required reads are valid.

### Three-byte groups without validation

The remaining high-bit-set first bytes are:

- `0x80` through `0xBF`.
- `0xF0` through `0xFF`.

For these values, the function reads the next two bytes and then resumes scanning three bytes after the original scan position. It does not validate either following byte.

This behavior has several consequences:

- A continuation-shaped byte can appear at a scan position without causing immediate rejection.
- A first byte conventionally associated with a four-byte UTF-8 sequence is not processed as a four-byte group.
- Values `0xF5` through `0xFF` receive the same unchecked three-byte treatment.
- Zero bytes in either following position are read and skipped, not recognized as terminators.
- Arbitrary following byte values can be accepted as part of the skipped group.

This branch does not report failure. Any eventual failure must arise from a later validated group.

### Advancement and termination

Every completed nonterminating iteration advances the scan position by one, two, or three bytes. The function does not move backward or revisit earlier positions as part of its visible algorithm.

The offsets used within a group are the fixed byte offsets one and two. Group selection uses 8-bit masks and comparisons, not signed-byte ordering. No character-value accumulation, length counter, or variable-width integer decoding is visible.

There is no explicit bound on the number of iterations. A region without a reachable scan-position zero or a rejecting group can cause continued scanning until an invalid access or another environment-dependent outcome occurs. The function has no explicit length-exhaustion result.

## Side effects

The visible implementation performs input reads and internal pointer updates. It contains no explicit writes to caller storage, global state, or output parameters.

The observable read order for each class is:

| Scan-position byte | Reads before a result or next iteration |
|---|---|
| `0x00` | Current byte only; then success. |
| `0x01`–`0x7F` | Current byte only; then advance by one. |
| `0xC0`–`0xDF` | Current byte, then next byte; then check and either fail or advance by two. |
| `0xE0`–`0xEF` | Current byte, next byte, then third byte; then checks and either failure or advancement by three. |
| `0x80`–`0xBF`, `0xF0`–`0xFF` | Current byte, next byte, then third byte; then unconditional advancement by three. |

These are the read operations established by the supplied static representation. Machine-level access granularity, speculative execution, and compiler or processor ordering guarantees are not established.

Invalid memory access is not translated into a `0` result by any visible mechanism. A fault, process termination, or other environment-specific response is outside the established normal-return contract.

## Dependencies

No calls to other functions are visible. No opaque dependency IDs other than F-3fdbe349da9d are needed to describe the established behavior.

The function relies on its execution environment for readable memory and valid pointer operations. There is no visible dependency that supplies a length, validates an address, decodes characters, or handles access faults.

No assumption about unknown callees preserving globals or owning memory is necessary, because no callee is present in the supplied behavior.

## Acceptance examples

The following are conditional expectations derived from static evidence, not executed tests. Byte sequences are written in hexadecimal and listed in increasing address order.

Except where an access limitation is the subject of the example, assume that all listed storage is readable, pointer advancement is valid, and the bytes remain unchanged during the call.

| Input bytes | Expected result | Reason |
|---|---|---|
| `00` | `1` | The initial scan-position byte is zero. No later byte is read. |
| `41 7F 00` | `1` | Both nonzero low-half bytes are consumed individually; the next scan-position byte terminates the scan. |
| `C2 80 00` | `1` | The two-byte group passes its one continuation check, then scanning reaches zero. |
| `C0 80 00` | `1` | The first byte is accepted as a two-byte leader despite the overlong UTF-8 interpretation. |
| `DF BF 00` | `1` | Both endpoints satisfy the two-byte group rules. |
| `C2 41` | `0` | The second byte is not continuation-shaped. No third byte is needed. |
| `C2 00` | `0` | Zero is checked as a following byte, not as a terminator. |
| `E1 80 BF 00` | `1` | Both following bytes satisfy the three-byte checks. |
| `E0 80 80 00` | `1` | No shortest-form constraint is imposed on the group. |
| `ED A0 80 00` | `1` | No surrogate-range exclusion is imposed. |
| `E1 41 80` | `0` | The third byte is read before the invalid second byte causes rejection. |
| `E1 80 00` | `0` | The third byte fails the continuation-shape check. |
| `80 41 42 00` | `1` | The initial byte selects an unchecked three-byte group; scanning next reaches zero. |
| `FF 00 00 00` | `1` | The first two zeros are skipped within an unchecked group; the final zero terminates. |
| `80 00 00 C2 41` | `0` | The unchecked group skips two zeros, after which the two-byte group fails. |

Two boundary cases require particular care:

- **Only `E1 41` is readable:** normal rejection is not established. Although the second byte is invalid, the function attempts to read the third byte before performing that rejection check.
- **Only `F0 90 80 80 00` is readable:** the final zero does not guarantee safe completion. The first group consumes offsets zero through two. At offset three, `0x80` selects another unchecked three-byte group, which reads offsets four and five. Offset five is outside the stated readable extent.

These cases distinguish the actual access contract from a conventional bounded validator or a scanner that always stops at the first zero in address order.

## Uncertainties

### Established static evidence

The byte classes, normal return values, relative read order, continuation-shape checks, and scan-position advancements are directly supported by the supplied representation.

The absence of explicit writes, allocation, deallocation, and function calls is also established within that representation.

### Inferred intent

The resemblance to text-encoding validation is an interpretation of the byte patterns. It does not justify strengthening the contract into Unicode or UTF-8 validation.

The success indicator can reasonably be understood as acceptance by this scanner, but not as an encoding-validity guarantee beyond the stated checks.

### documented_only unknowns

The following remain unknown and must not be filled in by assumption:

- The intended encoding and the caller’s meaning for success.
- Whether unchecked three-byte consumption is intentional or erroneous.
- The target pointer width, ABI details beyond the visible 32-bit result, and exact address-overflow semantics.
- The storage size and lifetime supplied by any caller.
- The behavior of the surrounding environment on invalid access.
- Concurrency guarantees for input that another agent can modify.
- Whether the static representation faithfully captures every machine-level memory-access property.

There is no established null-pointer handling. A null or otherwise invalid initial pointer does not have a defined failure return under this contract.

## Evidence

This contract is based solely on the supplied static representation of F-3fdbe349da9d. Its conclusions follow from the initial zero check, high-bit classification, two leader-class masks, ordered following-byte reads, continuation-shape tests, and fixed scan advancements.

No runtime tests have been performed. The acceptance examples are conditional static predictions, not observed execution results. No external encoding specification, caller behavior, or unstated dependency contract has been used to override the visible behavior.
