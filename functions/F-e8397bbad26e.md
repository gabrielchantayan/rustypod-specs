<!-- Trusted provenance: F-e8397bbad26e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e8397bbad26e

## Purpose

F-e8397bbad26e classifies a supplied 32-bit value under the control of a 32-bit mode argument. Depending on that classification, it performs one of three actions:

- Sets one bit in a 32-bit field of a caller-supplied context.
- Copies a 32-bit value from a global source into another context field.
- Delegates handling of the supplied value to F-96b68e21fa81 using an address within the context.

The established behavior is a small dispatch operation. The interpretation of the context as a parser, the supplied values as characters, or the returned values as parser states is inferred intent, not an established interface guarantee.

This contract describes the supplied static evidence. No runtime tests have been performed.

## Interface

### Inputs

The function accepts three arguments:

| Argument | Representation | Established use |
|---|---|---|
| Context address | Address-bearing value represented by a 32-bit integer in the supplied evidence | Base for fixed byte-offset accesses and for an address passed to the dependency |
| Mode | 32-bit integer | Compared for exact equality with 1 |
| Supplied value | 32-bit integer | Compared against four exact values, or passed unchanged to the dependency |

Only a mode value equal to 1 enables classification of the supplied value. Every other mode value selects the same global-to-context assignment, irrespective of the supplied value.

The supplied value is compared as a complete 32-bit integer. No byte extraction, character decoding, masking, or range normalization is shown. A value whose low byte matches a recognized value but whose other bits differ is not thereby recognized.

### Return value

The function returns a 32-bit result. On all explicitly shown paths that complete normally, the result is one of:

| Result | Circumstances |
|---|---|
| 2 | Mode is 1, and the supplied value either selects a bit update or is delegated |
| 13 | Mode is not 1, or mode is 1 and the supplied value is exactly `0x3E` |

A dependency result does not determine this return value: if the dependency returns normally, its result is ignored and this function returns 2.

These numbers are established return values, not established success or failure codes.

### Context layout used directly

Offsets are measured in bytes from the context address.

| Offset | Extent used | Access |
|---|---:|---|
| `0x1C` | Unknown | Address passed as the first argument to F-96b68e21fa81 |
| `0x28` | 4 bytes | Read and then overwritten on bit-update paths |
| `0x38` | 4 bytes | Overwritten on global-assignment paths |

The function does not establish the overall context size, field names, or the extent of storage required by the dependency at offset `0x1C`.

For direct accesses, the selected four-byte region must be readable or writable as required by the selected path, with any alignment required by the actual execution environment. The supplied evidence contains no explicit validity checks.

### Global sources

This contract assigns descriptive labels to two distinct global 32-bit sources:

- **Primary global source:** read unconditionally before dispatch.
- **Secondary global source:** read only when mode is 1 and the supplied value is not `0x2F`, `0x21`, or `0x3F`.

These labels describe roles, not symbol names or semantic types. Either source might contain an integer, an address representation, a state token, or another four-byte value. Its meaning is unknown.

## Behavior

### Entry and initial read

Every invocation first captures a 32-bit value from the primary global source.

This read occurs even on paths that do not subsequently use the captured value, including all bit-update paths and all delegated paths. It also precedes any explicit context-field access or dependency call.

The remaining behavior depends on the mode.

### Mode other than 1

If the mode is anything other than exactly 1:

1. The previously captured primary value is written as a four-byte value at context offset `0x38`.
2. The function returns 13.

The supplied value is not classified on this path. In particular, even values that would select a bit update under mode 1 do not cause a flags-field access here.

The secondary global source is not read, and the dependency is not called.

### Mode 1: recognized bit updates

Three supplied values select a read-modify-write operation on the four-byte field at context offset `0x28`:

| Supplied value | Bit mask combined with the field |
|---|---:|
| `0x2F` | `0x00000001` |
| `0x21` | `0x00000010` |
| `0x3F` | `0x00000020` |

For each of these cases, the function:

1. Reads the field as a 32-bit unsigned value.
2. Combines it with the selected mask using bitwise OR.
3. Writes the resulting 32-bit value back to the same field.
4. Returns 2.

The selected bit is set; all other bits retain the values observed by the field read. An already-set selected bit remains set.

No arithmetic increment, toggle, clearing operation, or multi-bit replacement is performed. The full four-byte field is written even when the computed value equals the value read.

The function does not explicitly write offset `0x38` on these paths. It does not read the secondary global source or call the dependency.

### Mode 1: alternate global assignment

For mode 1 and supplied value `0x3E`:

1. The function reads the secondary global source.
2. It writes that captured four-byte value at context offset `0x38`.
3. It returns 13.

The primary global read still occurs first, but its captured value is not used for this assignment.

The field at offset `0x28` is not explicitly read or written on this path. The dependency is not called.

### Mode 1: delegated values

For mode 1 and any supplied value other than `0x2F`, `0x21`, `0x3F`, or `0x3E`:

1. The function reads the secondary global source.
2. It calls F-96b68e21fa81 with:
   - First argument: the context address advanced by `0x1C` bytes.
   - Second argument: the original supplied 32-bit value.
3. If the call returns normally, the function returns 2.

Both global reads occur before the call: primary first, secondary second. Neither captured value is explicitly used after the secondary read on this path.

The caller performs no explicit access to context offset `0x28` or explicit write to offset `0x38` on this path. This is not a guarantee that those locations remain unchanged: the dependency’s effects are unknown.

## Side effects

### Ordered direct accesses

The path-specific order of explicit memory accesses and calls is:

| Selected path | Ordered operations |
|---|---|
| Mode other than 1 | Read primary global; write context `+0x38`; return 13 |
| Mode 1, `0x2F` | Read primary global; read context `+0x28`; write context `+0x28`; return 2 |
| Mode 1, `0x21` | Read primary global; read context `+0x28`; write context `+0x28`; return 2 |
| Mode 1, `0x3F` | Read primary global; read context `+0x28`; write context `+0x28`; return 2 |
| Mode 1, `0x3E` | Read primary global; read secondary global; write context `+0x38`; return 13 |
| Mode 1, any other value | Read primary global; read secondary global; call F-96b68e21fa81; return 2 if the call returns |

This ordering describes the supplied static operations. It does not establish memory barriers, volatile semantics, atomicity, or a cross-thread visibility guarantee.

### State and aliasing

The bit-update paths preserve unrelated bits relative to the value read. They are not shown to be atomic read-modify-write operations. Concurrent updates could therefore be lost unless the surrounding execution model supplies additional guarantees.

Global-assignment paths overwrite the destination field without first reading it. No disposal or release of its prior contents is shown.

The evidence does not establish that the context and global storage cannot alias. If they do alias, a context write may also change storage used as a global source. The written value remains the value captured by the preceding read, rather than a promised later reread.

### Ownership and lifetime

No direct allocation, deallocation, reference-count adjustment, or ownership transfer is shown.

Passing the interior context address to F-96b68e21fa81 establishes only that the address is provided to that dependency. It does not establish whether the dependency retains it, modifies nearby storage, changes globals, allocates memory, or releases resources.

### Errors and abnormal completion

There is no explicit error-return branch, null-address check, bounds check, alignment check, or dependency-failure translation.

Invalid memory, unsuitable addresses, or abnormal completion inside the dependency may prevent a normal return. The resulting fault or control-flow behavior is not specified by the supplied evidence. Results 2 and 13 must not be interpreted as proof that all memory operations or delegated work succeeded.

## Dependencies

### F-96b68e21fa81

This is the only explicit called dependency.

It is invoked exactly once on a delegated path and not invoked on any other path. It receives the context address advanced by 28 bytes and the unchanged supplied 32-bit value.

Its return value is not used. Its memory effects, global effects, ownership behavior, accepted input domain, and abnormal-completion behavior are **documented_only** unknowns.

In particular, the fact that this function has no explicit post-call context writes does not imply that the context remains unchanged after delegation.

## Acceptance examples

These examples are conditional expectations derived from static evidence, not executed tests. Unless stated otherwise, they assume valid accesses, normal completion, and no interfering memory modification.

1. **Set the low bit.**  
   With mode 1, supplied value `0x2F`, and a field value of `0x80000010` at offset `0x28`, the field becomes `0x80000011`, and the result is 2. The primary global is read first. Neither the secondary global nor the dependency is used.

2. **Set an already-set bit.**  
   With mode 1, supplied value `0x21`, and a field value of `0x00000010`, the field is read and written back as `0x00000010`. The result is 2. An unchanged numerical value does not eliminate the specified write.

3. **Preserve other bits.**  
   With mode 1, supplied value `0x3F`, and a field value of `0xFFFFFFDF`, the written value is `0xFFFFFFFF`, and the result is 2.

4. **Choose the secondary source.**  
   With mode 1 and supplied value `0x3E`, suppose the primary read obtains `0x11223344` and the subsequent secondary read obtains `0xAABBCCDD`. Offset `0x38` receives `0xAABBCCDD`, and the result is 13. The primary value is not assigned.

5. **Mode takes precedence.**  
   With mode 0 and supplied value `0x2F`, offset `0x38` receives the captured primary value, and the result is 13. No bit update occurs. The same branch applies to any other mode unequal to 1, including a negative mode in a signed 32-bit interpretation.

6. **Delegate an unrecognized value.**  
   With mode 1 and supplied value `0x41`, the primary and secondary globals are read in that order. The dependency receives the interior address at offset `0x1C` and the value `0x41`. If it returns normally, the result is 2 regardless of its returned value.

7. **Do not truncate to a byte.**  
   With mode 1 and supplied value `0x0000012F`, delegation occurs. The value is not treated as `0x2F`, despite the matching low byte.

8. **Dependency effects remain unconstrained.**  
   If a delegated call changes offset `0x38`, this function performs no explicit subsequent write that restores its prior contents. Acceptance cannot require that field to remain unchanged without a separate dependency contract.

## Uncertainties

The following remain **documented_only**:

- Semantic names and meanings of the context fields, global values, mode, and return values.
- Whether the recognized numeric values intentionally represent punctuation.
- The context’s full layout, total size, lifetime rules, and required alignment.
- Runtime byte order and the significance of individual bytes within four-byte fields.
- Valid address ranges and the behavior of address advancement near the address-space boundary.
- Whether global accesses have special hardware, volatile, or synchronization properties.
- Any restrictions preventing aliasing or concurrent modification.
- All effects and validity requirements of F-96b68e21fa81 beyond its observed arguments and call placement.
- Fault handling, nonlocal control transfer, and recovery behavior.

The supplied representation uses 32-bit integers and four-byte field accesses. No narrower integer operation is shown. Address calculations use fixed byte displacements, but safe wrapping or overflow behavior is not established; this contract does not promise it.

## Evidence

**Established static evidence:** the exact mode comparison; the four recognized supplied values; the three bit masks; four-byte field accesses at offsets `0x28` and `0x38`; the interior address at offset `0x1C`; the two global-read roles and their order; dependency-call placement and arguments; and normal return values 2 and 13.

**Inferred intent:** the dispatch may participate in character-oriented state processing, and offset `0x28` behaves like a flags field. These interpretations are not required for conformance.

**Unknown:** semantic types, broader state-machine meaning, ownership rules, dependency internals, runtime memory guarantees, and exceptional behavior.

No runtime tests, execution traces, or independent dependency inspection support this contract.
