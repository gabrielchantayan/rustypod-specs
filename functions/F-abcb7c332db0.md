<!-- Trusted provenance: F-abcb7c332db0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-abcb7c332db0

## Purpose

F-abcb7c332db0 performs a count-limited comparison of two byte sequences through a shared byte-translation table.

The comparison advances while the remaining count is positive, the current raw byte in the first sequence is nonzero, and the translated bytes at the two current positions are equal. On stopping, it returns zero if the count has been exhausted. Otherwise, it returns a result derived from the difference between the current translated bytes.

The stopping rule is asymmetric: only the first sequence’s raw byte is checked for zero. The second sequence’s raw zero byte does not independently terminate comparison.

A plausible intent is a bounded, translation-aware string comparison. Case-insensitive comparison is only an inference: the supplied evidence does not establish the table contents, its initialization, or any character encoding.

## Interface

### Inputs

The function accepts three arguments:

1. **First byte-sequence pointer.** Points to the first current byte of the first sequence.
2. **Second byte-sequence pointer.** Points to the first current byte of the second sequence.
3. **Signed comparison count.** Limits the number of positions through which the comparison may advance.

The supplied representation uses byte pointers, a signed integer count, unsigned integer conversions, and a signed integer return value. It does not independently establish the target ABI or integer representation.

For the conventional interpretation used in the examples:

- A byte is 8 bits.
- A table index is the zero-extended input byte, ranging from 0 through 255.
- The count and result are 32-bit signed integers.
- Unsigned integer arithmetic is 32 bits.

The algorithmic conclusions do not require a specific pointer width. The integer minimum and unsigned-to-signed result conversion require the qualifications described below.

### Shared input state

The function reads a shared value that supplies the base address of a byte-translation table. Each sequence byte selects one table entry.

A full-domain table contains at least 256 readable entries. A particular invocation only needs the entries actually selected by its reads to be valid.

The supplied evidence does not establish:

- Who creates or initializes the table.
- Whether it is immutable.
- Whether the shared table base can change.
- Whether access is synchronized.
- Whether table entries implement any particular text transformation.

### Output

The function returns a signed integer.

For ordinary stable-memory executions under the conventional integer interpretation:

- Zero means either the comparison count was exhausted or the final translated bytes are equal.
- A negative result means the final first translated byte is smaller.
- A positive result means the final first translated byte is larger.

The result is not restricted to −1, 0, and 1. Its conventional numerical range is −255 through 255.

Zero does not necessarily establish equality of the raw bytes. It can result from a table collision, count exhaustion, or comparison at a terminating first-sequence zero.

### Preconditions

For every executed access:

- Each sequence pointer must identify a readable byte.
- Every selected translation-table entry must be readable.
- Pointer advancement must remain valid for the accessed memory region.
- The shared table-base value must identify appropriate readable storage.

A zero count takes a path that does not dereference either sequence or read the translation table. No readable sequence storage is needed for that path.

A negative count also bypasses comparison reads when decrementing that count by one is representable. The minimum representable signed count is a separate uncertainty.

Neither sequence needs a zero terminator if the positive count is exhausted first.

## Behavior

### Advancing comparison

The function begins at the two supplied sequence positions and the supplied count.

At each position, evaluation is gated in this order:

1. Determine whether the remaining count is strictly positive.
2. Only if it is positive, read the current first-sequence byte and check whether it is nonzero.
3. Only if that byte is nonzero, compare the table entries selected by the current bytes of both sequences.

If all three conditions hold, both sequence positions advance by exactly one byte and the remaining count decreases by exactly one. The same decision is then made at the next position.

Consequences:

- The count is a maximum number of successful advances.
- Raw byte equality is unnecessary; translated equality controls advancement.
- A raw zero in the first sequence prevents advancement even if its translated value equals the second translated value.
- A raw zero in the second sequence permits advancement when the first raw byte is nonzero and the selected table entries are equal.
- A translated zero has no special terminating role.

The count decrement during advancement is safe with respect to signed underflow: it occurs only when the current count is positive.

### Stopping and result selection

Advancement stops when any of these conditions occurs:

- The remaining count is not positive.
- The first current raw byte is zero.
- The current translated bytes differ.

The function then tests whether the remaining count minus one is negative.

For representable arithmetic:

- A remaining count of zero selects a zero return without further sequence or table reads.
- A negative remaining count also selects that zero return.
- A positive remaining count selects a fresh translation of the two current bytes and computes their difference.

For a positive initial count, count exhaustion leaves the remaining count at zero. The function does not inspect the next bytes or look up their translations before returning zero.

Stopping at a first-sequence raw zero does not itself return zero. If the remaining count is positive, the function translates that zero byte and compares it numerically with the translation selected by the second current byte.

### Final arithmetic

The final translated values are read as bytes and converted to unsigned integers before subtraction.

For translated byte values \(a\) and \(b\):

- If \(a \ge b\), the unsigned subtraction directly represents \(a-b\).
- If \(a < b\), the unsigned subtraction wraps according to the unsigned integer width.
- That unsigned result is subsequently assigned to the signed return type.

Under the conventional 32-bit, two’s-complement conversion interpretation, the returned value is the mathematical difference \(a-b\), within −255 through 255.

The supplied representation alone does not establish the language or machine rule for converting a wrapped unsigned result to signed. Exact negative-result portability is therefore documented_only rather than independently proven.

### Read ordering and repeated reads

The short-circuit gates establish useful ordering constraints:

- A nonpositive count prevents the loop’s sequence and table reads.
- The first-sequence zero check precedes the loop’s translated comparison.
- A failed first-sequence zero check prevents that translated comparison.
- The final count decision precedes final-result reads.

The evidence does not establish a total evaluation order between the two operands of a translated equality or subtraction expression. In particular, it does not prove that the first table lookup is physically performed before the second.

Bytes used in the stopping comparison are not established as cached for the final subtraction. The final result expression reads the current sequence bytes and table entries again. Likewise, the first-sequence byte used for the raw zero test is separately referenced when selecting its table entry.

These distinctions matter if readable memory can change between accesses. With stable memory, they do not change the comparison result. With changing memory, the result need not reflect the values that caused the loop to stop. The function provides no visible snapshot or synchronization mechanism.

## Side effects

No explicit writes to either input sequence, the translation table, or other shared storage are present.

The function changes only its local traversal state:

- Its current first-sequence position.
- Its current second-sequence position.
- Its remaining comparison count.

Those changes do not update caller-held pointer or count objects.

No allocation, release, ownership transfer, external function call, logging, or error-reporting operation is visible.

Input regions may overlap or coincide because no writes are shown. Such overlap does not remove the requirement that all executed reads be valid.

Invalid addresses, insufficient readable storage, or an invalid table base have no explicit recovery path. Any resulting fault or language-level invalid behavior is outside the defined successful-input behavior; the function does not translate it into a distinguished return value.

## Dependencies

No callable dependency is visible.

The shared byte-translation table is a data dependency, not a function dependency. No separate opaque function ID is supplied for it.

The function’s behavior depends on:

- The bytes read from both sequences.
- The table-base value and selected table bytes.
- Pointer arithmetic semantics.
- Signed count arithmetic.
- Unsigned subtraction and conversion to the signed return type.

No preservation, ownership, or release behavior is attributed to an unknown callee because no callee is present.

## Acceptance examples

These are conditional acceptance examples derived from the supplied static evidence, not executed tests. Unless stated otherwise, memory is valid and stable, bytes are 8 bits, and the conventional integer interpretation applies.

### Zero count avoids input reads

With count zero, the result is zero. Neither input sequence nor the translation table is read, even if the first bytes would otherwise differ.

### Ordinary negative count avoids input reads

With count −1, the result is zero without sequence or table reads. This example does not generalize to the minimum representable signed count.

### Count exhaustion hides later differences

With an identity translation table, first sequence bytes 65, 66 and second sequence bytes 65, 67:

- Count one returns zero.
- Count two returns −1.

The first invocation does not inspect the second pair of bytes.

### A mismatch returns its magnitude

With an identity translation table, positive count, and current bytes 200 and 3, the result is 197.

Reversing the bytes gives −197 under the stated signed-conversion interpretation.

### Different raw bytes can compare equal

Suppose table entries 65 and 97 both contain 97. With first current byte 65, second current byte 97, and count one, the function advances once and returns zero.

This does not establish case folding beyond those conditional table contents.

### First-sequence zero still participates in the final result

With an identity translation table, positive count, first current byte zero, and second current byte 7, the result is −7.

The loop does not advance, but final-result reads occur.

### Zero in the second sequence need not stop advancement

Suppose table entries 5 and zero both contain 9. With first current byte 5, second current byte zero, and count one, the translated comparison succeeds. The function advances once and returns zero because the count is exhausted.

### Equal translated terminator values return zero without advancing

Suppose the first current raw byte is zero and both current bytes select table value 12. With positive count, the function returns zero without advancing past the first current position.

### Changing table contents invalidate stable-read expectations

If a table entry changes after a mismatch is observed but before the final subtraction, the final result can reflect the later value rather than the mismatching value. No single numerical result is required by this example without a defined external memory-change model.

## Uncertainties

The following are documented_only:

- **Exact ABI widths.** The supplied representation does not independently prove integer or pointer widths.
- **Minimum signed count.** The post-loop decision subtracts one from the count. For the minimum signed value, that operation is not representable in the same signed type. Under a 32-bit model, the exceptional input is −2,147,483,648. The evidence does not settle whether the actual execution wraps, traps, or follows another rule. It is unsafe to promise zero for every negative input.
- **Negative-result conversion.** The conventional mathematical difference depends on the signed conversion of a wrapped unsigned subtraction.
- **Table meaning.** No encoding, locale, case mapping, or table invariant is established.
- **Concurrent changes.** No synchronization or stable-memory guarantee is shown.
- **Physical access trace.** Short-circuit ordering is visible, but an exact machine-load count or total order requires evidence beyond this representation.
- **Invalid-input outcomes.** No specific fault, exception, or diagnostic behavior is established.

The inferred intent is bounded comparison using a translation table. The established behavior is the asymmetric, count-gated sequence of reads and advances described above; the inference does not override it.

## Evidence

The supplied static representation establishes:

- A signed-positive count gate before loop input reads.
- A raw-zero test applied only to the first sequence.
- Table indices formed from input bytes converted to unsigned integers.
- Translated equality as the condition for advancing both positions.
- One-byte advancement and a one-unit count decrement per successful iteration.
- A post-loop count-minus-one decision.
- A zero-return branch without further input reads.
- A final result derived from fresh translated-byte reads and unsigned subtraction.
- No explicit memory writes or external calls.

No runtime tests have been performed. The acceptance examples are static deductions under their stated conditions, not observations from execution.
