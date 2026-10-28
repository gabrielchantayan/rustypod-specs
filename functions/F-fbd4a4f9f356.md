<!-- Trusted provenance: F-fbd4a4f9f356; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fbd4a4f9f356

## Purpose

F-fbd4a4f9f356 stores a value from a source record into caller-supplied destination storage. Flags in the source record select a one-byte, two-byte, four-byte, or two-word representation.

The established behavior is a width-selected store, not a general-purpose numeric conversion service. There is no range check, destination-capacity check, error result, allocation, or dependency call in the supplied representation.

**Inferred intent:** the flags select the width of an integer destination, with the two-word form representing a sign-extended value. The meanings of the record and individual flags outside this operation are unknown.

**Evidence limitation:** this contract is based exclusively on static inspection of the supplied representation. No runtime tests have been performed. The byte counts below use the apparent target data model: eight-bit bytes, 16-bit short integers, and 32-bit integers. The supplied material does not independently establish that model or the target byte order.

## Interface

### Inputs

The operation takes two arguments:

| Argument | Role | Required storage under the apparent target model |
|---|---|---|
| Source-record address | Locates the flags and value to be stored | A readable 32-bit flags field at byte offset `0x18`, and a readable signed 32-bit value field at byte offset `0x3C` |
| Destination address | Receives the selected representation | A writable region of 1, 2, 4, or 8 bytes, depending on the flags |

The source-record address is represented by an integer-typed argument in the supplied material. Its treatment as an address, including integer-to-address interpretation, is target-dependent. It is not a record passed by value.

The destination is represented by a pointer to an integer-sized element. That apparent pointer type does not limit writes to four bytes: one branch writes two consecutive integer-sized elements.

The relevant source layout is:

| Byte offset | Field | Interpretation |
|---|---|---|
| `0x18` | Flags | Unsigned integer; bits selected by masks `0x400`, `0x100`, and `0x80` affect this operation |
| `0x3C` | Value | Signed integer; supplies the value for every branch |

No other source fields are visibly accessed.

### Output

The operation has no return value. Its result is communicated through destination memory.

Under the apparent target model, the destination layout is one of:

- One byte at destination offset zero.
- One 16-bit value at destination offset zero.
- One 32-bit value at destination offset zero.
- A 32-bit value at destination offset zero, followed by another 32-bit value at destination offset four.

The selected width is not returned separately.

### Preconditions

For defined execution on the intended target:

- Both source fields must be readable.
- The destination must have enough writable storage for the selected branch.
- The source and destination accesses must meet the target’s alignment and memory-access requirements.
- Address formation for the source fields and any second destination word must be valid.
- Concurrent modification or observation must obey whatever synchronization requirements apply to the surrounding program.

The operation does not validate these conditions. In particular, it does not accept a capacity argument or check either address for null.

## Behavior

### Source reads and width selection

The visible access order is:

1. Read the source flags field.
2. Read the source value field.
3. Select a destination representation using the captured flags.
4. Perform the selected destination write or writes.

Both source fields are read before any destination write. The value read is unconditional: even a one-byte destination selection requires access to the complete source value field.

The selection priority is:

| Priority | Condition | Selected representation |
|---|---|---|
| First | Flag `0x400` is set | One-byte narrowing |
| Second | Flag `0x400` is clear and flag `0x100` is set | Two-byte narrowing |
| Third | Flags `0x400` and `0x100` are clear, and flag `0x80` is set | Two-word form |
| Last | All three controlling flags are clear | One-word form |

Flags outside these masks do not affect selection in the visible body.

This priority resolves combinations explicitly. Setting `0x400` suppresses the effects of both other controlling flags. When `0x400` is clear, setting `0x100` suppresses the effect of `0x80`.

### One-byte narrowing

When flag `0x400` is set, the operation converts the captured source value to the target’s character-sized integer type and stores that result at the destination address.

Under the apparent eight-bit-byte model, exactly one destination byte is written. The operation does not clear or otherwise update following bytes.

**Width and conversion limitation:** the supplied representation uses a plain character-sized type, whose signedness is target-dependent. For values outside its representable range, the precise language-level conversion behavior is not established by this material alone. Retention of the low eight bits is the apparent target behavior, but should not be treated as an independently verified portability guarantee.

### Two-byte narrowing

When `0x400` is clear and `0x100` is set, the operation converts the captured source value to a signed short integer and stores that result at the destination address.

Under the apparent 16-bit-short model, exactly two destination bytes are written. Following bytes are untouched by the visible operation.

Values representable by the target short type are preserved numerically. For values outside that range, the supplied material does not establish a portable conversion rule. Low-16-bit retention is the apparent target interpretation, subject to the target’s signed-conversion behavior.

### One-word form

When all three controlling flags are clear, the operation stores the captured signed source value as one integer-sized destination element.

Under the apparent 32-bit-integer model:

- Exactly four destination bytes are written.
- The stored integer value equals the captured source value.
- No second destination word is written.
- Existing bytes beyond the first word are not cleared.

### Two-word form

When `0x400` and `0x100` are clear and `0x80` is set, the operation performs two ordered integer-sized stores:

1. Store the captured source value in the first destination word.
2. Store the result of a signed right shift of that same captured value by 31 positions in the second destination word.

Under a 32-bit integer model with arithmetic signed right shift, the second word is:

- Zero for a nonnegative source value.
- Negative one, represented by all bits set on a two’s-complement target, for a negative source value.

That interpretation yields a sign-extension pair. The first word is at the lower destination address; the extension word follows it.

The visible operation does **not** perform a single indivisible 64-bit store. It exposes two stores in sequence. A concurrent observer or memory-mapped destination must not assume the pair becomes visible atomically.

The second word is derived from the already captured value. The source value field is not visibly reread between the stores.

**Signed-shift limitation:** arithmetic right shift for negative values is an apparent target property, not established independently by the supplied representation. The fixed shift count of 31 is explicit; the conclusion that it extracts a 32-bit sign extension depends on the integer model.

### Overlapping source and destination

The visible body contains no overlap check.

Because both source fields are read before destination mutation, later destination writes do not alter the captured flags or value used by this invocation. If the destination overlaps a source field, that field can nevertheless be modified as an output side effect.

This ordering does not establish unrestricted support for every overlapping address. Alignment, type-access rules, address validity, and target semantics remain prerequisites.

## Side effects

The only explicit mutations are the selected destination stores.

Under the apparent target model, the write footprint is exactly 1, 2, 4, or 8 bytes. There is no visible destination read, zero-filling beyond the selected footprint, or preservation step that first copies destination contents elsewhere.

No explicit allocation, deallocation, ownership transfer, global-variable access, logging, callback, or external function call appears. The caller retains responsibility for the lifetime of both source and destination storage.

Aliasing can make a destination write modify the source record or another caller-visible object. Accordingly, “destination only” describes the explicit write address, not a guarantee that the source record remains unchanged.

No error status is produced. Invalid memory, insufficient capacity, unsupported alignment, or invalid address arithmetic has no defined recovery path in the supplied body. The material does not establish whether a particular target would fault, partially write, or behave otherwise in such circumstances.

The access ordering above describes the supplied representation. It is not an additional hardware memory-ordering, synchronization, or volatile-access guarantee.

## Dependencies

No opaque function dependency is invoked by F-fbd4a4f9f356 in the supplied body.

The behavior depends on target properties rather than unknown callees:

- Byte, short-integer, and integer widths.
- Integer and pointer representation.
- Signed narrowing-conversion rules.
- Signed right-shift behavior for negative values.
- Byte order.
- Alignment and memory-access rules.
- Validity of the supplied addresses and storage lifetimes.

There is no callee whose preservation of global state or ownership behavior must be assumed.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not recorded test results.

Unless stated otherwise, examples assume eight-bit bytes, 16-bit short integers, 32-bit integers, valid aligned storage, and no concurrent interference.

| Flags | Source value | Required observable result |
|---|---:|---|
| `0x000` | 42 | Store 42 in the first 32-bit destination word; leave bytes after that word unchanged |
| `0x080` | 42 | Store 42 in the first word, then zero in the second word |
| `0x100` | 42 | Store 42 in the first 16-bit destination element; leave following bytes unchanged |
| `0x400` | 42 | Store 42 in the first destination byte; leave following bytes unchanged |
| `0x180` | 42 | Select the two-byte form, not the two-word form |
| `0x480` | 42 | Select the one-byte form, not the two-word form |
| `0x500` | 42 | Select the one-byte form, not the two-byte form |
| `0x580` | 42 | Select the one-byte form despite all three controlling flags being set |
| `0x001` | 42 | Select the one-word form; this unrelated flag does not change selection |

Additional conditional cases:

- **Negative sign extension:** with arithmetic signed right shift and flags `0x080`, a source value of −7 produces a first word of −7 followed by a second word of −1. The first store precedes the second.
- **Signed lower boundary:** under the same assumptions, the minimum signed 32-bit value produces that value in the first word and −1 in the second.
- **Zero:** flags `0x080` and a source value of zero produce two zero words.
- **Narrowing beyond range:** with low-bit-retaining narrowing, flags `0x400` and source value 4660 produce destination byte `0x34`. Without that target conversion assumption, this exact byte result remains documented_only.
- **Preserved trailing storage:** for a valid destination initialized with distinguishable trailing bytes, the one-byte branch changes no bytes after offset zero, and the two-byte branch changes no bytes after offset one.
- **Read-before-write overlap:** if a valid destination overlaps the source flags field, selection still uses the flags captured before the first destination write.
- **No short-read shortcut:** selecting a one-byte output does not remove the requirement that the source value field be readable in its full integer width.

No exact multi-byte byte sequence is required without a byte-order assumption. Numeric word values and their destination offsets remain the relevant layout requirements.

## Uncertainties

The following items are **documented_only**:

1. **Target data model.** The apparent widths are 8, 16, and 32 bits, but no independent ABI description accompanies the supplied material.
2. **Narrowing outside the destination range.** Exact results depend on target conversion rules, especially for signed short integers and plain character-sized integers.
3. **Negative right shift.** The sign-extension interpretation requires arithmetic right shift.
4. **Byte order.** The first and second word locations are established; their internal byte order is not.
5. **Address representation and overflow.** The integer-typed source address and fixed-offset address calculations have no documented boundary or wraparound policy.
6. **Exceptional memory behavior.** Fault type, partial completion, and recovery behavior are not specified.
7. **Concurrency.** The body provides no visible synchronization or guarantee of atomic publication.
8. **Broader semantics.** Field names, record ownership conventions, and the meanings of the flags elsewhere are unknown.

## Evidence

Established static evidence comprises two source-field reads, three prioritized flag checks, and four possible destination write footprints. The two-word branch explicitly places the original value before the shifted value. Every branch finishes without a returned result.

The description of sign extension is inferred from the signed shift by 31 and the apparent 32-bit integer model. The interpretation of the flags as destination-width selectors is inferred from their observable effects.

No runtime execution, tests, ABI verification, disassembly comparison, or external dependency inspection has been performed.
