<!-- Trusted provenance: F-ffdc4691fc8f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ffdc4691fc8f

## Purpose

F-ffdc4691fc8f writes a four-byte result record selected by an enable bit, a flag word, and a signed selector.

The established behavior is a classification-and-payload transformation:

- A clear enable bit produces an all-zero record.
- A set enable bit produces a record whose first byte is `0x02`.
- The remaining fields depend on the selector class and the highest-priority applicable flag.
- Some payloads are fixed constants; others are the low 16 bits of externally stored values.

The application meaning of these fields, flags, selectors, and external values is unknown. Terms such as “enabled,” “selector,” and “payload” describe their mechanical roles, not an established protocol or domain interpretation.

## Interface

### Inputs

The function has four arguments:

| Argument | Established representation | Role |
|---|---|---|
| Destination | Pointer to byte-addressable storage | Beginning of the output record |
| Enable word | Unsigned 32-bit integer | Only bit `0x00000001` affects the result |
| Flag word | Unsigned 32-bit integer | Bits `0x40`, `0x80`, `0x10`, and `0x04` participate in selection |
| Selector | Signed 32-bit integer | Selects one of two recognized classes or the fallback case |

The widths above follow the supplied source representation. No independent ABI evidence was supplied.

### Output layout

The destination receives these fields:

| Byte offset | Width | Meaning in this contract |
|---|---|---|
| 0 | 8 bits | State byte |
| 1 | 8 bits | Kind byte |
| 2 | 16 bits | Payload |

The payload is written as a 16-bit value beginning at offset 2. Its physical byte order follows the target’s native representation, which is not established by the supplied evidence.

For example, a payload value of `0x1100` is established numerically. Whether the bytes at offsets 2 and 3 are `00 11` or `11 00` is not established.

There is no explicit return value and no reported status code.

### Preconditions and ownership

The caller must provide storage that supports writes to offsets 0 through 3, including a 16-bit store beginning at offset 2. Any alignment requirement imposed by the target for that store must also be satisfied.

The function does not check for a null destination, insufficient storage, invalid address, or unsuitable alignment. Behavior on such inputs is not defined by this contract beyond the attempted accesses described below.

The function writes through the supplied pointer. It neither allocates nor explicitly releases storage. There is no evidence of ownership transfer, pointer retention, or a dependency call that could manage the destination’s lifetime.

## Behavior

### External payload sources

Two external values participate in enabled execution:

- **Source A:** the external value from which the initial candidate payload is obtained.
- **Source B:** the external value consulted for one recognized selector class.

Only the low 16 bits of each source affect the payload. The complete storage widths, declarations, and semantic meanings of these sources are not established.

The names Source A and Source B are descriptive labels for data dependencies, not function identifiers.

### Disabled path

If bit `0x00000001` of the enable word is clear, the function writes:

- State byte: `0x00`.
- Kind byte: `0x00`.
- Payload: `0x0000`.

It then returns without reading Source A or Source B in the supplied representation.

All other enable-word bits are irrelevant. Thus, `0x00000000` and `0xFFFFFFFE` both select this path. The flag word and selector do not affect its result.

### Enabled path: initial read and selector classification

If bit `0x00000001` of the enable word is set, the function first obtains the low 16 bits of Source A.

This read precedes selector-dependent processing in the supplied representation. It occurs even when subsequent selection replaces or discards the value.

The selector belongs to one of these classes:

| Class | Selector values |
|---|---|
| Source-backed class | `0x30`, `0x34`, `0x35`, `0xCA` |
| Fixed-payload class | `0x20`, `0x24`, `0x25`, `0xC8` |
| Unrecognized | Every other signed 32-bit value |

The selector is compared as a signed value where an ordering comparison occurs. The recognized values are all positive. Negative values, including those with the high bit set, are unrecognized; there is no conversion that makes them aliases for a recognized positive value.

### Source-backed class

For selectors `0x30`, `0x34`, `0x35`, and `0xCA`, the function obtains the low 16 bits of Source B before deciding the result from the flag word.

Flag selection has this priority:

1. `0x40`.
2. `0x80`.
3. `0x10`.
4. `0x04`.
5. None of these bits.

The resulting record is:

| Highest-priority applicable condition | State byte | Kind byte | Payload |
|---|---:|---:|---|
| Flag `0x40` set | `0x02` | `0x03` | Low 16 bits of Source B |
| `0x40` clear, `0x80` set | `0x02` | `0x0B` | `0x1000` |
| `0x40` and `0x80` clear, `0x10` set | `0x02` | `0x0B` | Low 16 bits of Source A |
| Higher-priority flags clear, `0x04` set | `0x02` | `0x03` | Low 16 bits of Source B |
| All four relevant flags clear | `0x02` | `0x03` | `0x0000` |

Source B is read for this class even when the eventual result is the fixed payload `0x1000`, the Source A payload, or zero.

### Fixed-payload class

For selectors `0x20`, `0x24`, `0x25`, and `0xC8`, the same flag priority applies, but the `0x40` and `0x04` cases use a fixed payload instead of Source B:

| Highest-priority applicable condition | State byte | Kind byte | Payload |
|---|---:|---:|---|
| Flag `0x40` set | `0x02` | `0x03` | `0x1100` |
| `0x40` clear, `0x80` set | `0x02` | `0x0B` | `0x1000` |
| `0x40` and `0x80` clear, `0x10` set | `0x02` | `0x0B` | Low 16 bits of Source A |
| Higher-priority flags clear, `0x04` set | `0x02` | `0x03` | `0x1100` |
| All four relevant flags clear | `0x02` | `0x03` | `0x0000` |

Source B is not read on this path.

### Unrecognized selectors

For an unrecognized selector, the enabled result is always:

- State byte: `0x02`.
- Kind byte: `0x03`.
- Payload: `0x0000`.

The flag word does not change this result. In particular, an unrecognized selector does not gain the `0x1000` or Source A payload merely because `0x80` or `0x10` is set.

Source A has nevertheless already been read. Source B is not read.

### Width and arithmetic rules

Selection uses bit tests, equality comparisons, and a signed ordering comparison. There is no payload addition, subtraction, multiplication, or overflow-dependent computation.

External payload conversion retains only the low 16 bits. Upper bits do not affect the selected payload.

Only four flag bits affect the record. All other flag bits are ignored. Multiple relevant flags are not combined into a payload: the priority rules select one outcome.

## Side effects

### Access order

On the disabled path, the represented write sequence is:

1. Write an 8-bit zero at destination offset 0.
2. Write an 8-bit zero at destination offset 1.
3. Write a 16-bit zero beginning at destination offset 2.

On the enabled path, the represented sequence is:

1. Read Source A and retain its low 16 bits.
2. For the source-backed selector class only, read Source B and retain its low 16 bits.
3. Write `0x02` at destination offset 0.
4. Write the selected kind byte at destination offset 1.
5. Write the selected 16-bit payload beginning at destination offset 2.

There is no represented read of the destination’s previous contents. The function overwrites the complete four-byte record on every normally completing path.

These are source-level access observations, not a guarantee of bus transactions, instruction scheduling, volatile semantics, or cross-thread visibility. Those properties require evidence not supplied here.

### Aliasing and intermediate state

No non-aliasing guarantee is established. If the destination overlaps an external source, the represented source reads occur before the destination writes. A payload selected from a source therefore uses the value obtained before these writes, rather than rereading the source after updating the state or kind byte.

The three writes are not established to form an atomic record update. An observer capable of seeing intermediate writes could encounter partially updated contents. No synchronization or memory-ordering mechanism is shown.

No direct modification of Source A or Source B is shown apart from any effect that destination aliasing could have. No other persistent state change is established.

### Errors

The function has no explicit error branch or error return. An unrecognized selector is handled by the enabled fallback record, not by a reported failure.

Faults caused by inaccessible source storage, an invalid destination, or an unsupported store alignment are outside the defined successful behavior. No recovery or rollback is shown. In particular, the write sequence provides no transactional guarantee if a later store fails.

## Dependencies

No function calls are present in the supplied body. There are therefore no callee dependencies requiring opaque function IDs, and no basis for assumptions about callee ownership or preservation of global state.

The established non-call dependencies are:

- Writable destination storage for the four-byte record.
- Source A on every enabled path.
- Source B on source-backed selector paths.
- Target support for the represented byte and 16-bit memory accesses.

Disabled execution does not depend on the contents of either external source.

## Acceptance examples

These are conditional acceptance cases derived from static evidence. They have not been executed.

For the examples involving sources, assume the values read have low 16 bits:

- Source A: `0x2468`.
- Source B: `0x1357`.

Results are written as `(state byte, kind byte, numeric payload)`, without assuming payload byte order.

| Enable word | Selector | Flag word | Expected result |
|---:|---:|---:|---|
| `0x00000000` | `0x34` | `0xFF` | `(0x00, 0x00, 0x0000)` |
| `0xFFFFFFFE` | `0x20` | `0x40` | `(0x00, 0x00, 0x0000)` |
| `0x00000001` | `0x34` | `0x40` | `(0x02, 0x03, 0x1357)` |
| `0x00000001` | `0xCA` | `0x04` | `(0x02, 0x03, 0x1357)` |
| `0x00000001` | `0x30` | `0x80` | `(0x02, 0x0B, 0x1000)` |
| `0x00000001` | `0x35` | `0x10` | `(0x02, 0x0B, 0x2468)` |
| `0x00000001` | `0x34` | `0x00` | `(0x02, 0x03, 0x0000)` |
| `0x00000001` | `0x20` | `0x40` | `(0x02, 0x03, 0x1100)` |
| `0x00000001` | `0xC8` | `0x04` | `(0x02, 0x03, 0x1100)` |
| `0x00000001` | `0x24` | `0x80` | `(0x02, 0x0B, 0x1000)` |
| `0x00000001` | `0x25` | `0x10` | `(0x02, 0x0B, 0x2468)` |
| `0x00000001` | `0x33` | `0xFF` | `(0x02, 0x03, 0x0000)` |
| `0x00000001` | `-1` | `0x80` | `(0x02, 0x03, 0x0000)` |

Additional boundary and precedence conditions:

- With selector `0x34` and flags `0xC0`, `0x40` wins over `0x80`; the payload is `0x1357`, not `0x1000`.
- With selector `0x20` and flags `0x90`, `0x80` wins over `0x10`; the payload is `0x1000`.
- With selector `0x34` and flags `0x14`, `0x10` wins over `0x04`; the kind is `0x0B` and the payload is `0x2468`.
- With selector `0x20` and flags `0x80000000`, none of the relevant bits is set; the payload is zero.
- If Source A instead supplies a value whose low 16 bits are `0xABCD`, a recognized selector with only `0x10` set produces payload `0xABCD`, irrespective of any upper source bits.
- For an enabled unrecognized selector, Source A is still read even though its value cannot affect the record.
- For a disabled call, neither source is read and all four destination bytes become zero.

## Uncertainties

The following remain **documented_only**:

- Target endianness and the resulting payload byte order.
- Exact external-source storage widths, qualifiers, access atomicity, and application meanings.
- Destination alignment requirements and hardware behavior for misaligned stores.
- Whether destination storage can legitimately alias either source in the surrounding application.
- Concurrency rules and whether any observer can access the record while it is being updated.
- Machine-level ordering or access preservation beyond the supplied source representation.
- Calling-convention details beyond the represented argument and return types.

The interpretation of this operation as constructing a small classification record is inferred intent. The numeric layout, selector membership, flag priority, and payload choices are established static behavior.

## Evidence

This contract is based solely on the supplied body for F-ffdc4691fc8f.

The static evidence establishes:

- A low-bit enable gate with a separate all-zero output path.
- An enabled-path initial read of Source A.
- Four selectors in each recognized class.
- A Source B read for the source-backed class before flag-dependent selection.
- Priority `0x40` over `0x80` over `0x10` over `0x04`.
- State and kind bytes followed by a 16-bit payload store.
- No function calls, explicit allocation, explicit release, or error return.

No implementation access, runtime tests, target-memory inspection, or independent ABI verification has been performed. The acceptance examples are derived expectations, not observed execution results.
