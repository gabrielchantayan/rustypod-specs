<!-- Trusted provenance: F-98f82ca8bdf3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-98f82ca8bdf3

## Purpose

F-98f82ca8bdf3 reads a length-distribution description and an associated sequence of 32-bit values, expands the distribution into ordered lengths, generates integer codes, and constructs per-length lookup metadata.

**Inferred intent:** the resulting structures resemble a canonical variable-length-code decoding table for lengths 1 through 16. This interpretation explains the arrangement of counts, codes, and lookup ranges; it does not establish a particular file format, coding standard, or validation policy.

**Established behavior:** the function performs fixed-offset memory accesses and calls F-fbfa6017b0f5 to obtain input words. It does not explicitly validate count ranges, total capacity, code-space consistency, pointer validity, or an empty distribution. Its behavior on malformed data must not be replaced by an assumed validation error.

## Interface

### Invocation and result

The function takes no explicit arguments and returns no explicit value. Its observable results are dependency calls and writes through globally supplied pointers.

There is no explicit success indicator, error return, or rollback mechanism.

### Required external state

The function uses three global pointer sources:

1. **Context pointer.** Captured at entry. The referenced context contains:
   - At byte offset `0x44`: a pointer to the input/table block.
   - At byte offset `0x48`: a pointer to the output metadata block.
2. **Expanded-length workspace pointer.** Captured after input acquisition.
3. **Generated-code workspace pointer.** Captured after the expanded-length sequence and its terminator have been written.

The pointer values are represented as 32-bit quantities in the supplied evidence. Integer entries occupy four bytes. Byte order is not established by this function.

### Input/table block layout

Relative to the pointer obtained from context offset `0x44`:

| Byte offset | Meaning |
|---|---|
| `0x04` through `0x40` | Sixteen four-byte counts, indexed by lengths 1 through 16 |
| `0x90 + 4j` | Input value at zero-based position `j` |

The function does not initialize the word at offset zero or the gap between the count region and the value region.

Counts are used as signed 32-bit integers for positivity tests, arithmetic, and loop bounds. Associated input values are stored as opaque four-byte words; this function does not interpret their contents.

### Output metadata layout

The output block is addressed in four-byte words. For each length `l` from 1 through 16:

| Word index | Meaning |
|---|---|
| `0` | Last processed length whose count was nonzero, initially set to 1 |
| `l + 1` | Last generated code for that length, or `-1` for a zero count; one selected entry is subsequently incremented |
| `l + 0x25` | First generated code for that length, written only when its count is nonzero |
| `l + 0x49` | Starting position in the ordered value sequence, written only when its count is nonzero |

The greatest directly addressed word index is `0x59`, so the metadata block must make at least 360 bytes accessible. This is a storage requirement, not a claim that every byte is initialized.

### Preconditions for the ordinary valid-data path

The following are sufficient operational assumptions for the ordinary path described below:

- All required pointer sources and accessed storage are valid and appropriately aligned.
- Counts are nonnegative, with a positive total.
- Count accumulation, index arithmetic, code increments, and shifts remain representable under the execution environment’s integer rules.
- The input/table block has room for all acquired values.
- The expanded-length workspace has room for every expanded length plus one zero terminator.
- The generated-code workspace has room for every generated code.
- Writable regions do not destructively overlap one another, the context, or pointer sources.
- There is no concurrent mutation of the accessed state.

No checks enforce these assumptions. In particular, the source does not establish a maximum total of 256 or enforce that the count distribution describes a valid prefix code.

## Behavior

### 1. Acquire sixteen counts

The function captures the context pointer once, then makes exactly sixteen calls to F-fbfa6017b0f5.

For each length from 1 through 16, in increasing order:

1. Call the dependency.
2. Add the returned integer to a running signed total.
3. Read the input/table pointer from the captured context’s `0x44` field.
4. Store the returned four-byte count in the slot for that length.

The pointer field is read **after** each call. The function therefore does not necessarily write all sixteen counts to the same block if the dependency changes that field.

The total is accumulated from the returned values, not by rereading the stored count slots.

### 2. Acquire the associated values

If the accumulated total is positive, the function calls F-fbfa6017b0f5 that many additional times.

For each zero-based position, it obtains a word, reloads the input/table pointer from context offset `0x44`, and writes the word at offset `0x90 + 4j`.

If the total is zero or negative, there are no value-acquisition calls.

For a positive, representable total `N`, the ordinary dependency-call count is therefore `16 + N`. For a nonpositive total, it is 16, provided count accumulation itself has well-defined execution semantics.

The dependency’s values may affect the pointer field or other global state. No preservation guarantee is assumed.

### 3. Expand the current counts into a length sequence

After all dependency calls finish, the function captures the expanded-length workspace pointer and separately reads the current input/table pointer from the context. This latter pointer is retained for subsequent count accesses.

It visits lengths 1 through 16. For each positive count, it appends that length to the workspace once per count. Zero and negative counts append nothing. After all lengths have been processed, it writes a single zero terminator immediately after the last appended entry.

With stable nonnegative counts, this produces an ascending sequence such as:

- Counts of two at length 2 and one at length 4 produce lengths `2, 2, 4, 0`.

The number of expanded entries depends on the counts present **at this stage**, not necessarily on the earlier accumulated total. No equality check reconciles those quantities.

The expansion loop rereads the relevant count while deciding whether to continue. Aliasing or external mutation can therefore affect its iteration count; treating each count as a one-time snapshot is valid only under the stated stability assumptions.

### 4. Generate codes

The function captures the generated-code workspace pointer. It reads the first expanded length, initializes the current code to zero, and begins generating entries without first checking whether that first length is the terminator.

For an ordinary nonempty sequence:

- The first generated code is zero.
- Each entry is stored before the following expanded length is examined.
- After a store, the current code is incremented by one.
- When the following length equals the current length, generation normally proceeds with that incremented code.
- When the following length is a larger positive value, the current length advances one unit at a time, and the current code is left-shifted once for each advance.
- When a transition observes a following length of zero, generation ends and metadata construction begins.

Thus, between consecutive occupied lengths separated by `d`, the next group begins with the value obtained by taking one more than the preceding group’s last code and shifting it left `d` times.

#### The 256 comparison is not a capacity check

There is an additional transition condition when the index of the following entry is greater than `0x100`.

If the following length equals the current length and that next index is at most 256, the function continues directly. If the index is greater than 256, it instead enters the transition handling even though the lengths are equal.

For an equal positive length, that transition handling first increments the current length and shifts the code. It then continues until the current length equals the target. Because it has already advanced beyond an equal target, ordinary bounded progression cannot reach that target again. Eventual behavior depends on arithmetic and execution details not established here.

Consequently:

- This comparison does not stop generation at 256 codes.
- It does not return an error for oversized input.
- Some distributions with more than 256 entries can pass this point normally, while a sufficiently late same-length continuation takes the problematic path.

#### Empty distributions are not safely short-circuited

If the expanded sequence is empty, only workspace entry zero is initialized to the terminator. Code generation nevertheless stores a code and reads expanded-length entry one.

Behavior then depends on uninitialized or preexisting workspace contents. There is no established empty-table success result.

### 5. Construct per-length metadata

Once code generation reaches its terminator-handling path, the function reads the metadata pointer from context offset `0x48`. It initializes metadata word zero to 1 and the running position in the value sequence to zero.

Lengths 1 through 16 are processed in order.

For a count equal to zero:

- Write `-1` to the length’s last-code slot.
- Leave that length’s first-code and starting-position slots untouched.

For any count not equal to zero:

1. Store the current running position in the length’s starting-position slot.
2. Read the generated code at that position and store it as the first code.
3. Compute the position of the last entry as the current position plus the count minus one.
4. Advance the running position by the count.
5. Read the generated code at the computed last position and store it as the last code.
6. Set metadata word zero to the current length.

This stage distinguishes zero from nonzero, unlike expansion, which distinguishes positive from nonpositive. A negative count therefore takes the populated-length metadata path despite having contributed no expanded entries. Negative counts are not rejected and can lead to negative or otherwise invalid code-workspace indices.

After all sixteen lengths, the function reads the length recorded in metadata word zero and increments that length’s last-code slot by one.

For ordinary nonnegative, nonempty data, the recorded length is the greatest populated length. Its stored upper code endpoint becomes exclusive; other populated lengths retain inclusive last-code values.

## Side effects

The function can modify:

- Sixteen count slots, potentially across different input/table blocks if the pointer field changes during acquisition.
- The acquired-value region, again using a freshly read pointer for each store.
- The expanded-length workspace, including one zero terminator.
- The generated-code workspace.
- Selected fields of the metadata block.
- Any additional state modified by F-fbfa6017b0f5.

Writes are incremental. No staging or transactional commit is present. If execution fails during acquisition or construction, earlier writes have already occurred.

No allocation, deallocation, or explicit ownership transfer appears in the function. It uses externally supplied storage. The dependency’s ownership behavior is unknown.

Workspace contents beyond the written length sequence, terminator, and generated-code entries are not cleared. Metadata slots for absent lengths are only partially initialized. Consumers must not assume the function produces a fully zeroed block.

The function does not copy acquired values into the metadata block or use those values to generate codes. Their ordering is preserved by acquisition; the generated lookup positions refer to corresponding positions only under a consistent distribution and stable storage.

## Dependencies

### F-fbfa6017b0f5

**Established use:**

- Called with no explicit arguments.
- Supplies sixteen count words.
- Supplies an additional number of opaque value words when the accumulated count total is positive.
- Calls occur sequentially, with the corresponding store completed before the next call.

**Unknown contract:**

- Input source and encoding.
- Whether the return represents a byte, a wider integer, or an error sentinel.
- End-of-input and failure behavior.
- Global-state preservation.
- Memory ownership or release behavior.
- Whether it changes context fields, global pointer sources, or previously written memory.

The caller does not explicitly inspect an error status. Any interpretation of particular return values as dependency failures is documented_only.

## Acceptance examples

These examples are conditional static expectations, not runtime observations. Except where stated otherwise, assume stable, disjoint, sufficiently sized storage and representable arithmetic.

### Two adjacent occupied lengths

Suppose length 1 has count 1, length 2 has count 2, and every other count is zero.

Expected behavior:

- Nineteen dependency calls: sixteen counts, then three values.
- Expanded lengths: `1, 2, 2, 0`.
- Generated codes: `0, 2, 3`.
- Length 1 metadata: starting position 0, first code 0, last code 0.
- Length 2 metadata: starting position 1, first code 2, final upper endpoint 4.
- Metadata word zero: 2.
- Every absent length receives `-1` in its last-code slot; its other two per-length fields retain their prior contents.

### Gap between occupied lengths

Suppose length 2 has count 2 and length 4 has count 1.

Expected behavior:

- Expanded lengths: `2, 2, 4, 0`.
- Generated codes: `0, 1, 8`.
- Length 2 has starting position 0, first code 0, and last code 1.
- Length 4 has starting position 2, first code 8, and final upper endpoint 9.
- Metadata word zero is 4.

This demonstrates that missing intermediate lengths cause repeated shifts, not merely one shift.

### Exactly 256 entries at one length

Suppose length 8 has count 256 and every other count is zero.

Expected behavior:

- 272 dependency calls.
- Generated codes range from 0 through 255.
- Length 8’s final upper endpoint is 256.
- The terminator is observed after entry 255; the late same-length transition condition does not intervene.

### A late same-length continuation

Suppose length 9 has count 258.

After storing generated-code entry 256, the next expanded-length index is 257 and still contains length 9. The function enters transition handling despite equal lengths.

An implementation that simply accepts this as an ordinary continuation would not reproduce the supplied behavior. A graceful capacity error is also not established.

### Empty counts with zero-filled workspace

Suppose all sixteen counts are zero and expanded-length workspace entries 0 through 257 are accessible and zero.

The function makes only sixteen dependency calls, but it does not immediately finish construction. It stores generated-code values 0 through 256 at indices 0 through 256 before taking the terminator path at next index 257.

Metadata construction then writes `-1` for all sixteen last-code slots, retains metadata word zero as 1, and increments the length-1 last-code slot from `-1` to 0.

This outcome depends on the stated workspace contents; zero counts alone do not establish it.

## Uncertainties

- **documented_only — Arithmetic model:** the evidence uses four-byte signed integer operations, including addition and left shifts. Overflow, shifts producing unrepresentable signed values, and address wraparound cannot safely be assigned portable semantics from this representation alone.
- **documented_only — Input validity:** no valid count range, maximum total, or code-space constraint is supplied.
- **documented_only — Storage capacities:** pointer sources expose no allocation sizes.
- **documented_only — Dependency failures:** no failure encoding or recovery behavior is available.
- **documented_only — Aliasing and concurrency:** no exclusion guarantees are supplied. The ordinary examples require stable, nonoverlapping state.
- **Inferred intent:** canonical-code construction and decoding-table use are structural interpretations, not externally verified requirements.
- **Established limitation:** malformed or empty input is not handled by an explicit rejection path.

## Evidence

This contract derives solely from the supplied static representation of F-98f82ca8bdf3 and the supplied opaque dependency mapping.

Established observations include the sixteen count acquisitions, conditional value acquisitions, fixed memory offsets, positive-count expansion, zero terminator, code-generation transition rules, the next-index comparison against 256, and metadata writes.

No runtime tests have been performed. No dependency implementation, caller contract, allocation information, or external format specification was available.
