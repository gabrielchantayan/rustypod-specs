<!-- Trusted provenance: F-a43e27fd9dd4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a43e27fd9dd4

## Purpose

F-a43e27fd9dd4 selects one of three block-processing paths using a byte in a caller-provided state object. It invokes F-ce3bd5ea2520 for transformation work and an unmapped external dependency for apparent byte transfers.

The established processing ratio is **128 supplied length units per 16 bytes of buffer data**. A bit-count interpretation is plausible but not independently established.

The accepted modes are:

- **Mode 1:** independent transformation calls.
- **Mode 2:** wordwise exclusive-or chaining, followed by a final transfer into the state object.
- **Mode 3:** bitwise output updates and feedback into a shifting 16-byte state region.

The available evidence does not establish a cryptographic algorithm, key format, or security property. This contract distinguishes the wrapper’s direct behavior from conditional outcomes requiring dependency behavior.

## Interface

### Arguments and result

The function takes five arguments in this order:

| Argument | Established role |
|---|---|
| State pointer | Addresses a mode byte at offset 0 and a 16-byte state region at offsets 4–19. |
| Context pointer | Addresses a status byte, a 32-bit field at offset 76, and dependency-specific regions beginning at offsets 80 and 320. |
| Input pointer | Addresses input blocks. Mode 2 directly reads four 32-bit words per block. |
| Supplied length | Unsigned 32-bit quantity, also tested using its signed 32-bit interpretation. |
| Output pointer | Addresses output storage. No explicit null or capacity check is performed. |

The result is a signed 32-bit integer:

| Result | Meaning |
|---|---|
| −5 | Null state/context pointer, rejected context status, or unsupported mode. |
| 0 | An early no-work return, or an accepted mode with no complete block. |
| Positive multiple of 128 | Supplied length rounded down to a multiple of 128. |

A zero result does not guarantee absence of writes or dependency calls: mode 2 performs its final transfer even when a positive length selects zero blocks.

### Widths and storage layout

The supplied representation establishes 32-bit integer words and 32-bit pointer representations. All offsets below are byte offsets.

| State offset | Size | Direct use |
|---|---:|---|
| 0 | 1 byte | Mode selector. |
| 1–3 | 3 bytes | Not directly accessed. |
| 4–19 | 16 bytes | Initial chaining data in mode 2; live feedback state in mode 3. |

| Context offset | Size | Direct use |
|---|---:|---|
| 0 | 1 byte | Status; exactly 1 causes rejection. |
| 76 | 4 bytes | Reloaded before every transformation call. |
| 80 | Unknown extent | First transformation argument in modes 1 and 2. |
| 320 | Unknown extent | First transformation argument in mode 3. |

The wrapper also uses a contiguous local 16-byte region divided into four 32-bit slots. After reading the mode, but before dispatching its processing path, it initializes those slots in this order:

1. State pointer representation.
2. Context pointer representation.
3. Input pointer representation.
4. Original supplied length.

The output pointer is not part of this initialization. These initial contents are behaviorally relevant in mode 3; they are not merely uninitialized output scratch.

### Storage obligations

For a positive accepted length, let the block count be the supplied length divided by 128, rounded down. Nominal input and output extents are each 16 times that count.

Callers must provide storage valid for direct accesses and for the dependencies’ actual accesses. In particular:

- Mode 2 directly accesses input and chaining data as 32-bit words.
- Modes 2 and 3 use the state region at offsets 4–19.
- Mode 3 directly reads and writes output bytes.
- Mode 2 with a positive length below 128 still passes the address 16 bytes before the initial output pointer to the transfer dependency.

Required alignment, dependency access extents, and supported overlap arrangements are **documented_only** unknowns. Pointer arithmetic is not validated for wraparound or object bounds.

## Behavior

### Validation and dispatch

Validation has the following precedence:

1. If either the state pointer or context pointer is null, return −5 without dereferencing either.
2. Otherwise, read the context status byte at offset 0.
3. If that byte is exactly 1, return −5.
4. If the input pointer is null, the supplied length is zero, or the supplied length has its high bit set, return 0.
5. Calculate the complete-block count, read the state mode byte, and initialize the four local slots described above.
6. Dispatch modes 1, 2, or 3. Any other mode returns −5.

Consequently, invalid state/context arguments take precedence over no-work conditions. Conversely, an early no-work condition prevents the mode byte from being examined.

Context status values other than 1 are not rejected by the status check. The output pointer is not validated.

Status and mode are not rechecked during processing. Dependency changes to those bytes therefore do not cause redispatch or another status rejection. The context field at offset 76, however, is reread for every transformation invocation.

### Block calculation and return value

Only lengths from 1 through 2,147,483,647 reach processing. For these values:

- Block count is the integer quotient of length divided by 128.
- Incomplete trailing length units are ignored.
- Each block advances local input and output positions by 16 bytes.
- Normal return is the block count multiplied by 128.

For the largest accepted length, 2,147,483,647:

- Block count is 16,777,215.
- Nominal buffer extent is 268,435,440 bytes.
- Return value is 2,147,483,520.

These values fit the relevant positive signed 32-bit range. This arithmetic does not establish that corresponding addresses or storage are valid.

### Mode 1: direct transformation

For each complete block, in increasing buffer-address order, invoke F-ce3bd5ea2520 with:

1. Context address plus 80.
2. The current 32-bit value at context offset 76.
3. Current input block address.
4. Current output block address.

After each call returns, advance both buffer positions by 16 bytes.

The wrapper does not directly update the state region in this mode. It initializes the local four-slot region but does not pass that region to the transformation.

For a positive length below 128, no transformation call occurs and the result is 0.

### Mode 2: wordwise chaining

The initial chaining source is the state region at offsets 4–19.

For each complete block, the wrapper prepares the local 16-byte region as four consecutive 32-bit exclusive-or results. Each slot receives the corresponding input word exclusive-ored with the corresponding chaining word.

The four slot assignments occur in ascending word order. The evidence does not require a particular relative load order between the two operands within a single exclusive-or expression. All four assignments complete before the transformation call.

F-ce3bd5ea2520 then receives:

1. Context address plus 80.
2. A fresh read of the 32-bit field at context offset 76.
3. The local 16-byte region.
4. Current output block address.

After the call, the current output block becomes the next chaining source. Input and output positions advance by 16 bytes. Thus, later chaining values are read from the preceding output block in memory, not from a preserved private copy.

After the loop, the wrapper invokes the transfer dependency with:

- Destination: state address plus 4.
- Source: final output position minus 16 bytes.
- Size: 16.

This invocation is unconditional within mode 2.

With at least one block, its source is the final output block. With a positive length from 1 through 127, no transformation occurs, but the source is still 16 bytes before the initial output pointer. If the transfer returns normally, the wrapper returns 0.

The dependency’s return value is ignored.

### Mode 3: persistent temporary storage

Mode 3 uses the initialized local four-slot region as the final argument to F-ce3bd5ea2520.

Before any transformation call, its slots contain the original state pointer, context pointer, input pointer, and supplied length, respectively. Advancing the active input position does not itself update the pointer representation stored in the third slot.

The wrapper does **not** clear or reinitialize this region:

- Between bit iterations.
- Between blocks.
- Before interpreting its first slot after a transformation call.

After every call, the wrapper reads bit 7 of the first 32-bit slot, regardless of whether the dependency wrote that slot fully, partially, or not at all.

Accordingly:

- A dependency may read the initially stored pointer representations and length.
- Unwritten bits can retain their initial values.
- Later calls can observe contents left by earlier calls.
- A first-slot bit left untouched can continue influencing output across block boundaries.

Calling this region solely “transformation output” would therefore omit established behavior. Its actual role depends on the dependency’s unknown read/write contract.

### Mode 3: block and bit processing

Each block begins with a transfer-dependency invocation:

- Destination: current output block.
- Source: current input block.
- Size: 16.

The wrapper then executes exactly 128 bit iterations.

Each iteration first invokes F-ce3bd5ea2520 with:

1. Context address plus 320.
2. A fresh read of the 32-bit field at context offset 76.
3. The live state region at state offset 4.
4. The persistent local 16-byte region.

Next, the wrapper obtains bit 7 of the first local 32-bit slot. That single bit controls whether the current output bit is toggled.

Output positions are visited in this order:

- First output byte, bit 7 through bit 0.
- Second output byte, bit 7 through bit 0.
- Continuing through the sixteenth byte.

For each iteration, the observable direct-update sequence is:

1. Read the selected output byte and write it back exclusive-ored with the selected one-bit mask.
2. Update state bytes 0 through 14 in increasing order within the 16-byte state region.
3. Reread the selected output byte.
4. Update state byte 15 using the selected bit from that second output-byte read.

Each of the first 15 state-byte updates retains the low eight bits of its old value shifted left once, combined with the old high bit of the following byte. Because updates proceed in increasing order, the following byte has not yet been updated by that iteration’s direct shift.

The final state byte receives its previous value shifted left once, truncated to a byte, with the feedback bit placed in bit 0.

The feedback bit is therefore taken from output **after** the exclusive-or write and **after** the first 15 state-byte writes. If output overlaps state, those intervening writes can change the value reread for feedback. A simple nonoverlapping-buffer interpretation must not be applied universally.

Each full block causes one transfer invocation, 128 transformation invocations, 128 direct output-byte updates, and 128 complete state shifts. Buffer positions then advance by 16 bytes.

With a positive length below 128, mode 3 performs no transfer, transformation, or direct output/state update and returns 0.

## Side effects

### Direct effects and ownership

The wrapper directly reads validation fields, mode, per-call context values, mode-2 input/chaining words, and mode-3 temporary/state/output values.

Its direct writes are:

- Initialization of the local four-slot region.
- Mode-2 exclusive-or results in that region.
- Mode-3 output-byte updates.
- Mode-3 state-byte updates.

Output writes in modes 1 and 2 occur through F-ce3bd5ea2520. The final mode-2 state transfer and initial mode-3 block transfer occur through the unmapped dependency.

No explicit allocation, release, or caller-buffer ownership transfer appears in the wrapper. Local pointer advancement does not update caller-held pointer variables.

This does not establish that dependencies avoid allocation, release, pointer retention, global changes, or writes through aliases.

### Failure and partial progress

Neither dependency’s return value is used to decide success or failure. There is no established rollback or dependency-error propagation path.

The −5 result represents explicit validation or mode rejection, not a general memory or transformation error. Invalid storage or abnormal dependency termination can prevent a normal return and can leave partial output, temporary contents, or state updates.

Dependency effects can influence later work. Examples include changes to the reread context field, preceding output blocks used for chaining, live feedback state, and persistent local slots.

## Dependencies

### F-ce3bd5ea2520

The visible interface has four arguments:

1. Context-relative address.
2. A 32-bit context-derived value.
3. Input-like address.
4. Output-like or local-storage address.

Modes 1 and 2 use context offset 80; mode 3 uses offset 320.

In mode 2, all four local slots are overwritten with exclusive-or results before each call. In mode 3, the dependency receives initialized, persistent local storage whose existing contents may matter. The wrapper unconditionally uses bit 7 of its first slot after each call.

A 16-byte block transformation is inferred from usage. Exact access extents, transformation, partial-write behavior, return semantics, ownership effects, and other side effects are **documented_only** unknowns.

### Unmapped transfer dependency

No opaque function ID was supplied for this dependency; none is invented.

The wrapper supplies destination, source, and the value 16. Byte-copy intent is inferred from usage. Exact transfer semantics, overlap support, access ordering, failure behavior, and unrelated side effects are **documented_only** unknowns.

## Acceptance examples

These are static contract predictions, not executed tests. Unless stated otherwise, storage is valid and nonoverlapping, the context status is not 1, and dependencies return normally. Additional dependency assumptions are stated where necessary.

### Validation precedence

- Null state pointer and zero length: return −5.
- Nonnull state/context, status 1, null input: return −5.
- Valid state/context, null input, unsupported mode: return 0 without reading the mode.
- Valid state/context, nonnull input, length 2,147,483,648: return 0 without reading the mode.
- Valid state/context, nonnull input, length 128, unsupported mode: return −5 without block processing.

### Rounding and call counts

For length 255, each accepted mode selects one block and normally returns 128:

- Mode 1: one transformation call.
- Mode 2: one transformation call, then one final transfer.
- Mode 3: one initial transfer and 128 transformation calls.

The remaining 127 length units cause no partial-block processing.

### Mode 2 chaining and zero-block behavior

For two blocks, assuming each transformation writes a complete 16-byte output and does not otherwise alter relevant buffers:

- The first transformation receives the first input block exclusive-ored wordwise with initial state.
- The second receives the second input block exclusive-ored wordwise with first output.
- If the final dependency copies the requested bytes without other relevant writes, final state equals second output.

For length 1, no transformation occurs. The final transfer still receives a source 16 bytes before the initial output pointer. Normal return is 0.

### Mode 3 retained contents

If a transformation does not write the first local slot, the wrapper still uses that slot’s bit 7.

For the first iteration, if no earlier dependency effect has changed the initialized slot, this is bit 7 of the original state pointer representation.

If an iteration leaves the first slot with bit 7 set and subsequent calls do not alter that bit, subsequent iterations continue selecting a toggle. This persistence extends across a block boundary unless another dependency effect changes the stored bit.

### Mode 3 output and feedback

For an individual wrapper update:

- Generated bit 0 preserves the selected output bit at that update.
- Generated bit 1 toggles it.
- With nonoverlapping output/state, feedback uses the resulting output bit, not the original input bit or generated bit alone.

A whole-block equality prediction requires stronger assumptions: the initial transfer copies the input correctly, transformation calls make **no other changes to output**, and no other aliasing effect modifies output. Under those conditions, 128 generated zero bits leave final output equal to the copied input.

Without the no-other-output-writes condition, 128 generated zero bits establish only that the wrapper’s exclusive-or operations do not toggle output. They do not establish final equality with input.

## Uncertainties

### Established static evidence

Validation precedence, field offsets, 32-bit widths, mode dispatch, block calculation, pointer strides, dependency arguments, local-slot initialization and persistence, unconditional mode-2 final transfer, and mode-3 direct update ordering are established.

### Inferred intent

The length likely counts bits. Mode 2 resembles block-output chaining. Mode 3 resembles a bitwise feedback transformation. The unmapped dependency appears intended to copy bytes.

These interpretations do not establish an algorithm name or stronger dependency guarantees.

### documented_only unknowns

- Public ABI and externally defined length unit.
- Dependency transformations, complete memory footprints, and preservation guarantees.
- Meaning of the context field at offset 76.
- Context-region extents and alignment requirements.
- Transfer dependency identity and exact semantics.
- Supported aliasing and in-place arrangements.
- Dependency ownership, lifetime, retention, and global-state effects.
- Whether callers exclude positive sub-block mode-2 lengths.
- Whether F-ce3bd5ea2520 reads initial local contents or overwrites every subsequently consumed bit.

## Evidence

This contract is based on the supplied static representation and opaque dependency mapping. Dependency implementations were not provided.

The initialization of all four local slots precedes mode dispatch. Mode 3 repeatedly passes that same region and reads its first slot without intervening wrapper reinitialization. The output-feedback read follows the first 15 state-byte updates.

No runtime tests have been performed. Acceptance examples are conditional deductions from static evidence, not observed execution results.
