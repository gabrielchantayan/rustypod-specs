<!-- Trusted provenance: F-64f05dadf1ac; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-64f05dadf1ac

## Purpose

F-64f05dadf1ac conditionally obtains a pair of 32-bit output values and returns a 32-bit result. A one-byte discriminator in a mutable state object selects one of two behaviors:

- **Discriminator 1:** invoke F-37f0845bb2ad, potentially copy two outputs from its results, and either return a nonzero result word or enter a completion/error path.
- **Discriminator 2:** when a 32-bit remaining-count field is nonzero, consume one entry from a table, publish its first and third words, advance an index, decrement the count, and return an address-like sum involving its second word.
- **Other discriminators, or discriminator 2 with a zero count:** return zero without writing either output.

The apparent iterator or record-extraction role is inferred intent. The established contract is the field access, dependency invocation, output publication, and state transition described below. In particular, a zero return does not establish that outputs were left untouched.

## Interface

### Arguments and return

The function has four arguments:

| Argument | Meaning established by its use |
|---|---|
| State | Address of mutable, byte-addressable storage containing a discriminator and mode-dependent fields. |
| First output | Address at which a four-byte value may be written. |
| Second output | Address at which a four-byte value may be written; also passed indirectly to F-37f0845bb2ad in discriminator-1 mode. |
| Auxiliary value | An opaque four-byte input used only in discriminator-1 mode. |

The return is represented as a 32-bit integer. Some paths return zero; other paths return a word obtained from dependency-provided storage or a sum of two 32-bit values.

A nonzero return is not a universal Boolean success indicator: it may itself carry an address-like or other integer value. Conversely, discriminator-2 processing can complete its writes and state updates while returning zero.

### State layout

Offsets are byte offsets from the state address. Fields are mode-dependent; offset 16 is reused.

| Offset | Width | Use |
|---|---:|---|
| 0 | 1 byte | Discriminator. Only values 1 and 2 select active paths. |
| 4 | 4 bytes | In discriminator-2 mode, pointer-like reference to a descriptor. |
| 12 | 4 bytes | In discriminator-2 mode, pointer-like reference to another descriptor. |
| 16 | 4 bytes | In discriminator-1 mode, dependency input and completion/error check. In discriminator-2 mode, current entry index. |
| 20 | 4 bytes | In discriminator-2 mode, remaining count. |

The discriminator-2 descriptor referenced at state offset 4 supplies:

- A four-byte base value at descriptor offset 0.
- A four-byte base value at descriptor offset 8.

The descriptor referenced at state offset 12 supplies a four-byte displacement at descriptor offset 12.

The table selected using these values consists of entries with a **12-byte stride**:

| Entry offset | Width | Use |
|---|---:|---|
| 0 | 4 bytes | Value written to the first output. |
| 4 | 4 bytes | Value added to the base from the first descriptor’s offset 8 to form the return value. |
| 8 | 4 bytes | Value written to the second output. |

These accesses establish a 32-bit address-oriented representation in the supplied evidence. They do not establish portability to a wider-pointer environment, byte order, or permissibility of unaligned accesses.

### Preconditions

For each executed path, all accessed state fields, descriptor fields, table entries, and output destinations must be accessible for the indicated operation. No null, bounds, alignment, or capacity validation is visible.

Output destinations need not be valid on an inactive path because that path does not dereference them. In active paths, validity requirements depend on whether execution reaches the corresponding write.

No disjointness requirement is enforced. Aliasing can change later reads, selected records, return values, or state updates. The ordinary description assumes nonoverlapping state, descriptors, table storage, outputs, and dependency result storage; alias-sensitive behavior is identified separately.

## Behavior

### Initial dispatch

The function begins with a zero result and reads the one-byte discriminator.

If the discriminator is neither 1 nor 2, it returns zero. It does not intentionally modify the state or outputs and does not invoke a dependency.

For discriminator 2, it additionally reads the four-byte count at offset 20. A zero count produces the same no-write, zero-return outcome. A negative count is not rejected: the condition is nonzero, not positive.

### Discriminator 1: dependency-mediated retrieval

The function creates two local working slots:

- A four-byte value slot initialized from the auxiliary argument.
- A pointer-valued slot initialized to the second-output address.

It reads the four-byte state field at offset 16 and calls F-37f0845bb2ad with:

1. That field’s value.
2. The address of the local value slot.
3. The address of the local pointer slot.

The dependency can therefore replace the value slot, replace the pointer slot, and potentially access storage through the initially supplied pointer. No stronger dependency behavior is established.

#### Nonzero dependency result

If F-37f0845bb2ad returns nonzero, the function performs these operations in order:

1. Writes the current local value-slot contents to the first output.
2. Reads the first four-byte word through the current local pointer and writes it to the second output.
3. Reads the next four-byte word through that pointer, at offset 4.
4. If that word is nonzero, reads that word again and returns the value of the return-path read.

Under ordinary stable-memory assumptions, the nonzero word tested in step 3 is the returned value. The evidence nevertheless contains a separate read for the return, rather than a saved copy of the tested value.

The pointer supplied by the dependency is not checked for null or accessibility. Reaching this path requires its first two four-byte words to be readable.

The output writes precede the result-word check. Consequently, both outputs may be updated even if the function subsequently returns zero or enters the error path.

#### Zero dependency result or zero result word

If the dependency returns zero, the two output-copy operations above are skipped.

If the dependency returns nonzero but its result word at offset 4 is zero, the output copies have already occurred.

Both cases then follow the same completion/error logic:

1. Read the current four-byte state field at offset 16 again.
2. If it is nonzero, invoke F-c2bce6cc767e and then F-feee9156a257, assuming the first dependency returns.
3. Otherwise, store zero to each byte at offsets 16, 17, 18, and 19, in that order, and return zero.

The check uses the field’s value **after** F-37f0845bb2ad and any output writes. It is not a test of the originally passed value.

F-feee9156a257 is marked nonreturning in the supplied static evidence. On that basis, the nonzero-field path has no normal return. The evidence does not establish what either error-path dependency does internally.

The four clearing stores are individual byte writes, not an established single four-byte store. They do not reset the discriminator or any other state field.

### Discriminator 2: table retrieval

When the count at offset 20 is nonzero, the function processes one table entry.

First, it computes and retains a table address by adding:

- The four-byte value at offset 0 of the descriptor referenced by state offset 4.
- The four-byte value at offset 12 of the descriptor referenced by state offset 12.

It then performs the following operations in observable sequence:

1. Reads the current index from state offset 16, selects an entry with a 12-byte stride, and writes that entry’s word at offset 8 to the second output.
2. Reads the four-byte return-base value at offset 8 of the descriptor currently referenced by state offset 4.
3. Reads the current index again and obtains the selected entry’s word at offset 4.
4. Reads the current index again, obtains the selected entry’s word at offset 0, and writes it to the first output.
5. Forms the return value by adding the retained return-base value and retained entry-offset value.
6. Reads the current index, increments it by one, and writes it to state offset 16.
7. Reads the current count, decrements it by one, and writes it to state offset 20.
8. Returns the computed sum.

The discriminator remains unchanged.

For nonoverlapping, stable storage, all three table-word reads use the same original index. The ordinary transition is therefore: publish one entry, advance the index by one, and reduce the count by one.

The implementation does not establish that the table access is within bounds. A nonzero count alone is insufficient to establish validity.

### Aliasing and read-order consequences

The operation order matters when storage overlaps:

- In discriminator-1 mode, writing the first output can alter storage that the subsequent pointer-based reads will use.
- Writing the second output can alter the next word that is checked and potentially returned.
- Either output write can alter state offset 16 before the completion/error check.
- In discriminator-2 mode, the second output is written before the later descriptor and index reads. An overlapping destination can redirect those reads or change which entry supplies subsequent words.
- The first-output write occurs before the index and count updates. If it overlaps either field, those updates use the overwritten value.
- If the two outputs identify the same ordinary four-byte location, discriminator-1 mode writes the auxiliary/dependency value first and the pointer-derived value second. Discriminator-2 mode writes the entry’s third word first and its first word second. The later write determines the final bytes, subject to effects on intervening reads.

The evidence orders the operations listed above, but does not fully establish operand-read ordering within every address calculation. It provides no concurrency, atomicity, or volatile-memory contract.

## Side effects

### Direct memory effects

Depending on the path, the function directly writes:

- Four bytes to the first output.
- Four bytes to the second output.
- Four separate zero bytes at state offsets 16 through 19.
- An incremented four-byte index at state offset 16.
- A decremented four-byte count at state offset 20.

No direct allocation or release operation is visible. The function does not visibly retain an argument pointer in persistent storage.

Those observations do not establish ownership behavior of dependencies. In particular, F-37f0845bb2ad receives addresses of temporary local slots; whether it retains those addresses, modifies other storage, or changes ownership is unknown. The wrapper supplies no visible lifetime protection for retained local-slot addresses.

### Error and termination effects

No recoverable error object or separate error code is constructed.

For discriminator 1, reaching the completion/error logic with a nonzero state field at offset 16 causes dependency invocation rather than an ordinary zero return. Termination, unwinding, or other behavior beyond those calls is not established.

Invalid pointers, insufficient storage, out-of-range indices, and arithmetic overflow are not handled by visible checks.

## Dependencies

| Opaque function ID | Established interaction | Unknown behavior |
|---|---|---|
| F-37f0845bb2ad | Called only in discriminator-1 mode. Receives a four-byte state value and addresses of two local working slots. Its zero/nonzero return controls whether outputs are copied. | Meaning of the state value, result ownership, pointer validity, external mutations, lifetime rules, and failure semantics. |
| F-c2bce6cc767e | Called when discriminator-1 completion/error logic observes a nonzero state field at offset 16. No explicit arguments are visible. | Effects on globals, resources, control flow, and error state. |
| F-feee9156a257 | Called after F-c2bce6cc767e returns on that path. Marked nonreturning in the evidence. | Exact termination mechanism and externally visible effects. |

No dependency is assumed to preserve global or shared memory. The wrapper’s rereads are described using the values actually present when those reads occur.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless stated otherwise, storage is valid, nonoverlapping, stable, and arithmetic is representable.

1. **Inactive discriminator.**  
   Given discriminator 0, the function returns zero, leaves both outputs unchanged, and makes no dependency calls. The same applies to any discriminator other than 1 or 2.

2. **Exhausted table mode.**  
   Given discriminator 2 and count zero, the function returns zero without reading table entries or changing the index, count, or outputs.

3. **Ordinary table entry.**  
   Given discriminator 2, index 3, count 2, a return base of 4096, and entry 3 containing words 17, 24, and 99, the first output becomes 17, the second becomes 99, the return is 4120, the index becomes 4, and the count becomes 1.

4. **Zero-valued table result still consumes an entry.**  
   If the return base and selected entry’s second word sum to zero, the function still publishes both output words and updates index and count. Its return is zero.

5. **Negative count is active.**  
   With discriminator 2 and count −1, the path is entered. If all accesses and arithmetic are valid, one entry is processed and the count becomes −2. Negative values are not treated as exhausted.

6. **Dependency supplies a nonzero result word.**  
   Suppose F-37f0845bb2ad returns nonzero, leaves the local value slot as 7, and supplies readable words 42 and 9 through the local pointer. The outputs become 7 and 42, and the function returns 9 without executing the completion/error check.

7. **Dependency declines, state field already zero.**  
   Suppose F-37f0845bb2ad returns zero and the subsequent state-field read is zero. The wrapper performs no output-copy writes, clears the four bytes at offset 16 individually, and returns zero. Any dependency-originated output changes remain outside that no-copy statement.

8. **Outputs published before completion.**  
   Suppose the dependency returns nonzero and supplies words 42 and 0, while the state field checked afterward is zero. Both outputs are written before the byte-clearing operation, and the function returns zero.

9. **Completion/error path with nonzero state field.**  
   If that same completion/error check instead reads a nonzero value, F-c2bce6cc767e is called, followed by F-feee9156a257 if execution continues. No normal return is established.

10. **Overlapping table outputs.**  
    If both output addresses name the same ordinary four-byte location outside all source and state storage, the final stored value is the entry’s first word, because its write follows the third-word write.

## Uncertainties

### Established static evidence

The dispatch values, field offsets, access widths, entry stride, branch conditions, direct writes, dependency-call order, and repeated index reads are visible in the supplied material.

### Inferred intent

The discriminator-2 fields resemble an entry index and remaining count. Its returned sum resembles a base-relative address. Discriminator 1 resembles retrieval through a separate producer. These interpretations are not required for the mechanical contract.

### documented_only

The following remain unestablished:

- Byte order and alignment requirements.
- Valid state, descriptor, and table capacities.
- Whether callers prohibit aliasing.
- Ownership and lifetime of dependency-provided storage.
- Whether any pointer-like value represents a conventional process address.
- The precise meaning of dependency return values beyond zero/nonzero branching.
- Error-path diagnostics and termination mechanism.
- Behavior under concurrent mutation.
- Integer-overflow semantics.

Address formation, index multiplication by 12, return-value addition, increment, and decrement operate on values represented as 32-bit integers in the evidence. Signed overflow and address wraparound are not assigned defined behavior here. Acceptance examples require representable arithmetic rather than assuming modular wraparound.

## Evidence

This contract is based solely on the supplied static representation of F-64f05dadf1ac and the supplied opaque dependency mapping. Dependency implementations and external callers were not provided.

No runtime tests have been performed. The acceptance examples describe conditional consequences of the visible control flow and memory operations; they are not observed executions.
