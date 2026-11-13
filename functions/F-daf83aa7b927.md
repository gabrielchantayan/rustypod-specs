<!-- Trusted provenance: F-daf83aa7b927; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-daf83aa7b927

## Purpose

F-daf83aa7b927 conditionally populates a caller-provided record using values obtained through opaque dependencies. It processes a sequence of presence flags, fixed-width values, variable-width values, and up to two delegated subrecords.

The established behavior is an ordered, nontransactional update:

- Each directly decoded value is stored before its associated validation or advancement call is checked.
- A zero result from that check terminates processing with failure.
- Earlier writes remain in place on failure.
- Fields excluded by a presence flag are not initialized by this function.
- Two delegated subrecord operations have no checked return value.

Interpreting the operation as parsing a bit-oriented metadata record is inferred intent. The encoding, field meanings, and correctness guarantees of the dependencies are not established.

## Interface

### Arguments

| Argument | Established role |
|---|---|
| First | Opaque 32-bit value forwarded to decoding, checking, and subrecord dependencies. It may identify input state, but its representation is unknown. |
| Second | Address of the writable destination record. Direct accesses extend through byte offset 0x280. |
| Third | Opaque 32-bit argument that is not used by the visible function. |
| Fourth | Opaque 32-bit value copied into a mutable local width slot before dependency calls begin. |

The direct-write footprint requires an accessible record extent of at least 0x281 bytes. That is not a complete safety bound: the delegated operations receive addresses inside the record, and their writable extents are unknown.

The destination must support the observed byte, 16-bit, and 32-bit accesses. Required alignment and permissible overlap with dependency-managed state are not established. The function performs no explicit pointer, capacity, alignment, or overlap checks.

### Return value

The return is 32 bits:

- **0x00000000:** the visible processing sequence reaches a success exit.
- **0xFFFFFFFF:** an explicitly checked dependency result is zero.

The latter bit pattern can represent −1 under a signed 32-bit interpretation. No finer error classification is exposed.

Success does not independently certify the success of either delegated subrecord operation, because their results are not checked.

### Local state passed to dependencies

Two addressable 32-bit slots exist for the duration of the call:

- A shared state slot starts at zero. Fixed-width decoding, variable-width decoding, and checking receive its address.
- A mutable width slot starts with the fourth argument. Variable-width decoding receives its address; subsequent checking receives its current value.

Neither slot is reset between fields. A dependency may change a slot it receives by address. The function does not expose either final slot value directly.

## Behavior

### Common decode/store/check rule

Each directly decoded field follows the same observable ordering:

1. Invoke the applicable decoding dependency.
2. Store its result into the destination field, narrowing to the destination width where applicable.
3. Invoke F-8a872239bf2c with the relevant width and the shared state address.
4. Continue only if that dependency returns a nonzero 32-bit result.

For fixed-width fields, F-d54421325734 receives the first argument, the requested width, and the shared state address. The subsequent check receives the same requested width.

For variable-width fields, F-2b01710262ca receives the first argument and the addresses of both mutable slots. The subsequent check receives the width slot’s value **after** that decoding call. Widths are not recomputed or bounded by this function.

Thus, failure checking occurs after the destination write. A failed check does not retract that write or restore dependency state.

The requested widths below describe dependency arguments, not independently proven input consumption. Actual cursor movement, bit ordering, and availability checks belong to the opaque dependencies.

### Destination layout

Offsets are byte offsets from the destination address. Ranges are inclusive. Descriptive names identify this contract’s groups, not established format terminology.

| Offset | Storage | Requested width or producer | Condition |
|---|---:|---|---|
| 0x000 | 1 byte | 1 | Always |
| 0x001 | 1 byte | 8 | Byte 0x000 is nonzero |
| 0x002–0x003 | 16 bits | 16, or F-892aa643a358 | Byte 0x000 is nonzero |
| 0x004–0x005 | 16 bits | 16, or F-892aa643a358 | Byte 0x000 is nonzero |
| 0x006 | 1 byte | 1 | After the first group |
| 0x007 | 1 byte | 1 | Byte 0x006 is nonzero |
| 0x008 | 1 byte | 1 | After the preceding group |
| 0x009 | 1 byte | 3 | Byte 0x008 is nonzero |
| 0x00A | 1 byte | 1 | Byte 0x008 is nonzero |
| 0x00B | 1 byte | 1 | Byte 0x008 is nonzero |
| 0x00C–0x00E | Three separate bytes | 8 each | Bytes 0x008 and 0x00B select this branch |
| 0x00F | 1 byte | 1 | After the preceding group |
| 0x010–0x011 | Two separate bytes | Variable-width, separately | Byte 0x00F is nonzero |
| 0x012 | 1 byte | 1 | After the preceding group |
| 0x014–0x017 | 32 bits | 32 | Byte 0x012 is nonzero |
| 0x018–0x01B | 32 bits | 32 | Byte 0x012 is nonzero |
| 0x01C | 1 byte | 1 | Byte 0x012 is nonzero |
| 0x01D | 1 byte | 1 | After the preceding group |
| Starting at 0x020 | Unknown extent | F-161f98159436 | Byte 0x01D is nonzero at the relevant branch |
| 0x148 | 1 byte | 1 | After the first delegated-group opportunity |
| Starting at 0x14C | Unknown extent | F-161f98159436 | Byte 0x148 is nonzero at the relevant branch |
| 0x274 | 1 byte | 1 | At least one reread delegated-group flag is nonzero |
| 0x275 | 1 byte | 1 | After the delegated-group section |
| 0x276 | 1 byte | 1 | After byte 0x275 |
| 0x277 | 1 byte | 1 | Byte 0x276 is nonzero |
| 0x278–0x27B | 32 bits | Variable-width | Byte 0x276 is nonzero |
| 0x27C–0x280 | Five separate bytes | Variable-width, separately | Byte 0x276 is nonzero |

Offsets 0x013 and 0x01E–0x01F have no direct writes. The same is true of other gaps outside the explicitly listed fields. No preservation guarantee is made for locations reachable by opaque dependencies.

Multibyte fields are stored as native 16-bit or 32-bit objects. Their byte order is not established by the supplied evidence.

### Initial conditional group

The function first writes and checks the one-bit-request field at offset 0x000. If its stored byte is zero, it skips offsets 0x001–0x005 without clearing them.

Otherwise it writes and checks the eight-bit-request field at offset 0x001. That stored byte selects one of two alternatives:

- If its comparison against −1 succeeds, two fixed-width 16-bit fields are decoded, stored, and checked in order at offsets 0x002 and 0x004.
- Otherwise, F-892aa643a358 is called with the byte-derived selector and destination addresses at offsets 0x002 and 0x004. Its return value is not checked.

The comparison’s signed-byte interpretation matters. On a signed eight-bit-byte interpretation, the all-ones byte represents −1 and selects the explicit 16-bit fields. The supplied representation does not independently establish the original target’s byte signedness; this branch must not be silently specified as an unsigned 255 comparison without that qualification.

### Intermediate flag groups

After the initial group, the function processes these groups in order:

1. Offset 0x006, followed by offset 0x007 only if the stored byte at 0x006 is nonzero.
2. Offset 0x008. When nonzero, this enables offsets 0x009, 0x00A, and 0x00B, in that order. A nonzero stored byte at 0x00B then enables the three eight-bit-request fields at offsets 0x00C, 0x00D, and 0x00E.
3. Offset 0x00F. When nonzero, this enables two separate variable-width values at offsets 0x010 and 0x011.
4. Offset 0x012. When nonzero, this enables the 32-bit field at 0x014, the 32-bit field at 0x018, and the one-bit-request field at 0x01C, in that order.

Every listed direct decode has its own store and check. Failure stops the sequence immediately; later fields are not processed.

The two variable-width results in the third group are each narrowed to one byte. No overflow rejection, saturation, range validation, or cross-field consistency check is visible.

### Delegated subrecords and their flags

The function writes and checks offset 0x01D, then reads that destination byte to decide whether to invoke F-161f98159436 with the first argument and the destination address at 0x020.

Next, it writes and checks offset 0x148, then reads that byte to decide whether to invoke the same dependency with the destination address at 0x14C.

Neither invocation receives the local shared state address or width-slot address as an explicit argument. Neither invocation’s return value is inspected.

After both opportunities, the function reads offset 0x01D again. If it is nonzero, offset 0x274 is enabled without requiring a read of offset 0x148 for this condition. If offset 0x01D is zero, the function reads offset 0x148 and enables offset 0x274 only if that byte is nonzero.

This final condition depends on the record’s then-current contents, not cached original decoding results. If a dependency changes these bytes through accessible aliases or other means, those changes can affect whether offset 0x274 is processed. Preservation of record flags by dependencies is not assumed.

### Final fields and optional tail

The function then writes and checks offset 0x275 and offset 0x276, in that order.

If the stored byte at 0x276 is zero, the function returns success immediately. Offsets 0x277–0x280 are not directly written on this path.

If it is nonzero, the function processes:

- One fixed-width one-bit-request field at offset 0x277.
- One variable-width result stored as 32 bits at offset 0x278.
- Five additional variable-width results, each narrowed to one byte, at offsets 0x27C through 0x280 in ascending order.

Each has its own subsequent check. Success is returned only after the final check returns nonzero.

## Side effects

### Destination mutation

The destination is caller-provided storage and is modified incrementally. The function does not initialize the full record, clear absent groups, or roll back failed processing.

A preexisting destination can therefore contain a mixture of:

- Newly written fields whose checks succeeded.
- The most recently written field whose check failed.
- Untouched values in skipped or not-yet-reached fields.
- Effects of opaque dependencies.

Presence flags are tested from destination memory after their checks. They are not merely tested from temporary decode results. The post-delegation condition explicitly rereads earlier flags.

### Input and external state

The first argument is repeatedly forwarded to dependencies. Those dependencies may advance input state or change other state, but their exact effects are unknown. Failure does not trigger a visible rewind.

No direct allocation, release, ownership transfer, global access, or external I/O is visible in this function. This does not establish their absence inside dependencies.

The destination remains caller-supplied; no visible operation releases it. Dependencies receive interior destination addresses and, where specified, addresses of stack-lifetime state slots. Whether they retain addresses or manage other resources is documented_only rather than assumed.

## Dependencies

| Opaque ID | Role established at this call site | Unestablished behavior |
|---|---|---|
| F-d54421325734 | Supplies fixed-width-request values; receives the first argument, a width, and shared state address | Extraction rules, cursor effects, failure signaling, globals |
| F-8a872239bf2c | Runs after each direct store; zero causes failure, nonzero permits continuation | Whether it checks availability, advances input, changes shared state, or performs other work |
| F-2b01710262ca | Supplies variable-width values; receives mutable width and shared state addresses | Encoding, supported widths, result range, width updates, failure representation |
| F-892aa643a358 | Handles the initial selector alternative using two interior destination addresses | Output extents, selector interpretation, errors, other side effects |
| F-161f98159436 | Handles either optional delegated subrecord | Subrecord size, input movement, error behavior, state preservation |

No dependency is assumed to preserve globals, avoid aliasing, or acquire or release memory.

## Acceptance examples

These are conditional acceptance scenarios derived from static control flow, not executed tests. Unless a scenario says otherwise, dependencies leave unrelated destination bytes unchanged.

1. **All optional groups absent.**  
   Let each encountered presence field produce zero and every check return nonzero. The function processes offsets 0x000, 0x006, 0x008, 0x00F, 0x012, 0x01D, 0x148, 0x275, and 0x276, then returns zero. It does not directly write optional payloads or offset 0x274. These are nine one-bit requests; actual input movement remains dependency-defined.

2. **Failure on the first field.**  
   The first fixed-width dependency returns a value whose low byte is stored at offset 0x000. If the immediately following check returns zero, the result is 0xFFFFFFFF. The offset 0x000 write has already occurred, and no subsequent field is processed.

3. **Failure midway through an explicit initial pair.**  
   Assume the initial presence flag is nonzero and the selector comparison selects explicit fields. If the check following the write at offset 0x004 fails, both 16-bit destination writes have occurred. Offset 0x006 is not processed.

4. **Mutable variable width and narrowing.**  
   Assume offset 0x00F enables its payload. If the first variable decoder returns 0x123 and changes the width slot to 11, offset 0x010 receives 0x23, and the following check receives width 11. The next variable decoder receives that same slot, including any retained value; the original fourth argument is not recopied.

5. **Optional nested bytes absent.**  
   If offset 0x008 is nonzero but offset 0x00B is zero, offsets 0x009–0x00B are processed while offsets 0x00C–0x00E remain unwritten by the direct sequence.

6. **Delegated result is not a direct failure channel.**  
   If a flag enables F-161f98159436, the function proceeds after that invocation without testing its return. An adverse return value alone does not establish that F-daf83aa7b927 returns failure.

7. **Reread flags control the extra field.**  
   If the destination bytes at both 0x01D and 0x148 are zero when the post-delegation condition runs, offset 0x274 is skipped, even if a previously decoded flag was nonzero before an intervening dependency changed it.

8. **Tail failure preserves the failing store.**  
   If offset 0x276 enables the tail and the check after writing offset 0x27E fails, the function returns 0xFFFFFFFF. The writes through 0x27E remain; offsets 0x27F and 0x280 are not processed.

## Uncertainties

The following remain **documented_only**:

- Domain-specific meanings of all record fields and selectors.
- Whether width arguments count bits, despite that being consistent with their use.
- Input byte order, bit order, cursor behavior, and exhaustion behavior.
- Native multibyte storage order and alignment requirements.
- Signed-byte interpretation of the initial selector comparison.
- Dependency result ranges, especially for nominal one-bit requests and variable-width values.
- Validity or bounds of the mutable width slot; this function imposes none.
- Whether the shared state slot represents an error, cursor, accumulator, or something else.
- Delegated subrecord extents and failure conventions.
- Dependency effects on globals, aliases, record contents, ownership, or retained addresses.
- A complete safety contract for invalid pointers, insufficient input, overlapping storage, or dependency failures outside the checked zero results.

Requested widths do not prove returned values fit those widths. Branches use the stored byte’s zero/nonzero value; they do not visibly enforce Boolean normalization.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. Established observations include argument forwarding, destination offsets, storage widths, dependency order, conditional branches, mutable local-slot use, and return bit patterns.

The parsing interpretation and descriptive grouping names are inferred intent, not independently established format semantics. Dependency internals and target-specific representation details were not supplied.

No runtime tests have been performed. Acceptance examples are conditional consequences of the visible control flow, not observed executions.
