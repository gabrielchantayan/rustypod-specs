<!-- Trusted provenance: F-14d5f1ca2fa8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-14d5f1ca2fa8

## Purpose

F-14d5f1ca2fa8 selects or visits an entry through an indirect callback, asks F-eda6f7b141ff whether processing should continue, and conditionally updates two one-byte fields in a caller-supplied context.

For an ordinary entry, the candidate value is the first byte of a byte sequence obtained through another indirect callback. A nonzero candidate may lower the context’s byte at offset 8 and raise its byte at offset 9. An entry with a particular flag set instead causes unconditional writes of 0 and 127 to those respective fields.

The interpretation of these fields as lower and upper bounds is inferred intent, not an established API meaning. In particular, the flagged-entry path overwrites both fields rather than comparing them with their existing values.

## Interface

### Inputs

The recovered interface has three inputs:

1. **Context address.** An address used to locate a secondary object and two mutable byte fields.
2. **Initial entry address.** A pointer initially placed in a writable local slot. The first indirect callback receives the address of that slot and can replace its contents.
3. **Position.** A signed 32-bit value used both to derive a callback selector and to initialize a neighboring local slot.

There is no established validation of these inputs.

### Output

The function has no specified return value. Its observable results are:

- Calls to two indirect callbacks and F-eda6f7b141ff, subject to the branches described below.
- Possible writes to context offsets 8 and 9.
- Any additional effects performed by the callees.

An early return is not accompanied by a status code or diagnostic.

### Visible memory layouts

Offsets are byte offsets unless stated otherwise.

| Object | Offset | Established use |
|---|---:|---|
| Context | 4 | Pointer to the secondary object |
| Context | 8 | Mutable unsigned byte, used as the lower comparison field |
| Context | 9 | Mutable unsigned byte, used as the upper comparison field |
| Secondary object | 0 | Pointer to a table of callable entries |
| Secondary object | 4 | Signed 32-bit value used to recognize the final position |
| Secondary object’s callable table | 32 | First indirect callback target |
| Selected entry | 0 | Pointer to a table of callable entries |
| Selected entry | 4 | Byte containing the tested flag bit |
| Selected entry’s callable table | 32 | Ordinary-entry indirect callback target |

These layouts reflect the apparent 32-bit pointer and integer model of the supplied static representation. They do not establish complete object sizes, alignment requirements, other fields, or object types.

The flag test examines bit value 8 in the byte at selected-entry offset 4. It does not inspect the entire 32-bit field containing that byte.

### Preconditions

For defined, successful execution, each memory access and indirect call actually reached must be valid:

- The context must permit reading its pointer field at offset 4.
- The secondary object and its callable table must support the required reads and callback.
- The selected entry, after callback processing, must be acceptable to F-eda6f7b141ff.
- If that dependency returns nonzero, the selected entry must support the flag read.
- On the ordinary-entry path, its callable table and returned byte pointer must be valid when used.
- Context bytes 8 and 9 must be readable or writable as required by the selected branch.

The function provides no visible null checks for the context, secondary object, callable tables, callback targets, or selected entry. The null check described below applies only to the byte pointer returned by the ordinary-entry callback.

## Behavior

### 1. Derive a selector

The function first reads the secondary-object pointer from context offset 4.

It then derives a signed 32-bit selector from the supplied position:

- Position −1 produces selector 0. This case takes precedence and does not require reading the secondary object’s value at offset 4 for this decision.
- Otherwise, if the position equals that value minus 1, the selector is 2,147,483,647.
- Otherwise, the selector is the position plus 1.

The value at secondary-object offset 4 resembles a count, and the maximum signed selector resembles an end sentinel. Those meanings are inferred, not established.

The subtraction and addition are expressed as signed 32-bit operations in the supplied representation. Behavior at signed-overflow boundaries is not established. No saturation, range check, wider arithmetic, or portable wraparound guarantee should be added to this contract.

### 2. Invoke the selection callback

Two local 32-bit slots are initialized:

- A pointer-bearing slot receives the initial entry address.
- A neighboring integer slot receives the original position, not the derived selector.

The callback is obtained from offset 32 of the secondary object’s callable table. The visible call supplies these argument values, in this logical order:

1. The secondary-object address.
2. The derived selector.
3. The address of the local pointer-bearing slot.
4. The callback address obtained from the same callable-table offset.
5. The original context address.

The recovered call shape does not establish a higher-level callback prototype or calling convention. In particular, passing the callback address as an argument must not be silently omitted or reinterpreted as an ordinary user-data field.

The callback can replace the selected entry by writing through the supplied slot address. Although a position slot is shown adjacent to it, this alone does not establish a public two-field structure, a length, or permission to access arbitrary neighboring stack memory.

No callback return value is visibly consumed. Its effects on the slot, context, secondary object, entries, or other memory are not restricted by the available evidence.

### 3. Apply the dependency gate

After the selection callback returns, the pointer-bearing slot supplies the entry address passed to F-eda6f7b141ff.

- A zero result causes an immediate return.
- Any nonzero result continues processing.

The dependency’s purpose is unknown. It must not be specified as merely a predicate that preserves all memory. The gate establishes only the branch taken on its result; it does not establish entry validity, ownership, or absence of side effects.

Because the local slot’s address was exposed to the first callback, the evidence also does not justify an unconditional assumption that its contents can never change again through retained aliases.

### 4. Inspect the selected-entry flag

After a nonzero gate result, the function reads the flag byte at selected-entry offset 4 and tests bit value 8.

#### Flag set

If the bit is set:

1. Write 0 to context offset 8.
2. Write 127 to context offset 9.
3. Return.

The ordinary-entry callback is not invoked. Neither context byte is compared with its old value.

In particular, an existing upper byte greater than 127 is reduced to 127.

#### Flag clear

If the bit is clear, processing continues through the ordinary-entry callback.

### 5. Obtain and examine the ordinary-entry byte

The function invokes the callback at offset 32 of the selected entry’s callable table. The supplied representation shows no explicit arguments for this invocation. That does not establish whether the actual ABI supplies an implicit receiver or other hidden state.

The result is represented as 64 bits:

- Its low 32 bits are interpreted as a byte pointer.
- Its high 32 bits are initially extracted as an unsigned value.

If the pointer is non-null, the first byte at that pointer is read as an unsigned value, replacing the extracted high-half value.

The function returns without its own context-field writes if either:

- The pointer is null; or
- The pointer is non-null and the first byte read is zero.

Thus the high half of the callback result has no effect on the eventual context updates: a null pointer always returns, and a non-null pointer replaces that half with a byte read. This is a behavioral consequence of the visible branches, not evidence that the actual callback formally returns a pointer-and-value pair.

### 6. Conditionally update the ordinary-entry fields

For a non-null pointer whose first byte read was nonzero:

1. Read the unsigned byte at context offset 8.
2. If the first byte value is smaller, write that value to context offset 8.
3. Read the byte through the returned pointer again.
4. Read the unsigned byte at context offset 9.
5. If the newly read byte is less than or equal to the upper field, return without writing that field.
6. Otherwise, write the newly read byte to context offset 9.

The two byte reads through the returned pointer are distinct. The second occurs after any lower-field write.

Consequently, the same value is not guaranteed to govern both updates. Aliasing, callback-established shared memory, or external mutation may make the second byte differ from the first. The function does not reapply the nonzero test to the second read.

For stable, non-aliasing memory, the ordinary path behaves like a lower-minimum and upper-maximum update using an unsigned nonzero byte. That simplified description is conditional, not a replacement for the access order above.

## Side effects

### Direct writes

The function directly writes only:

- Its local pointer-bearing and neighboring position slots during initialization.
- Context byte 8 when the flagged path is taken or an ordinary candidate is smaller.
- Context byte 9 when the flagged path is taken or the second ordinary candidate is larger.

On the flagged path, the lower-byte write precedes the upper-byte write.

On the ordinary path, a lower-byte write, if performed, precedes the second candidate-byte read and any upper-byte write.

### Indirect effects

Both indirect callbacks and F-eda6f7b141ff may have additional effects. No preservation of globals, context fields, object fields, or pointed-to storage is established.

An early return means no further direct operations by this function. It does not imply that earlier callees left the context unchanged.

### Ownership and lifetime

No direct allocation, deallocation, retention operation, or reference-count update is visible.

This does not establish ownership rules for:

- The initial or replacement entry.
- The secondary object.
- The returned byte sequence.
- Any memory touched by a dependency.

The returned byte pointer must remain usable through the final byte read when that path is reached. The function does not visibly release it.

The address passed to the selection callback refers to local storage. Whether the callback is permitted to retain it is undocumented; the function supplies no visible lifetime-management mechanism for such retention.

### Failure behavior

There is no visible recovery for invalid memory, invalid callback targets, callee failures, or arithmetic outside the established range. No error translation, cleanup handler, or fallback entry is shown.

## Dependencies

### F-eda6f7b141ff

Established interaction:

- Receives the entry pointer obtained after the selection callback.
- Returns a value compared against zero.
- Zero stops processing; nonzero permits the flag read and subsequent branch.

Unknowns include its actual purpose, accepted entry states, side effects, ownership behavior, and failure mechanisms.

### Indirect selection dependency

No opaque function ID is supplied for this dynamically selected target.

It is resolved through the secondary object’s callable-table offset 32. Its visible arguments and writable local-slot access are described above. Its return value is ignored.

### Indirect ordinary-entry dependency

No opaque function ID is supplied for this dynamically selected target.

It is resolved through the selected entry’s callable-table offset 32. Its result is consumed using the low/high 32-bit interpretation described above. Its actual ABI and ownership contract remain unknown.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless stated otherwise, assume valid addresses, arithmetic within the established signed range, stable non-aliasing storage, and callees that make no changes except those specified.

1. **Initial-position selector.** With position −1, the selection callback receives selector 0 and the neighboring local position slot contains −1. The secondary object’s count-like field does not affect this branch.

2. **Final-position selector.** With the secondary field equal to 5 and position equal to 4, the selector is 2,147,483,647. The neighboring position slot remains 4.

3. **Ordinary selector.** With the secondary field equal to 5 and position equal to 2, the selector is 3.

4. **Replaced entry.** If the selection callback replaces the local entry pointer with another address, that replacement is supplied to F-eda6f7b141ff. The initial entry is not preferentially retained.

5. **Gate rejection.** If F-eda6f7b141ff returns zero, no flag read, ordinary-entry callback, or direct context-byte write follows. Earlier dependency effects remain possible.

6. **Flagged overwrite.** Given a nonzero gate result, flag bit 8 set, and context bytes initially 20 and 240, the function writes 0 and then 127. It does not retain 240 as a larger upper bound.

7. **Null byte pointer.** With the flag clear and an ordinary-entry result whose low 32 bits are zero, the function performs no direct context-byte writes, regardless of the high 32 bits.

8. **Zero first byte.** With a valid returned pointer whose first byte is zero, the function returns before reading either context comparison byte.

9. **Lower-only ordinary update.** With lower 80, upper 120, and stable candidate 50, the final bytes are 50 and 120.

10. **Upper-only ordinary update.** With lower 20, upper 100, and stable candidate 200, the final bytes are 20 and 200. The candidate is compared as unsigned, not as a negative signed byte.

11. **Equal candidate.** With both fields equal to 75 and stable candidate 75, neither field is written.

12. **Distinct candidate reads.** If the first candidate read yields 50 and the second yields 200, with lower 80 and upper 100, the direct updates produce lower 50 and upper 200. The function does not cache the first value for both comparisons.

## Uncertainties

The following are **documented_only**, not implementation requirements invented to fill gaps:

- Exact source-language types, ABI, implicit callback arguments, and complete object layouts.
- The semantic meaning of the position, count-like field, selector sentinel, flag, and context bytes.
- Whether invalid or out-of-range positions are permitted.
- Signed-overflow behavior for selector arithmetic.
- Callback and dependency mutation, synchronization, ownership, and lifetime rules.
- Whether the adjacent local slots form an intentional callback-visible record.
- Whether the 64-bit ordinary-entry result reflects a real declared return type or an artifact of the recovered representation.
- Whether aliases or concurrent mutation are valid inputs under the original program’s contract.

No runtime tests have been performed.

## Evidence

**Established static evidence:** The supplied representation shows the context and entry offsets, selector branches, writable entry slot passed to the first callback, zero-result dependency gate, flag-bit branch, 64-bit result decomposition, null and zero-byte exits, unsigned byte comparisons, and ordered conditional writes described above.

**Inferred intent:** Selection or traversal of an entry, use of a terminal selector, and accumulation of lower and upper byte bounds. These interpretations explain the shape of the operations but do not strengthen the established guarantees.

**Unknown:** Dependency implementations, valid input domain, complete calling conventions, ownership contracts, and runtime behavior outside the visible operations. No external implementation evidence or execution results were available.
