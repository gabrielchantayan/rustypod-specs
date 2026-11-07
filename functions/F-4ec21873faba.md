<!-- Trusted provenance: F-4ec21873faba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4ec21873faba

## Purpose

F-4ec21873faba coordinates selection, acquisition, classification, and downstream processing of an indexed byte sequence.

The established behavior is to:

- Normalize the distinguished index value −1 to zero.
- Reject an index at or beyond a supplied signed bound with status 1.
- Select a 32-bit table entry and pass it to an input-associated dependency.
- Obtain a signed 32-bit length or sentinel.
- Request storage and populate it through dependencies.
- Select one of two opaque dispatch addresses using a length-gated, four-byte signature check.
- Invoke a downstream dependency with the acquired buffer, normalized index, selected address, and a forwarded 32-bit value.

The apparent intent is to distinguish an `OTTO`-prefixed representation from a default representation associated with the name “truetype.” That interpretation is inferred. The established contract is the byte comparison and address selection, not a complete file-format validation or a guarantee that either address names a particular string.

## Interface

### Inputs

The function has six inputs:

| Position | Contract role | Observed representation |
|---|---|---|
| 1 | Context address | Pointer to storage from which one 32-bit word is read |
| 2 | Input-associated address | Byte-pointer-shaped value passed to three dependencies and initially used as the upper return word |
| 3 | Entry-table base | 32-bit integer-shaped address used to locate 32-bit entries |
| 4 | Entry bound | Signed 32-bit integer |
| 5 | Requested index | Signed 32-bit integer; exactly −1 is normalized to zero |
| 6 | Forwarded value | Opaque 32-bit word passed unchanged to the final dependency |

The first input has two distinct uses: an initial 32-bit read supplies an argument to the storage-request dependency, while the original context address is later passed to the final dependency. These uses must not be conflated. A dependency that changes the context after the initial read does not change the already captured word.

The second input is not directly dereferenced by this function. Its required memory layout, if any, is determined by the dependencies receiving it.

The third input is used as an address rather than merely as a numeric identifier. Each selected entry occupies four bytes. This function directly reads one entry, without establishing that the table contains the supplied number of entries.

### Result

The observed result is a 64-bit value composed of two 32-bit fields:

- Bits 31–0: status.
- Bits 63–32: an address-shaped value.

This describes bit positions, not the byte order of a serialized result.

Before dispatch classification, the upper field is the original second input. Once classification has completed, it is one of two selected opaque addresses. The lower field is either the locally produced status 1 or a status returned or written by a dependency.

No standalone Boolean success result is established. Zero statuses permit progress at the intermediate stages, but the meaning of the final dependency’s status remains its own contract.

### Preconditions and limits

For the initial read to be valid, the first input must reference at least one readable 32-bit word. This requirement applies even when the bound check would immediately terminate processing.

If execution reaches entry selection, the effective table-entry address must support a readable 32-bit access. There is no explicit null, alignment, accessibility, or table-extent validation.

Any returned buffer must support the operations performed by the filling dependency. If classification reaches the signature check, the buffer must also support the required direct byte reads. The function does not independently establish these properties.

## Behavior

### 1. Normalize the index and capture context state

Only an index equal to −1 is changed, becoming zero. Other negative indices remain unchanged.

The function then reads the first 32-bit word at the context address. This read precedes the bound decision and all dependency calls.

The original second input is retained as the upper return field until classification selects a replacement.

### 2. Apply the signed bound check

Processing stops when the signed bound is less than or equal to the normalized signed index.

This path:

- Returns status 1.
- Returns the original second input in the upper 32-bit field.
- Makes no dependency calls.
- Does not read a table entry.

There is no separate test that the normalized index is nonnegative. Consequently, the signed bound comparison alone does not establish a valid array index.

### 3. Read and select an entry

When the bound is greater than the normalized index, the function reads the 32-bit entry at the table base plus four times that index.

It calls F-e40a3ceddab2 with:

1. The original second input.
2. The selected entry value.

The dependency’s 32-bit return value becomes the current status.

A nonzero result terminates processing immediately. That status is returned with the original second input in the upper field. No length query, storage request, fill operation, or final dispatch follows.

The four-byte stride is established. Exact overflow behavior for the address expression is not established by the supplied representation. In particular, the contract does not promise checked multiplication, checked address addition, or rejection of negative offsets.

### 4. Obtain a length or sentinel

After a zero result from F-e40a3ceddab2, the function calls F-10d2ce969adf with:

1. The original second input.
2. The address of writable local 32-bit status storage.

At entry to this call, that status storage contains zero. The function receives a signed 32-bit return value, used here as the length, and then examines the status storage.

The status check has precedence over the length sentinel:

- If the written status is nonzero, return it immediately with the original second input as the upper field.
- Otherwise, if the returned length is exactly −1, return status 1 with the original second input as the upper field.
- Otherwise, proceed with that length unchanged.

There is no general rejection of negative lengths. Values below −1 continue to the storage-request dependency. A zero length also continues.

### 5. Request storage

The function calls F-3ce007b692cc with:

1. The 32-bit context word captured before the bound check.
2. The length returned by F-10d2ce969adf.
3. The address of the same local status storage.

The status storage contains zero immediately before this call, because the preceding stage could continue only with a zero status.

The dependency returns a byte-pointer-shaped value and may write a status.

If the status is nonzero, the function returns it immediately. The upper return field remains the original second input. The returned buffer value is not passed onward or directly inspected on this path.

If the status is zero, the function proceeds without an explicit null-pointer check.

Calling this a storage request describes its role in this sequence, not an established allocation or ownership guarantee.

### 6. Populate the buffer

The function calls F-0c21b5cad025 with:

1. The original second input.
2. The buffer address returned by F-3ce007b692cc.

No length is supplied as a separate argument to this call.

Its 32-bit return value becomes the status. A nonzero value terminates processing with that status and the original second input in the upper return field.

Only after this call returns zero does the function perform its direct buffer inspection. Thus classification observes the buffer after any changes made by this dependency, not necessarily the storage request’s initial contents.

### 7. Select the dispatch address

The signed length gates the signature check:

- For any length less than five, select the default address without directly reading buffer bytes.
- For a length of at least five, compare the first four buffer bytes, in order, against `O`, `T`, `T`, and `O`.
- Select the alternate address only if all four comparisons match.
- Otherwise select the default address.

The comparisons short-circuit:

1. Read offset zero.
2. Read offset one only if offset zero matched.
3. Read offset two only if the first two matched.
4. Read offset three only if the first three matched.

The fifth byte is not read for classification. Nevertheless, a reported length of four does not qualify for the alternate address, even if the available first four bytes spell `OTTO`.

The default address is associated with “truetype” in the supplied representation. The alternate address is represented as a label address. Neither fact establishes a complete readable string, a required terminator, or permission to dereference the selected address as text.

### 8. Invoke downstream processing

The function calls F-08ec95defbd3 with these arguments, in order:

1. The original context address.
2. The buffer address.
3. The length.
4. The normalized index.
5. The selected dispatch address.
6. The unchanged sixth input.

Its 32-bit return value becomes the lower return field. The selected dispatch address becomes the upper return field.

The function does not branch on the final status, retry processing, perform another classification, or explicitly release the buffer afterward.

## Side effects

### Directly established memory operations

On an ordinary, non-faulting execution, direct reads occur in this order:

1. One 32-bit context-word read.
2. If the bound permits continuation, one 32-bit table-entry read.
3. After successful selection, length acquisition, storage request, and buffer filling, zero to four one-byte buffer reads for classification.

The direct buffer reads are conditional on both a signed length of at least five and the preceding comparison results.

No direct caller-memory write is visible in this function. Its local status storage is passed by address to F-10d2ce969adf and F-3ce007b692cc. Those dependencies can affect control flow by writing that storage.

### Dependency effects

Dependency calls occur in this order, stopping at the first terminating condition:

1. F-e40a3ceddab2.
2. F-10d2ce969adf.
3. F-3ce007b692cc.
4. F-0c21b5cad025.
5. F-08ec95defbd3.

The function supplies no rollback path for earlier dependency effects when a later stage fails.

Nothing in the available evidence establishes that dependencies preserve globals, leave context state unchanged, preserve input-associated state, or avoid external effects. In particular, the captured context word is a snapshot; the context address later passed downstream may designate state changed by intervening calls.

### Ownership and lifetime

There is no explicit release operation in this function.

The evidence does not establish:

- Whether the storage-request result is newly allocated, borrowed, shared, or cached.
- Whether the filling dependency assumes ownership.
- Whether the final dependency assumes ownership or releases storage.
- Whether a buffer requires release after a fill failure.
- Whether any dependency retains supplied addresses after returning.

Absence of a visible release is not sufficient evidence of either a leak or an ownership transfer.

## Dependencies

| Opaque ID | Established interaction | Unspecified behavior |
|---|---|---|
| F-e40a3ceddab2 | Receives the input-associated address and selected 32-bit entry; returns a status that must be zero to continue | Entry interpretation, validation, state changes, and external effects |
| F-10d2ce969adf | Receives the input-associated address and writable 32-bit status storage; returns a signed length or sentinel | How the length is determined and whether other state changes |
| F-3ce007b692cc | Receives the captured context word, length, and writable status storage; returns a buffer-shaped address | Allocation semantics, capacity, initialization, ownership, and failure details |
| F-0c21b5cad025 | Receives the input-associated address and buffer; returns a status that must be zero to continue | How much it writes, where its size information comes from, and ownership effects |
| F-08ec95defbd3 | Receives context, buffer, length, normalized index, selected address, and forwarded value; returns the final status | Processing semantics, interpretation of the selected address, retention, and cleanup |

These descriptions specify only interactions visible at this boundary.

## Acceptance examples

The following are conditional acceptance cases derived from static evidence. They are not reports of executed tests. Each assumes valid memory for all reached accesses and dependencies that return normally as stated.

1. **Immediate bound termination.** With bound zero and index −1, the index becomes zero. The context word is still read. No table entry or dependency is accessed. The result has status 1 and the original second input as its upper field.

2. **Negative index other than −1.** With bound zero and index −2, the signed bound check permits continuation. The selected entry address is eight bytes before the table base, assuming the address arithmetic is representable. The function does not convert this index to zero or reject it independently.

3. **Entry-selection failure.** If F-e40a3ceddab2 returns a nonzero 32-bit value, that exact bit pattern is the lower return field. No later dependency is called.

4. **Length-status precedence.** If F-10d2ce969adf returns −1 but writes status 7, the result status is 7, not 1. The storage request does not occur.

5. **Length sentinel.** If F-10d2ce969adf returns −1 and leaves status zero, the result status is 1. The upper field remains the original second input.

6. **Other negative length.** If the length is −2 and the status is zero, F-3ce007b692cc receives −2. If subsequent stages report zero statuses, classification takes the default-address path without direct buffer reads, and the final dependency receives length −2.

7. **Storage-request failure.** If F-3ce007b692cc writes a nonzero status, that status is returned with the original second input as the upper field. The fill and final dependencies are not called.

8. **Fill failure.** If F-0c21b5cad025 returns a nonzero status, no signature byte is directly read and no final dispatch occurs. No explicit release is performed by this function.

9. **Four-byte boundary.** With length four, successful prior stages, and buffer bytes beginning `OTTO`, the default address is selected. The function does not directly inspect those bytes for classification.

10. **Alternate classification.** With length five or greater and the first four bytes equal to `OTTO`, the alternate address is supplied to F-08ec95defbd3 and returned in the upper field. The fifth byte does not affect this selection.

11. **Early mismatch.** With length at least five and a first byte other than `O`, classification directly reads only the first byte. The default address is selected.

12. **Final failure after classification.** If F-08ec95defbd3 returns a nonzero status, that status is returned together with the selected address—not the original second input. There is no explicit recovery call.

## Uncertainties

The following remain **documented_only**:

- Exact calling convention and whether consumers interpret the 64-bit result as a scalar or two machine-word fields.
- Endianness of stored 32-bit values and results.
- Exact overflow and fault behavior for entry-address arithmetic.
- Required alignment and valid address ranges.
- Full layouts of context and input-associated objects.
- Meanings of dependency-specific statuses.
- Dependency behavior when given zero or negative lengths, null buffers, or invalid entries.
- Ownership, retention, cleanup, and global-state behavior of all dependencies.
- Actual memory contents and intended semantic types of the two selected addresses.
- Whether the five-byte threshold is intentional format policy or incidental behavior.

Inferred intent is limited to a format-like classification followed by downstream processing. No stronger format, safety, ownership, or error-recovery guarantee is justified.

## Evidence

This contract is based exclusively on static inspection of the supplied function representation and the supplied opaque dependency mapping.

Established evidence includes the six-input interface, 32-bit context and table reads, signed comparisons, exact −1 normalization and sentinel handling, dependency order, status precedence, conditional byte-read order, absence of an explicit release call, and the 64-bit return-field composition.

Names and apparent type hints are not treated as authoritative behavioral documentation. Where a name suggests meaning beyond the observed operations, that meaning is identified as inference or left documented_only.

No runtime tests have been performed.
