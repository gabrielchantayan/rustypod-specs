<!-- Trusted provenance: F-f5cd16faac60; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f5cd16faac60

## Purpose

F-f5cd16faac60 performs a fixed sequence of four preparation operations, invokes a common dependency after each of the first three, and then conditionally processes a zero-terminated sequence through an object obtained from its third input.

For every item decoded during the final processing phase, it queries whether a corresponding position is already represented. If the query returns zero, it increments an object field and sets a bit in a word array.

The established behavior is this call sequence and conditional state update. The preparation labels suggest display-related configuration, and the final updates resemble membership registration. Those interpretations are inferred intent, not established requirements. In particular, neither the item encoding nor the meanings of the object fields are supplied.

## Interface

The supplied representation exposes four inputs and no explicit return value.

| Input | Exposed width | Established use |
|---|---:|---|
| First input | 32 bits | Not directly read or forwarded by the visible body. |
| Second input | 32 bits | Not directly read or forwarded by the visible body. |
| Third input | Unsigned 32 bits | Forwarded unchanged to three calls of F-6ec4235d8ea3; later has its least significant bit cleared before a null check and indirect dispatch. |
| Fourth input | 32 bits | Forwarded unchanged to the same three calls of F-6ec4235d8ea3. |

The third input has two distinct observable forms:

- Its original bit pattern is used in the three direct dependency calls.
- Its value masked with hexadecimal FFFFFFFE is used for the final dispatch.

The low bit is therefore ignored for the final object lookup, but is not removed from the earlier forwarded input.

The implementation model is 32-bit: object links, dispatched-function slots, counters, and bit-array words are accessed through 32-bit quantities. A nonzero masked third input is interpreted as an address. Its first 32-bit word supplies a dispatch-table address, and the entry at byte offset 40 in that table supplies the indirect target.

No explicit parameters to that indirect target are established by the supplied representation. A receiver argument, calling convention, and complete callable signature remain undocumented.

The function does not directly report success, failure, the number of decoded items, or the number of newly set bits. Dependency failures and invalid-memory behavior are not translated into a documented error result.

Required conditions for normal completion include:

- Every memory access actually reached must be valid.
- Every dependency invoked must satisfy the calling and data-layout requirements of that invocation.
- If final sequence processing occurs, its cursor must remain valid and eventually reach a zero byte.
- Every computed bit-array location must be addressable.
- The object field increment and address calculations must be supported for the values encountered.

These are execution preconditions, not checks performed by this function.

## Behavior

### Initial pointer capture and preparation sequence

Before the first dependency call, the function reads one global pointer-valued location and retains the resulting pointer.

The supplied body does not establish whether this pointer refers to immutable text, shared writable storage, a preparation result buffer, or another representation. The important distinction is that the pointer value is captured once: the global pointer-valued location is not visibly reloaded between the subsequent preparation operations.

The function then performs the following calls in order:

| Step | Dependency | Established arguments |
|---|---|---|
| 1 | F-59c1430f85c2 | Address of local scratch storage, the LCD preparation label, numeric value 12, numeric value 0 |
| 2 | F-6ec4235d8ea3 | Original third input, fourth input, captured pointer |
| 3 | F-59c1430f85c2 | Same scratch-storage address, the TV preparation label, numeric value 20, numeric value 0 |
| 4 | F-6ec4235d8ea3 | Original third input, fourth input, captured pointer |
| 5 | F-59c1430f85c2 | Same scratch-storage address, the LCD preparation label, numeric value 12, numeric value 2 |
| 6 | F-6ec4235d8ea3 | Original third input, fourth input, captured pointer |
| 7 | F-59c1430f85c2 | Same scratch-storage address, the TV preparation label, numeric value 20 |

The LCD label denotes the supplied MonoHope label ending in LCD; the TV label denotes its counterpart ending in TV.

The numeric values 12 and 20 are established arguments. Their meaning is not established: they must not be assumed to be string lengths, buffer capacities, display widths, or element counts.

The last preparation call exposes only three arguments. Unlike the preceding preparation calls, it does not establish a fourth numeric argument. It is not valid to specify that the last call uses mode 0, mode 2, or any other particular mode.

The preparation calls receive the same scratch-storage address. The three calls of F-6ec4235d8ea3 receive the same captured pointer value, but not necessarily unchanged pointed-to contents. Dependencies may mutate accessible storage.

The function itself does not visibly assign the captured pointer from any preparation-call return value. Any relationship between preparation and the storage subsequently inspected is therefore dependent on behavior or aliasing not supplied here.

### Conditional object acquisition

After the fourth preparation call returns, the function initializes a local sequence cursor from the captured pointer.

It then clears the least significant bit of the third input and tests the result against zero.

- If the masked value is zero, the function performs no final indirect dispatch, decoding, membership query, counter increment, or bit-array update.
- If the masked value is nonzero, the function uses it as the address of an object containing a dispatch-table pointer in its first word.

The function loads an indirect callable from byte offset 40 of that table and invokes it. The returned 32-bit value is retained as the address of the state object used by the final phase.

If this returned value is zero, final processing is skipped. There is no alternate object lookup or fallback.

The object lookup occurs after all seven direct calls listed above. Those calls can therefore affect the memory later used by the dispatch. The contract does not assume that the dispatch-table contents or other globals remain unchanged across dependencies.

### Sequence processing and membership queries

When the indirect call returns a nonzero object address, processing proceeds while the byte at the current sequence cursor is nonzero.

Each iteration has this externally relevant order:

1. Read the byte at the current cursor to decide whether processing continues.
2. Call F-d3922c7fe0ec with the address of the cursor storage.
3. Treat its result as an unsigned 32-bit item value.
4. Derive a word index by shifting the item value right by five bits.
5. Derive a bit position from the item value’s low five bits.
6. Call F-88658862e573 with the retained state-object address, word index, and bit position.
7. If that call returns zero, increment the object field and set the corresponding array bit.
8. Test the byte at the current cursor again for the next iteration.

F-d3922c7fe0ec receives access to the cursor itself, so it can update the pointer used by the next loop test. The caller does not independently advance the cursor.

There is no visible check that decoding advanced the cursor, returned a valid character, remained within a buffer, or produced a bounded item value. A dependency that leaves the cursor on a nonzero byte can cause repeated processing or nontermination.

A zero byte terminates processing before a decoder call. The function does not pass that terminator to the decoder.

### Conditional state updates

The nonzero state-object address returned by the indirect call is used with the following visible layout:

| Byte offset | Accessed contents |
|---:|---|
| 4 | 32-bit field incremented when the membership query returns zero |
| 8 | Pointer to an array of 32-bit words |

For an unsigned decoded value \(v\):

- The word index is \(\lfloor v / 32 \rfloor\).
- The bit position is \(v \bmod 32\).
- The selected array word is located at the array base plus four times the word index.
- The selected mask has one bit set at that position.

F-88658862e573 is invoked before these conditional updates. A nonzero result suppresses both explicit writes. Any nonzero value has that effect; the branch does not require a particular true value.

If the query returns zero, the function first increments the 32-bit field at offset 4 by one. It then updates the selected array word by combining its previous contents with the selected mask using bitwise OR. Other bits in that word are preserved by this operation.

The array-base pointer is obtained from the state object for the update after the query call. The behavior must not be specified as using an array pointer cached before that call.

The function does not independently check the bit before incrementing the field. Thus, its “already present” decision is entirely delegated to F-88658862e573. If that dependency returns zero for an already-set bit, the field is still incremented even though the OR operation leaves the word unchanged.

## Side effects

Directly established effects are:

- Four calls to F-59c1430f85c2, interleaved with three calls to F-6ec4235d8ea3 in the order specified above.
- A possible indirect dispatch through the masked third input.
- A possible sequence of decoder and membership-query calls.
- For each zero membership-query result, an increment at object offset 4 followed by an OR update to one array word.

The function does not visibly allocate or release heap storage. It provides local scratch storage and local cursor storage to dependencies. No ownership transfer, retention permission, cleanup obligation, or lifetime extension can be established from those calls alone.

The captured sequence pointer is not visibly freed or reassigned from a dependency result. Its contents and lifetime remain dependency-sensitive.

The retained state-object address is reused across loop iterations. There is no visible reacquisition or lifetime validation after a decoder or membership-query call. Normal execution therefore requires that dependencies leave subsequent accesses valid; this is not a guarantee that they preserve ownership or memory.

The counter and array updates are separate operations, not a transaction. No locking, atomic operation, rollback, or synchronization is visible. Concurrent access is outside the established contract.

## Dependencies

| Opaque function ID | Established role at this call site | Unestablished properties |
|---|---|---|
| F-59c1430f85c2 | Receives local scratch storage and preparation arguments four times | Complete signature, final-call fourth argument, output representation, relation to captured sequence storage, allocation and lifetime effects |
| F-6ec4235d8ea3 | Receives the original third input, fourth input, and captured pointer three times | Input interpretation, mutations, ownership effects, error behavior |
| F-d3922c7fe0ec | Receives the address of the sequence cursor and returns a 32-bit value | Encoding, cursor-advance rules, validation, malformed-input handling, termination guarantees |
| F-88658862e573 | Receives the retained object address, word index, and bit position; its zero/nonzero result controls updates | Whether it actually reads a membership bit, all mutations, complete return-value meanings, lifetime effects |

The indirect dependency has no supplied opaque identifier. Its target is selected through the dispatch-table entry at byte offset 40. Its zero/nonzero result controls whether sequence processing occurs.

No dependency is assumed to preserve globals, preserve the contents of the captured storage, or leave an object alive unless that property is supplied separately.

## Acceptance examples

These examples are conditional observations, not runtime test results.

1. **Masked-null third input.**  
   If the third input is either zero or one, clearing its low bit produces zero. Provided all preceding calls return normally, the function still performs the four preparation calls and three interleaved calls, then finishes without indirect dispatch or sequence processing.

2. **Low-bit-tagged address.**  
   If the third input consists of a valid even object address with its low bit set, the three calls of F-6ec4235d8ea3 receive the tagged value. Final dispatch uses the even address. This does not establish what the low bit means.

3. **Null result from indirect dispatch.**  
   With a valid dispatch object whose selected callable returns zero, no sequence byte is tested by the final loop and no decoder or membership-query call occurs.

4. **Empty sequence after object acquisition.**  
   If the indirect call returns a valid nonzero state-object address and the cursor points to a zero byte, the function performs no decoder call and no explicit state-object update.

5. **Item at the first word boundary.**  
   If decoding returns 32, the membership query receives word index 1 and bit position 0. If it returns zero and the object fields remain valid, the field at offset 4 increases by one and bit 0 of array word 1 is set.

6. **Highest bit in a word.**  
   If decoding returns 31, the query receives word index 0 and bit position 31. The intended 32-bit mask selects hexadecimal 80000000. Executability of this high-bit shift depends on the original machine semantics, as distinguished from the supplied expression notation.

7. **Nonzero membership result.**  
   If the query returns any nonzero value, neither explicit state write occurs for that iteration. Dependency-side mutations are not excluded.

8. **Repeated decoded value.**  
   If the same value is decoded twice and the second query reports nonzero after the first insertion, only the first iteration increments the field. If both queries report zero, both iterations increment it, regardless of whether the second OR changes the array word.

9. **Changed global pointer.**  
   If a dependency changes the global pointer-valued location after its initial capture, later explicit uses still carry the originally captured pointer. Changes to storage reachable through that pointer can nevertheless affect processing.

## Uncertainties

The following are **documented_only**:

- The complete calling convention and parameter lists, particularly the final preparation call and the indirect dispatch.
- The size and layout of the scratch storage and its relationship, if any, to the sequence.
- The identity, encoding, bounds, ownership, and lifetime of the captured storage.
- The semantic meanings of the preparation labels, numeric arguments, counter, and bit array.
- Error handling inside dependencies and behavior on malformed or unterminated input.
- Whether the decoder guarantees progress.
- Whether the membership dependency preserves the state object or changes its fields.
- The counter’s valid range and behavior when incrementing the largest representable signed value. The supplied representation uses a signed 32-bit increment; a portable overflow policy is not established.
- Exact language-level semantics of shifting into bit position 31. The 32-bit bit-selection intent is evident, but no portable signed-shift guarantee follows.
- Valid array capacity. A full-width decoded value can yield a word index as large as 134,217,727 and a byte displacement as large as 536,870,908. No bounds check is visible, and adding that displacement to a base address may exceed available address space.
- Fine-grained ordering of reads within the array read-modify-write expression. The counter update precedes the array update, but a stronger atomic or bus-level ordering contract is unsupported.

## Evidence

This contract is based exclusively on the supplied static function representation and opaque dependency mapping.

Established evidence includes call order, explicitly supplied arguments, one-time pointer capture, low-bit masking, dispatch-table offset, zero/nonzero branches, cursor-mediated iteration, 32-bit index decomposition, object-field offsets, and conditional state writes.

Display configuration, character decoding, membership tracking, and population counting are interpretations consistent with that structure, not independently established semantics.

No dependency implementations, authoritative ABI description, allocation contracts, runtime traces, or runtime tests were supplied. No runtime tests have been performed.
