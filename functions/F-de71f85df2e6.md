<!-- Trusted provenance: F-de71f85df2e6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-de71f85df2e6

## Purpose

F-de71f85df2e6 conditionally prepares two records, submits them to a processing dependency, and, when that dependency reports more than one result, processes and quantizes the first two returned records before invoking an indirect callback.

The established behavior is expressed in terms of memory offsets, 32-bit values, call order, and conditional writes. The apparent roles of the records, their fields, and the callback are not established. In particular, “slot,” “record,” and “configuration” below are descriptive labels, not recovered application types.

This contract distinguishes:

- **Established static evidence:** operations and ordering visible in the supplied representation.
- **Inferred intent:** possible interpretations that are not required behavior.
- **Unknowns / documented_only:** behavior requiring dependency implementations, ABI information, or runtime evidence.

No runtime tests have been performed.

## Interface

The function accepts three arguments and returns no explicit value:

| Argument | Contract label | Observable use |
|---|---|---|
| First | Context | Base address for configuration, mode bytes, processing dependencies, and a callback-bearing object. |
| Second | Record A | Input record whose flags participate in the gate; also a possible source of prepared data and a destination for slot writes. |
| Third | Record B | Input record whose flags participate in the gate; the first record prepared by the optional preparation stages. |

The supplied representation uses 32-bit argument, pointer, integer, and slot values. Address calculations therefore need to be interpreted in the target’s 32-bit execution model, not silently widened to host-sized arithmetic.

### Relevant layouts

Offsets are byte offsets from the applicable base address.

| Base | Offset | Observed access |
|---|---:|---|
| Context | `0x8FD` | One-byte mode value; zero and nonzero select different preparation paths. |
| Context | `0x9EC` | One-byte selection value; exactly `1` selects Record A’s own source. All other values select reuse of Record B’s source or result. |
| Context | `0xA8C` | 32-bit pointer to an object containing an indirect callable entry at offset `0x48`. |
| Context | `0xA90` | 32-bit pointer to configuration storage. |
| Configuration | `0x04` | Signed 32-bit index controlling the first preparation stage. |
| Configuration | `0x08` | Signed 32-bit index controlling the second preparation stage. |
| Either input record | `0x08` | Signed 32-bit value used by the second preparation stage. |
| Either input record | `0x3C` | Address supplied to F-02a35f55d6ef in the zero-mode path. |
| Either input record | `0x4C` | Address supplied to F-02a35f55d6ef in the nonzero-mode path. |
| Either input record | `0x6C + 4 × index` | Address of an indexed four-byte slot. |
| Either input record | `0x90` | 32-bit flags word used by the initial gate. |
| Returned record | `0x20`, `0x24` | Four-byte fields quantized after result processing. |
| Callback-bearing object | `0x48` | Indirect callable entry. |

The extents and internal layouts of the regions beginning at `0x3C` and `0x4C` are unknown. Supplying their addresses does not establish that each region is only four bytes long.

### Preconditions

For defined execution, every memory access actually reached must address valid storage with the required access permissions and target alignment. In particular:

- Both input records must support the initial flags reads, even if the gate subsequently suppresses all work.
- An open gate requires a readable context configuration pointer and the configuration fields used by the enabled stages.
- A nonnegative slot index must identify valid storage for the corresponding operation. No upper-bound check is visible.
- A result count greater than one requires a valid returned pointer array containing at least two usable record pointers.
- The callback-bearing object and its callable entry must be valid if the final path is reached.

No explicit checks for null pointers, index bounds, dependency failure, or callback validity are visible.

## Behavior

### 1. Initial gate

The function reads the flags words at offset `0x90` in Record A and Record B. Their relative evaluation order is not established.

Processing continues only if the bitwise intersection of those flags has none of bits 2 through 7 set. Equivalently, processing is suppressed when the two records share any set bit in mask `0xFC`.

Consequences:

- Shared bits 0 or 1 do not suppress processing.
- Shared bits above bit 7 do not suppress processing.
- Different set bits in the two records do not suppress processing unless at least one bit in positions 2–7 is shared.

When the gate suppresses processing, no subsequent context access, dependency call, or explicit write shown in this function occurs.

### 2. First optional preparation stage

The function obtains a signed 32-bit index from configuration offset `0x04`. A negative index skips this entire stage. A nonnegative index enables it.

The byte at context offset `0x8FD` is then read to choose one of two paths.

#### Zero-mode path

When the mode byte is zero:

1. F-02a35f55d6ef is called with:
   - the address at Record B offset `0x3C`;
   - the address of Record B’s slot selected by the initially obtained index.
2. After that call, the context’s configuration pointer and configuration index at offset `0x04` are read again to form Record A’s slot address.
3. The selection byte at context offset `0x9EC` is read.
4. F-02a35f55d6ef is called again:
   - with Record A offset `0x3C` as its first address if the selection byte equals `1`;
   - otherwise with Record B offset `0x3C` as its first address;
   - and with the already calculated Record A slot address as its second address.

The second call need not use the same slot index as the first. The intervening dependency may change configuration.

#### Nonzero-mode path

When the mode byte is nonzero:

1. F-f370660ffa4f is called with the context, Record B, and the integer value `1`.
2. The configuration pointer and index at offset `0x04` are read again.
3. F-02a35f55d6ef is called with Record B offset `0x4C` and the resulting Record B slot address.
4. The selection byte at context offset `0x9EC` is read.
5. If that byte equals `1`:
   - F-f370660ffa4f is called with the context, Record A, and `1`;
   - the configuration pointer and index at offset `0x04` are read again;
   - F-02a35f55d6ef is called with Record A offset `0x4C` and that Record A slot address.
6. Otherwise:
   - the configuration pointer and index at offset `0x04` are read again;
   - F-02a35f55d6ef is called with Record B offset `0x4C` and that Record A slot address.

Only the initial index read is checked for negativity. Later rereads are used without another sign or bounds check.

The first and second address arguments of F-02a35f55d6ef are established; their source/destination semantics and memory effects are not.

### 3. Second optional preparation stage

After the first stage finishes or is skipped, the function reads the current configuration pointer and the signed 32-bit index at configuration offset `0x08`.

A negative index skips this stage. Otherwise:

1. The signed 32-bit field at Record B offset `0x08` is read.
2. If negative, it is negated; otherwise it is used unchanged.
3. F-96ada6d600f3 is called with the context and the resulting value.
4. The returned four-byte value is written to Record B’s selected slot.
5. The selection byte at context offset `0x9EC` is read after that write.
6. If the byte equals `1`:
   - Record A’s signed field at offset `0x08` is read;
   - the same conditional-negation operation is applied;
   - F-96ada6d600f3 is called with the context and that value;
   - its four-byte result becomes the value for Record A.
7. The selected four-byte value is written to Record A’s slot.

If the selection byte is not `1`, Record A receives the result previously returned for Record B. This is reuse of the saved return value, not a fresh read from Record B’s slot.

Unlike the first preparation stage, this stage retains its original index across the dependency calls. Changes to configuration during those calls do not select a new slot for its explicit writes.

For negative inputs other than the minimum signed 32-bit value, conditional negation produces the mathematical magnitude. The minimum value, `−2147483648`, has no positive signed 32-bit counterpart. Its exact treatment is documented_only; the supplied signed representation does not establish a portable overflow rule.

### 4. Submission of the input pair

The function next prepares invocation-local storage comprising:

- two adjacent 32-bit record addresses, ordered Record A followed by Record B;
- a 64-byte scratch region with no visible initialization;
- a 32-bit output-pointer cell initialized to null.

It calls F-a6abddefd72e with five arguments, in this order:

1. Context;
2. integer count `2`;
3. address of the two-address input sequence;
4. address of the scratch region;
5. address of the output-pointer cell.

This call occurs whenever the gate is open, including when both preparation indices are negative.

The return value is interpreted as an unsigned 32-bit count. Counts `0` and `1` cause no further explicit work. Every value greater than `1` selects the final processing path, including values whose signed interpretation would be negative.

### 5. Processing the first two results

For an unsigned result count greater than one:

1. The first pointer is read from the currently stored output-array pointer and passed to F-0a29566d2e48 with the context.
2. After that call, the second pointer is read from the output array and passed to F-0a29566d2e48 with the context.
3. After both calls, the callback-bearing object pointer is read from context offset `0xA8C`.
4. The first and second returned-record pointers are read again from the output array and retained for the explicit field updates.
5. Four field updates occur in this order:
   - first returned record, offset `0x20`;
   - first returned record, offset `0x24`;
   - second returned record, offset `0x20`;
   - second returned record, offset `0x24`.
6. The callable entry at offset `0x48` of the previously obtained callback-bearing object is invoked.

Each field update reads its current 32-bit value, adds `0x800` using modulo-\(2^{32}\) arithmetic, clears its lowest 12 bits with mask `0xFFFFF000`, and writes the resulting 32-bit value back.

This is a bit-level quantization rule. It can wrap at the 32-bit boundary. A geometric coordinate or fixed-point interpretation is inferred intent, not established type information.

Only two output records are explicitly processed, regardless of whether the reported count is two or larger.

## Side effects

Explicit writes performed directly by this function are:

- Up to one four-byte slot write in Record B and one in Record A during the second preparation stage.
- Initialization of invocation-local input and output-pointer storage.
- Four ordered returned-record field writes on the final processing path.

Other memory effects may occur through dependencies, including F-02a35f55d6ef, whose argument direction is unknown.

No explicit allocation, deallocation, release, or ownership transfer is visible. This does not establish that dependencies avoid those operations. The ownership and lifetime of the output array and its records remain documented_only.

The function does not prevent aliasing:

- Record A and Record B may identify the same storage.
- The two returned pointers may be equal.
- Output records may overlap input records.
- Configuration, slots, or other regions may overlap unless an external precondition forbids it.

Where aliasing is valid, later reads observe earlier writes according to the visible sequence and dependency effects. In particular, identical returned pointers cause the same fields to be quantized twice, rather than once.

Dependency calls must not be assumed to preserve context fields, output-array entries, or other reachable memory. The rereads described above are behaviorally significant.

## Dependencies

| Opaque ID | Established use | Not established |
|---|---|---|
| F-02a35f55d6ef | Receives two addresses during the first preparation stage. | Copy direction, accessed widths, conversion behavior, return meaning, ownership effects. |
| F-f370660ffa4f | Receives context, one input record, and `1` in the nonzero-mode path. | Meaning of the flag, fields changed, allocation or release behavior. |
| F-96ada6d600f3 | Receives context and a conditionally negated signed input; supplies a four-byte slot value. | Mapping performed, failure signaling, global or context mutations. |
| F-a6abddefd72e | Receives a pair of record addresses, scratch storage, and an output-pointer location; returns an unsigned count. | Result ownership, output-array capacity, scratch contents, retention of passed addresses, failure conventions. |
| F-0a29566d2e48 | Called on each of the first two result entries before quantization. | Record transformations, mutation of output-array entries, ownership effects. |
| Indirect callback | Invoked through offset `0x48` after all four explicit quantization writes. | Semantic identity, full calling convention, implicit arguments, return behavior. |

No explicit arguments to the indirect callback are represented. This is not proof of its complete machine-level signature.

## Acceptance examples

These examples are conditional expectations derived from static evidence, not executed tests.

1. **Closed gate:** If both flags words contain bit `0x04`, no preparation or submission dependency is called.

2. **Irrelevant shared flags:** If the only shared bits are `0x01`, `0x02`, or bits above `0xFF`, the gate remains open.

3. **Both preparation stages disabled:** With both configuration indices negative and otherwise valid state, F-a6abddefd72e is still called with Record A preceding Record B.

4. **Zero-mode address selection:** With a nonnegative first-stage index, a zero mode byte, and selection byte `1` when inspected, the two calls to F-02a35f55d6ef use Record B offset `0x3C` and Record A offset `0x3C`, respectively, as their first addresses.

5. **Configuration mutation:** If the first zero-mode dependency call changes the first-stage index, Record A’s slot address uses the reread index rather than necessarily matching Record B’s slot.

6. **Shared second-stage result:** If the second-stage index is enabled and the selection byte is not `1` after Record B’s write, F-96ada6d600f3 is called once for that stage. Its saved four-byte return value is written to both selected slots.

7. **Independent second-stage results:** If the selection byte equals `1` at that decision, the dependency is called first for Record B’s conditionally negated field and then for Record A’s. The same retained slot index governs both writes.

8. **No final processing:** Counts `0` and `1` produce no F-0a29566d2e48 calls, quantization writes, or callback invocation by this function.

9. **More than two results:** A count of `3` still produces only two F-0a29566d2e48 calls and four explicit field updates.

10. **Quantization boundary:** A field containing `0x000007FF` becomes `0x00000000`; one containing `0x00000800` becomes `0x00001000`.

11. **Quantization wrap:** A field containing `0xFFFFF800` becomes `0x00000000` under the specified 32-bit unsigned addition and mask.

12. **Repeated returned pointer:** If both returned entries designate the same stable record and no dependency invalidates it, its fields are updated twice in the stated order.

## Uncertainties

The following are documented_only:

- Application meanings of configuration indices, record fields, modes, and slot contents.
- Valid index ranges and actual object extents.
- Exact target behavior for negating the minimum signed 32-bit input.
- Defined behavior, if any, for overflowing or invalid address calculations.
- Dependency errors, exceptions, nonlocal exits, and ownership changes.
- Whether dependency contracts permit retaining pointers to invocation-local storage.
- Whether returned records or the output array can be absent despite a count greater than one.
- Callback signature and ABI details beyond the visible indirect invocation.
- Concurrency, reentrancy, and synchronization requirements.

No recovery path is visible for invalid memory, malformed results, or dependency failure. Absence of a visible recovery path is not a guarantee of any particular fault or error-reporting mechanism.

## Evidence

The supplied static representation establishes the gate, field offsets, access widths, conditional paths, dependency argument order, repeated configuration and output-array reads, unsigned count comparison, ordered quantization writes, and final indirect invocation.

It does not supply dependency bodies, authoritative type definitions, ownership contracts, or runtime observations. Interpretations involving copying, coordinate rounding, transformations, or rendering remain inferred intent rather than required semantics.

No runtime tests have been performed.
