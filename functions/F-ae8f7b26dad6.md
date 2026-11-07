<!-- Trusted provenance: F-ae8f7b26dad6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ae8f7b26dad6

## Purpose

F-ae8f7b26dad6 recognizes sixteen complete 32-bit selector values, constructs a selector-dependent four-byte payload, conditionally passes that payload to F-fc9444c74c1d, and then calls F-ab545ff44214.

Its 64-bit return value records recognition in the low 32 bits. The high 32 bits contain either the original selector or zero, depending on the selected branch. Recognition does not establish that either dependency succeeded.

Selectors outside the recognized range cause an immediate return without a dependency call or an explicit memory access through the object reference.

The payload may represent three components and an additional constant component. Interpreting these as color channels and opacity is inferred intent, not an established requirement.

## Interface

### Inputs

| Input | Width | Established role |
|---|---:|---|
| Object reference | 32 bits | Address-like value used for accesses at byte offsets `0xB0` and `0xB4`; passed unchanged to F-ab545ff44214. |
| Selector | 32 bits, unsigned | Selects the payload, determines recognition, and supplies the high result word on paths without payload submission. |

The object reference is represented as a signed 32-bit quantity in the supplied static material. Its use for addressing is established. Its original pointer type, address-conversion rules, and valid address range are not established.

Recognition uses the entire selector. There is no preliminary low-byte mask or narrowing. For example, `0x00000020` is recognized, but `0x00000120` is not.

### Result

The returned 64-bit bit pattern has these two words:

| Execution path | High 32 bits | Low 32 bits |
|---|---:|---:|
| Unrecognized selector | Original selector | `0` |
| Recognized selector; initial field read at `0xB0` is zero | Original selector | `1` |
| Recognized selector; initial field read at `0xB0` is nonzero | `0` | `1` |

The unrecognized result is the unsigned selector widened to 64 bits and shifted left by 32 bits. No selector bits are lost.

On the recognized zero-field path, the result has that same high word and a low word of one. On the nonzero-field path, normal completion returns the 64-bit value one.

These are bit-pattern guarantees. A signed interpretation of the result may be negative when the high word has its top bit set; this does not change recognition or branch selection.

### Directly accessed object fields

| Byte offset | Access width | Use |
|---|---:|---|
| `0xB0` | 32 bits | Compared with zero; read again for the first argument to F-fc9444c74c1d if the comparison is nonzero. |
| `0xB4` | 32 bits | Read only on the submission branch; masked to its low eight bits for the second argument to F-fc9444c74c1d. |

The routine contains no explicit store through the object reference.

A recognized selector requires a valid initial access at `0xB0`. The nonzero-field branch additionally requires valid accesses for the repeated `0xB0` read and the `0xB4` read. Dependency-specific preconditions may be broader.

The supplied material does not establish address-overflow behavior, alignment requirements, or a complete object allocation size. The offsets do not independently establish those properties.

## Behavior

### Recognition and payload construction

Every selector from `0x20` through `0x2F`, inclusive, is recognized. Each adjacent even/odd pair selects the same payload construction.

Before selector dispatch, the routine initializes the third byte of its temporary three-byte component storage to zero. This local initialization also precedes an unrecognized-selector return; it is not an access through the object reference.

For recognized selectors, the three-byte component is formed as follows:

| Selector pair | Established component construction |
|---|---|
| `0x20`, `0x21` | A complete 24-bit assignment with numerical value `0xF0F0F0`. |
| `0x22`, `0x23` | A complete 24-bit assignment with numerical value `0x056005`. |
| `0x24`, `0x25` | A complete 24-bit assignment with numerical value `0xA5630D`. |
| `0x26`, `0x27` | A complete 24-bit assignment with numerical value `0x8D7E04`. |
| `0x28`, `0x29` | First byte set to `0x79`, second byte set to zero, and third byte retaining its initial zero. |
| `0x2A`, `0x2B` | First two bytes assigned the 16-bit value `0x4C6D`; third byte set to `0x27`. |
| `0x2C`, `0x2D` | First two bytes assigned the 16-bit value `0x2F55`; third byte set to `0x69`. |
| `0x2E`, `0x2F` | First and second bytes set to zero, with the third byte retaining its initial zero. |

The distinction between complete 24-bit assignments and byte or partial-width assignments matters. A 16-bit numerical assignment does not establish its increasing-address byte order without an endianness assumption. Likewise, a sequence of individual byte assignments does not establish a particular packed numerical value independently of the representation used to interpret those bytes.

The routine then forms a four-byte payload by concatenating the eight-bit value `0xCC` above the selected 24-bit component. The payload is fully initialized before any dependency call. No recognized case exposes an uninitialized component.

The following table gives the resulting numerical values and increasing-address bytes **under a little-endian interpretation**:

| Selector pair | 24-bit component | 32-bit payload | Four bytes at increasing addresses |
|---|---:|---:|---|
| `0x20`–`0x21` | `0xF0F0F0` | `0xCCF0F0F0` | `F0 F0 F0 CC` |
| `0x22`–`0x23` | `0x056005` | `0xCC056005` | `05 60 05 CC` |
| `0x24`–`0x25` | `0xA5630D` | `0xCCA5630D` | `0D 63 A5 CC` |
| `0x26`–`0x27` | `0x8D7E04` | `0xCC8D7E04` | `04 7E 8D CC` |
| `0x28`–`0x29` | `0x000079` | `0xCC000079` | `79 00 00 CC` |
| `0x2A`–`0x2B` | `0x274C6D` | `0xCC274C6D` | `6D 4C 27 CC` |
| `0x2C`–`0x2D` | `0x692F55` | `0xCC692F55` | `55 2F 69 CC` |
| `0x2E`–`0x2F` | `0x000000` | `0xCC000000` | `00 00 00 CC` |

In particular, the packed numerical values shown for `0x28`–`0x29`, `0x2A`–`0x2B`, and `0x2C`–`0x2D` are conditional on little-endian layout. They are not unconditional numerical consequences of the byte and partial-width stores.

The complete 24-bit numerical assignments in the first four rows, and the numerical concatenation operation, are directly represented. Their increasing-address byte layouts still require an endianness interpretation. The supplied material does not independently establish the target ABI or endianness.

### Unrecognized selectors

For any selector outside `0x20` through `0x2F`:

- Return the original selector in the high result word and zero in the low word.
- Do not call either dependency.
- Do not explicitly access memory through the object reference.
- Do not complete or submit a selected four-byte payload.

The earlier initialization of one local component byte does not alter these object-access and call guarantees.

### Recognized selector with a zero initial field read

After completing the payload, the routine reads the 32-bit field at object offset `0xB0`.

If that read yields zero:

1. F-fc9444c74c1d is skipped.
2. F-ab545ff44214 receives the original object reference.
3. If F-ab545ff44214 returns normally, the result contains the original selector in the high word and one in the low word.

This branch performs no explicit read at offset `0xB4` and no second explicit read at offset `0xB0`. F-ab545ff44214 may perform its own accesses; its internal behavior is unknown.

### Recognized selector with a nonzero initial field read

If the initial `0xB0` read yields a nonzero value, the routine selects the submission branch and replaces the selector value used for the final high result word with zero.

It calls F-fc9444c74c1d with six arguments in these positions:

| Position | Value |
|---:|---|
| 1 | A fresh 32-bit read from object offset `0xB0`. |
| 2 | A 32-bit read from object offset `0xB4`, masked with `0xFF`. |
| 3 | Zero. |
| 4 | Zero. |
| 5 | Zero. |
| 6 | A reference to the temporary payload storage. |

The second argument is in the inclusive range `0` through `255`. Bits 8 through 31 of the field read do not affect it. The routine does not scale, clamp, or otherwise transform that low-byte value.

The branch-decision read and the first-argument read at `0xB0` are distinct. The first argument is therefore not guaranteed to equal the earlier value or to be nonzero.

Both argument-producing field reads occur after the branch decision and before entry into F-fc9444c74c1d. Their relative evaluation order is not established by the supplied representation.

After F-fc9444c74c1d returns normally, F-ab545ff44214 receives the original object reference. If that call also returns normally, the routine returns one.

Neither dependency’s return value contributes to the result.

### Completion and error behavior

There is no visible local validation, dependency-status check, retry, rollback, or alternate error result.

A low result word of one establishes selector recognition, not successful submission or successful completion of a semantic operation. Dependency success criteria are unknown.

A normal return requires all accesses and reached dependency calls to complete normally. No local recovery is shown for invalid memory access, dependency faults, nonlocal exits, or nonreturning dependency behavior. If F-fc9444c74c1d does not return normally, the subsequent F-ab545ff44214 call is not guaranteed to occur.

## Side effects

### Established ordering and access effects

The observable call-site sequence is:

- Initialize one local component byte before dispatch.
- For a recognized selector, finish constructing the payload.
- Read the field at `0xB0` for the branch decision.
- On the nonzero branch, reread `0xB0`, read `0xB4`, and call F-fc9444c74c1d.
- Call F-ab545ff44214 after any preceding submission call returns normally.
- Return the branch-dependent result after the final dependency returns normally.

The two argument-producing field reads have no established relative order.

F-fc9444c74c1d is called at most once. F-ab545ff44214 is called once for a recognized selector if execution reaches it. Neither is called for an unrecognized selector.

There are no explicit object-field stores. This does not guarantee that the object, globals, or other reachable state remain unchanged: dependency effects are unknown.

### Ownership and lifetime

The payload is temporary storage local to this invocation. Its address is passed to F-fc9444c74c1d, not returned to the caller.

There is no visible allocation, release, ownership-transfer operation, or explicit copy into persistent storage. Whether F-fc9444c74c1d copies, mutates, retains, or releases anything is undocumented. Retention of the temporary address must not be assumed safe after this invocation ends.

Passing the object reference to F-ab545ff44214 does not establish that it preserves or releases the object. Neither dependency is assumed to preserve globals, retain object validity, or leave unrelated memory unchanged.

## Dependencies

### F-fc9444c74c1d

**Established call-site contract:** Receives six arguments: a reread 32-bit object field, a masked low-byte value, three zeros, and a reference to the selected temporary payload.

**Condition:** The selector is recognized and the initial read at object offset `0xB0` is nonzero.

**Ordering:** Called before F-ab545ff44214.

**Result handling:** Any returned value is ignored.

**documented_only:** Parameter semantics, signedness expectations beyond the observed bit patterns, payload access extent, mutation permissions, retention, ownership, global effects, failure signaling, and object-lifetime effects.

### F-ab545ff44214

**Established call-site contract:** Receives the original 32-bit object reference.

**Condition:** The selector is recognized, and any preceding F-fc9444c74c1d call returns normally.

**Result handling:** Any returned value is ignored.

**documented_only:** Whether the operation updates state, emits notifications, performs cleanup, releases resources, or serves another purpose. No such interpretation is an acceptance requirement.

## Acceptance examples

These are conditional acceptance observations, not executed tests. Unless stated otherwise, required accesses are valid and reached dependencies return normally.

### Values outside the range

For selector `0x1F`, the result is `0x0000001F00000000`.

For selector `0x30`, the result is `0x0000003000000000`.

In both cases, neither dependency is called and the routine performs no explicit object-field access. The local byte initialization before dispatch does not constitute an object-field access.

### Recognition uses all selector bits

For selector `0x00000120`:

- The selector is not treated as `0x20`.
- The result is `0x0000012000000000`.
- Neither dependency is called.

For selector `0xFFFFFFFF`, the result bits are `0xFFFFFFFF00000000`, again with no dependency call or explicit object-field access.

### Recognized value without submission

For selector `0x24`, if the initial `0xB0` read yields zero:

- F-fc9444c74c1d is skipped.
- F-ab545ff44214 receives the original object reference once.
- No explicit `0xB4` read occurs.
- The result is `0x0000002400000001`.

### Recognized value with submission

For selector `0x25`, suppose both `0xB0` reads yield `0x12345678` and the `0xB4` read yields `0xABCDEF92`:

- F-fc9444c74c1d receives `0x12345678`, `0x92`, three zeros, and the temporary payload reference.
- The complete 24-bit component assignment is `0xA5630D`, and numerical concatenation produces `0xCCA5630D`.
- Under little-endian layout, the four payload bytes are `0D 63 A5 CC`.
- F-ab545ff44214 is called afterward.
- The result is `0x0000000000000001`.

### Partial-width construction

For selector `0x2C` or `0x2D`, construction assigns the first two component bytes the 16-bit value `0x2F55` and the third component byte `0x69`.

Under little-endian layout, this yields component value `0x692F55`, payload value `0xCC692F55`, and payload bytes `55 2F 69 CC`.

Without that layout assumption, those packed numerical values and that byte sequence are not acceptance requirements; the observed partial-width and byte assignments remain established.

### Paired selectors with distinct results

Selectors `0x2A` and `0x2B` use identical component construction: a 16-bit assignment of `0x4C6D` followed by a third component byte of `0x27`.

Under little-endian layout, both produce packed payload value `0xCC274C6D`.

If the initial `0xB0` read is zero:

- `0x2A` returns `0x0000002A00000001`.
- `0x2B` returns `0x0000002B00000001`.

If the submission branch is selected, both return one after normal completion of both calls.

### Distinct reads at the same offset

If the environment permits the initial `0xB0` read to yield a nonzero value and the subsequent read to yield zero:

- The submission branch remains selected.
- F-fc9444c74c1d receives zero as its first argument.
- The final result is one after normal dependency completion.

This example does not establish concurrency safety, volatile semantics, or permission for unsynchronized mutation.

## Uncertainties

The following remain **documented_only**:

- Original object type, target ABI, endianness, alignment rules, and address-arithmetic behavior.
- Semantic meanings of selector pairs and payload components.
- Whether `0xCC` represents opacity, a flag, or another quantity.
- Intended interpretation of the result beyond its established word construction.
- Relative order of the two argument-producing field reads.
- Complete dependency contracts, including memory access extents and ownership effects.
- Object and global-state preservation across dependency calls.
- Concurrency, synchronization, volatile-memory, and reentrancy requirements.
- Behavior after invalid accesses or abnormal dependency completion.

In particular, target endianness must not be inferred from the convenient little-endian payload table.

## Evidence

Static evidence establishes the complete selector comparisons, paired construction paths, initial local-byte initialization, numerical constants used by complete and partial-width assignments, constant-byte concatenation, object offsets, field widths, low-byte mask, dependency argument positions, conditional call structure, call ordering, and 64-bit result construction.

For selector pairs `0x28`–`0x29`, `0x2A`–`0x2B`, and `0x2C`–`0x2D`, the stated packed numerical values are little-endian interpretations of byte or partial-width construction—not endianness-independent static facts. Increasing-address payload byte sequences are likewise conditional on the stated layout.

Color, opacity, refresh, cleanup, notification, and ownership-transfer interpretations are not established.

No runtime tests have been performed.
