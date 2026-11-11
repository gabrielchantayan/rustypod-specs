<!-- Trusted provenance: F-db9cc30e2b4e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-db9cc30e2b4e

## Purpose

F-db9cc30e2b4e partially initializes an 80-byte output record using a selected 46-byte descriptor, an indexed auxiliary address, and a byte buffer already referenced by the output record. It scans the buffer for recognized identifiers, combines descriptor defaults with scan-derived overrides, and may copy and convert payload records into a baseline region and an allocated linked list.

These are established structural roles. Interpreting the operation as parsing a particular external format is inferred intent; the format’s name and semantics are not established.

The function has two explicit completion statuses:

- **0:** the signed index is not less than the descriptor count, and that count is less than one.
- **1:** execution reaches the end of the initialization path.

Status 1 does not certify bounds validity, successful allocation, successful conversion, or complete input validation. Exceptions, faults, and nonlocal dependency failures have no established handling here.

## Interface

### Arguments and result

The function takes four 32-bit arguments:

| Argument | Established role |
|---|---|
| Context address | References descriptor metadata, an auxiliary-address array, and a persistent flag |
| Index | Used in signed comparisons and in output, descriptor, and auxiliary-array address calculations |
| Output-array address | Base address of records with an 80-byte stride |
| Additional word | Initially stored in temporary storage subsequently covered by an eight-byte copy |

The selected output-record address is the output-array address plus the original index multiplied by 80. Descriptor fallback does not change this index.

The result is 64 bits:

- Low 32 bits: completion status, 0 or 1.
- High 32 bits: the original index’s 32-bit bit pattern.

The whole result is therefore not a Boolean status. For example, a nonzero index can produce a nonzero 64-bit result even when the low-word status is zero.

### Context and descriptor layout

| Context offset | Width | Use |
|---|---:|---|
| +0x38 | 1 byte | Persistent flag, conditionally written as 1 |
| +0x90 | 4 bytes | Descriptor-array address |
| +0x94 | 4 bytes | Signed descriptor count |
| +0x98 | 4 bytes | Address of an array of 32-bit auxiliary addresses |

Descriptor stride is 46 bytes. The accessed descriptor fields are:

| Descriptor offset | Width | Use |
|---|---:|---|
| +0x10 | Address passed to dependency | Flag-related interpretation |
| +0x14, +0x15 | 1 byte each | Output bytes at +0x3c and +0x3d |
| +0x16 | 4 bytes | Raw copy to output +0x4c |
| +0x1a | 8 bytes | Initial override/default storage |
| +0x26 | 2 bytes | Baseline field at output +0x30 |
| +0x28, +0x29 | 1 byte each | Baseline bytes at +0x32 and +0x33 |
| +0x2a | 4 bytes | Raw copy to output +0x34 |

### Output-record layout

| Output offset | Width | Established role |
|---|---:|---|
| +0x00 | 4 bytes | Existing length or end-address contribution |
| +0x04 | 4 bytes | Existing buffer address |
| +0x0c | 4 bytes | Decoded unsigned big-endian 16-bit buffer prefix |
| +0x10 | 4 bytes | Buffer address plus 2 |
| +0x20 | 1 byte | Flag conditionally written as 1; not explicitly cleared |
| +0x2c–+0x37 | 12 bytes | Baseline initialization and optional conversion region |
| +0x38 | 4 bytes | List head, explicitly cleared and possibly replaced |
| +0x3c, +0x3d | 1 byte each | Descriptor-derived bytes |
| +0x40 | 4 bytes | Captured auxiliary address plus 8 |
| +0x44–+0x4b | 8 bytes | Default or override words |
| +0x4c–+0x4f | 4 bytes | Descriptor-derived raw bytes |

Within the baseline region, +0x2c and +0x2e receive separate 16-bit zero writes. Offset +0x30 receives a 16-bit descriptor value, +0x32 and +0x33 receive bytes, and +0x34 receives four copied bytes.

Bytes outside the listed writes are not explicitly initialized by this function.

### Preconditions

The normal path has no explicit null, bounds, alignment, allocation-result, or arithmetic-overflow checks. Safe execution requires valid storage for every access actually reached, including:

- The selected output record and its existing buffer.
- The selected descriptor and original-index auxiliary entry.
- Identifier and payload bytes accessed during scanning.
- Every dependency’s actual readable and writable extent.
- Returned node storage covering the requested allocation size.

The loop’s cursor-versus-end test does not establish that an identifier or its payload fits within the buffer.

Negative indices are not rejected. Address calculations use the represented 32-bit address model; the function does not establish safe behavior when calculations leave valid storage or wrap.

## Behavior

### Descriptor selection and early completion

The function computes the output-record address, then reads the signed descriptor count.

If the signed index is less than that count, it selects the descriptor at the original index. This branch does not require either operand to be nonnegative.

Otherwise:

- A count less than one produces status 0.
- A positive count selects the last descriptor.

Only descriptor selection falls back. Output-record selection and auxiliary lookup retain the original index.

On the explicit status-0 path, no output-record dereference, output write, or dependency call is shown. Computing the output address does not itself initialize that record.

### Captured auxiliary address and defaults

The function reads the auxiliary-array address from the context and reads its original-index 32-bit entry.

It then invokes the byte-copy dependency to copy eight bytes from descriptor +0x1a into temporary storage. Under the apparent copying contract, this replaces the temporary words initially containing the output-array argument and additional argument. The additional argument has no subsequent independent use.

The captured auxiliary address and copied defaults precede the flag-related calls. They are not refreshed afterward. Later descriptor reads are separate accesses and must not be treated as part of the same snapshot.

### Flag-related calls and writes

F-4055be9c3f18 is called twice, each time with descriptor address +0x10.

After the first call, either bit 31 or bit 30 in its 32-bit result causes a one-byte write of 1 to context +0x38. Otherwise, that byte is untouched.

After the second call:

- If result bit 30 is set, output +0x20 is written as 1 without reading the context flag for that decision.
- If bit 30 is clear, context +0x38 is read. A nonzero byte causes output +0x20 to be written as 1.
- Otherwise, output +0x20 is untouched.

The results of the two calls are independent observations. The second call occurs after the possible context write and may have undocumented effects on the later flag read.

Neither flag is generally reset to zero. A previously nonzero output flag can survive a call in which no condition requests setting it.

### Buffer setup

The first two bytes referenced by the output’s existing buffer-address field are decoded as an unsigned big-endian 16-bit integer. The resulting value, between 0 and 65535, is stored in the 32-bit field at output +0x0c.

The function then writes, in order:

1. Captured auxiliary address plus 8 to output +0x40.
2. Buffer address plus 2 to output +0x10.

The initial extension cursor is formed from the buffer-address field plus output +0x0c plus 2.

These uses of the buffer-address field are accesses to output storage, not an established immutable snapshot taken before all preceding calls and writes. The ordering of individual byte reads within the prefix-decoding expression is not separately established.

### Extension-loop condition and identifier reads

Before every iteration, including the final failed condition evaluation, the loop rereads both:

- The buffer-address field at output +0x04.
- The length contribution at output +0x00.

Their sum supplies that evaluation’s end address. Scanning continues only while the current cursor is below that end address. The bound is not captured once for the entire scan.

Each entered iteration decodes bytes at cursor +4 through +7 into one unsigned big-endian 32-bit identifier. That decoded value is then compared against global values in this schedule:

1. Read the **override global** and compare it with the decoded identifier.
2. Only after an override mismatch, read the **sequence global** and compare it.
3. Only after both earlier comparisons fail, read the **flag global** and compare it.

Thus the three globals are neither an established constant set nor an up-front snapshot. A matching override suppresses the other two reads; a matching sequence suppresses the flag-global read. Each subsequent iteration performs the applicable reads again.

If global values coincide at the times they are read, branch precedence is override, then sequence, then flag. Aliasing or undocumented state changes can matter to both global reads and repeated loop-bound reads.

### Recognized override

The function decodes four unsigned big-endian 16-bit quantities from cursor +8 through +15.

They replace two temporary 32-bit words:

- First quantity: low 16 bits of the first word.
- Second quantity: high 16 bits of the first word.
- Third quantity: low 16 bits of the second word.
- Fourth quantity: high 16 bits of the second word.

The next cursor is the current cursor plus 16.

A later override replaces both earlier words. This constructs integer values; it is not an eight-byte raw payload copy. The byte representation of their later ordinary 32-bit stores depends on native byte order.

### Recognized sequence

Bytes at cursor +8 and +9 are decoded big-endian, then interpreted as a signed 16-bit count and widened to 32 bits.

The saved payload address becomes cursor +10. Its first two bytes are decoded big-endian and retained as a signed 16-bit comparison value.

The next cursor is the saved payload address plus the signed count multiplied by 12.

The comparison value is read even when the count is zero or negative. No check first establishes the existence of a complete record.

A later sequence replaces the saved payload address, count, and comparison value. Other extension types do not clear them.

### Recognized flag and unrecognized identifiers

A recognized flag writes 1 to output +0x20 and advances the cursor by 8.

An unrecognized identifier advances the cursor by 4.

The unrecognized case is not a length-delimited skip: it reads identifier bytes at +4 through +7 despite advancing by only four bytes.

### Initialization after scanning

After the loop terminates, the function performs these operations in order:

1. Copies descriptor bytes +0x14 and +0x15 to output +0x3c and +0x3d.
2. Invokes the byte-copy operation for four bytes from descriptor +0x16 to output +0x4c.
3. Stores the final temporary words at output +0x44 and +0x48.
4. Writes 16-bit zeros at output +0x2c and +0x2e.
5. Reads a 16-bit value at descriptor +0x26 and writes it at output +0x30.
6. Copies descriptor +0x29 to output +0x33, then descriptor +0x28 to output +0x32.
7. Invokes the byte-copy operation for four bytes from descriptor +0x2a to output +0x34.
8. Writes a null 32-bit list head at output +0x38.

The descriptor accesses in this phase occur after scanning and the earlier dependency calls. Aliasing and dependency side effects can therefore distinguish them from the defaults captured earlier.

### Baseline comparison and optional conversion

Sequence processing occurs only if the saved payload address is nonzero. A saved address that becomes zero through address arithmetic does not pass this guard.

The baseline comparison uses a **32-bit read at output +0x2c**, followed by conversion of that value to signed 16 bits. The resulting value is compared with the saved signed 16-bit payload value.

The conversion selects the loaded word’s low 16 bits and interprets them as signed. It is not expressed as a signed 16-bit memory read at +0x2c. Because native byte order is unresolved, those low bits cannot generally be identified with the two bytes at +0x2c.

With ordinary copying, disjoint storage, and no interfering dependency effects, the two preceding zero writes make all four bytes of this loaded word zero. In that conditional case, the comparison value is zero regardless of native byte order.

If the comparison matches:

- Twelve bytes from the saved payload are copied over the baseline region.
- F-064272b39d1f is called with the baseline-region address.
- The 32-bit signed count is reduced by one.
- Subsequent node copying starts with payload record two.

If it does not match, the baseline is retained, the count is unchanged, and node copying starts with payload record one.

The comparison and possible baseline copy do not require a positive count. Decrementing a count of −32768 produces −32769 in the 32-bit count; it does not wrap as a 16-bit decrement.

### Allocated list construction

If the resulting count is positive, F-fab3799b3190 is called with count multiplied by 16. Its 32-bit result is written to output +0x38 and used as the base of contiguous 16-byte nodes.

For each node, in increasing order:

1. Copy 12 bytes from the selected payload record into the node.
2. Call F-064272b39d1f with the node address.
3. Write the node’s final four bytes as the next node’s address, or zero for the last node.

The link write follows conversion and overwrites any converter result in those four bytes. The caller does not check the allocation result before using it.

The maximum positive count is 32767, so the largest explicit request is 524272 bytes. This size bound does not establish allocation success or pointer validity.

After this path, or when no positive node count remains, the function returns status 1.

## Side effects

Established direct effects are partial output initialization, conditional flag writes, list-head replacement, node writes, and dependency calls.

The function does not explicitly release an old list before clearing its head. It shows no explicit release of newly obtained storage. Ownership transfer, old-list disposal, and eventual release obligations remain **documented_only**.

No rollback is shown. A fault or nonlocal failure may occur after flags, output fields, or nodes have already been modified.

Aliasing is not rejected. Context, descriptors, buffer, output, global identifier storage, and returned storage must not be assumed disjoint. Later reads can observe earlier writes where storage overlaps. Dependency calls must not be assumed to preserve globals or unrelated memory, or to acquire or release ownership.

## Dependencies

| Dependency | Established interaction | Unresolved contract |
|---|---|---|
| F-4055be9c3f18 | Two calls with descriptor +0x10; selected result bits control flags | Meaning, determinism, mutation, failures, ownership effects |
| F-064272b39d1f | Called after a 12-byte copy into the baseline or a node | Conversion semantics, accessible extent, global effects, failures, ownership effects |
| F-fab3799b3190 | Called with a positive multiple of 16; result used as writable storage | Allocation behavior, failure convention, initialization, ownership, release |

An additional byte-copy dependency has no supplied opaque function ID. Calls request 4, 8, or 12 bytes. Its apparent role is copying the specified ranges; overlap handling and additional side effects are not established. No identifier or stronger contract is invented for it.

## Acceptance examples

These are conditional static expectations, not runtime observations. Unless stated otherwise, assume valid disjoint storage, ordinary byte copying, stable identifier globals, and no interfering dependency effects.

1. **Empty collection:** Index 0 and count 0 produce status 0 and high word 0, without output initialization.

2. **Fallback is descriptor-only:** Count 2 and index 5 select descriptor 1 while using output record 5 and auxiliary entry 5.

3. **Negative index bypasses empty rejection:** Count 0 and index −1 take the indexed-descriptor branch. The function provides no protection for the resulting accesses.

4. **Preserved flag:** If output +0x20 starts as 7, neither call requests setting it, the context flag is zero, and no flag extension occurs, it remains 7.

5. **Independent flag results:** A first result with bit 31 set writes the context flag. A second result with bit 30 clear then causes an output-flag write if it leaves the context flag nonzero.

6. **Conditional global reads:** An override match reads no sequence or flag global during that iteration. A sequence match reads the override and sequence globals but not the flag global.

7. **Fresh loop bound:** If the output length changes between condition evaluations, the next evaluation uses the changed length rather than the initial one. The same applies to the buffer-address field.

8. **Unsigned prefix:** Prefix bytes 0x01 and 0x02 yield 258 at output +0x0c and, with an unchanged buffer address, an initial cursor 260 bytes beyond its base.

9. **Word construction:** Override quantities 0x1122, 0x3344, 0x5566, and 0x7788 produce words 0x33441122 and 0x77885566. Their stored byte order remains native-order dependent.

10. **Baseline consumption:** Count 3 and first comparison value zero match an undisturbed zero baseline word. One payload record is copied and converted in place; two nodes require 32 bytes, with the first linked to the second and the second to zero.

11. **No baseline consumption:** Count 2 and a nonzero comparison value retain the undisturbed baseline and place both records into a 32-byte allocation.

12. **Zero-count sequence:** A zero count still permits the comparison-value read. If it matches, one baseline record is copied and converted, the count becomes −1, and no nodes are requested.

13. **Read width matters:** If interference makes the baseline’s 32-bit word nonzero before comparison, acceptance depends on that word’s signed low-16-bit conversion—not an assumed two-byte read at +0x2c.

14. **Unchecked allocation failure:** A zero allocation result for a positive count is stored as the head and then used as the node base. No graceful failure status is established.

## Uncertainties

The following remain **documented_only**:

- Numeric identifier values and external format semantics.
- Dependency internals, hidden state changes, failure behavior, and ownership effects.
- Native byte order for ordinary multibyte accesses.
- Alignment requirements and overlapping-copy behavior.
- Ordering of operand reads within expressions where not separately established.
- Intended behavior for truncated, malformed, negative-count, or aliasing inputs.
- Ownership and release obligations for existing and new lists.

Sequence counts range from −32768 through 32767 before optional decrement. Negative displacement can move scanning backward, repeat parsing, access invalid storage, or prevent termination. The function does not reject it.

The decoded prefix does not replace the output length field. That field’s semantic meaning is established only by its repeated use in the loop-end calculation.

## Evidence

This contract is based solely on the supplied static representation and dependency mapping. Established claims concern visible access widths, offsets, signed conversions, byte decoding, conditional read schedules, call order, writes, arithmetic, and return construction.

“Descriptor,” “extension,” “baseline,” and “node” are structural labels, not verified external-format terminology. Parser-like intent is inferred; unknown behavior remains explicitly documented rather than supplied by assumption.

No runtime tests have been performed.
