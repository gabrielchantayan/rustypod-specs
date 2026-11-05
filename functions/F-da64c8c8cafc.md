<!-- Trusted provenance: F-da64c8c8cafc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-da64c8c8cafc

## Purpose

F-da64c8c8cafc conditionally constructs a pair of 32-bit values from an input record, removes at most one leading and one trailing double-quotation code unit from the corresponding boundaries, and passes the resulting pair through a sequence of dependencies.

The function returns a 64-bit value containing its final two boundary words. When the initial predicate returns zero, the return instead contains the first and second arguments unchanged.

**Established behavior:** recognition of the 16-bit value 0x0022, independent adjustment of two boundary words, ordered dependency calls, and packing of the final words into a 64-bit result.

**Inferred intent:** the two words likely describe the beginning and exclusive end of a region of 16-bit text, and the adjustments likely remove surrounding quotation marks. Neither text encoding nor a valid, ordered region is established. The contract therefore treats them as boundary words rather than guaranteeing a string or span.

## Interface

### Arguments

The interface has four arguments, each occupying a 32-bit word in the supplied representation:

| Argument | Interpretation established by direct use |
|---|---|
| Primary address | Used as the initial low return word, forwarded unchanged to one dependency, and, on the active path, used to locate a 32-bit field at byte offset 20. |
| Selector word | Passed to the initial predicate and used as the initial high return word. Although represented as a signed 32-bit integer, no local signed comparison is performed on it. |
| Range-record address | On the active path, identifies two consecutive 32-bit words at byte offsets 0 and 4. Its original value also initializes a separate context supplied to later dependencies. |
| Auxiliary word | An uninterpreted 32-bit value used to initialize the second word of that context. |

There is no separately supplied length, encoding indicator, error-output address, or ownership flag.

### Local records exposed to dependencies

The function initializes two adjacent-word records before calling the initial predicate:

- A **boundary record**, initially containing the primary address followed by the selector word.
- A **context record**, initially containing the range-record address followed by the auxiliary word.

The active path replaces the boundary record’s words with the two words read from the range record before performing boundary adjustments.

Dependencies receive the address of the context record. The first active-path dependency also receives the address of the boundary record. These are addresses of local storage, not newly allocated objects.

The context has an observed initial layout of two 32-bit words at offsets 0 and 4. This does not establish that any dependency interprets the context as an address-and-length pair, an ownership descriptor, or any other particular abstraction.

### Return value

The return is a 64-bit packed value:

- Bits 0–31 contain the final first word of the boundary record.
- Bits 32–63 contain the final second word.

Packing preserves each word’s 32-bit bit pattern. A second word with its top bit set is not sign-extended beyond its designated 32-bit half.

This is a bit-level result layout, not a specified byte order for storing the result in memory.

### Preconditions

Required memory validity depends on the executed path:

- The global-backed 32-bit predicate argument must be readable.
- The predicate’s own requirements must hold; they are not established here.
- If the predicate returns nonzero, the range record must permit the two 32-bit reads at offsets 0 and 4.
- A nonzero first boundary must permit a 16-bit read at that boundary.
- A nonzero second boundary must permit a 16-bit read at the address two bytes before it.
- If execution reaches the update dependency, the 32-bit field at primary-address offset 20 must be readable.
- Requirements imposed by the active-path dependencies must also hold.

A zero range-record address is not rejected before dereference on the active path. A zero primary address is not rejected before the offset-20 access.

## Behavior

### 1. Initialization and predicate

Before the predicate call, the boundary record contains the first two arguments, and the context record contains the third and fourth arguments.

The function reads a 32-bit word at byte offset 124, or 0x7c, from a global-backed base. It supplies that word and the selector word to **F-a1e68ae27737**, with the selector first.

Only zero versus nonzero matters for the predicate result.

The global-backed value is obtained before the predicate executes. Later changes to that global state do not change the argument already supplied to this invocation.

### 2. Predicate returns zero

The function skips all active-path work:

- No direct read is made through the range-record address.
- No boundary code unit is inspected.
- No direct read is made at primary-address offset 20.
- None of the other four dependencies is invoked.

The boundary record retains its initial words. The result therefore has the primary-address bit pattern in its low half and the selector bit pattern in its high half.

This path does not establish that the predicate is free of side effects.

### 3. Predicate returns nonzero

The function reads the first 32-bit word from the range record, then the second 32-bit word at offset 4. It stores them into the boundary record in that order.

These reads happen after the predicate returns. If the predicate changes the memory holding the range record, the subsequent reads observe that memory’s post-call state.

Both range-record words are read before either boundary code unit is inspected.

### 4. First-boundary adjustment

If the first boundary is zero, the function performs no 16-bit read through it and leaves it unchanged.

Otherwise, it reads one 16-bit code unit at that boundary:

- If the value equals 0x0022, it advances the boundary by two bytes.
- For every other value, it leaves the boundary unchanged.

Only one code unit is considered. Repeated leading quotation marks are not repeatedly removed. The comparison is against the complete 16-bit value, not merely its low byte.

The signed representation of the 16-bit load does not change which value matches 0x0022.

### 5. Second-boundary adjustment

After the first-boundary processing, the function examines the second boundary independently.

If the second boundary is zero, it performs no trailing-code-unit read and leaves the boundary unchanged.

Otherwise, it reads one 16-bit code unit at the address two bytes before that boundary:

- If the value equals 0x0022, it subtracts two from the second boundary.
- For every other value, it leaves the boundary unchanged.

The second boundary is not checked against the first boundary. This operation does not require, or verify, that the boundaries describe a nonempty or ordered region.

The first and second boundary probes may read the same code unit. They remain separate reads, with the first-boundary probe occurring first.

### 6. Ordered dependency sequence

After boundary adjustment, the function invokes these dependencies in order:

1. **F-34348fd1b45d**, receiving:
   - the address of the context record;
   - the original primary address;
   - the address of the boundary record.
2. **F-44362605ab79**, receiving the address of the context record and producing a 32-bit result.
3. **F-2444b7019483**, receiving:
   - a 32-bit value loaded from byte offset 20 relative to the original primary address;
   - the result from F-44362605ab79.
4. **F-289ec4311c64**, receiving the address of the context record.

The primary-address field is read after F-44362605ab79 returns. Consequently, changes made by earlier dependencies to the memory containing that field can affect the value passed to F-2444b7019483.

The field address remains relative to the original primary argument. It is not computed from the adjusted first boundary.

The supplied implementation does not use results from F-34348fd1b45d, F-2444b7019483, or F-289ec4311c64 to choose another local branch.

### 7. Return after dependency processing

The function packs the boundary record’s words only after F-289ec4311c64 returns.

F-34348fd1b45d receives the boundary record’s address and may modify it. Other dependencies might also affect it through aliases or retained addresses; their implementations are unknown.

Accordingly, the active-path return is **not unconditionally the locally trimmed pair**. It is the pair present in the boundary record after all four calls complete.

## Side effects

### Direct memory operations

The direct read order is:

1. Global-backed predicate word.
2. Predicate call.
3. On a nonzero result, first range-record word.
4. Second range-record word.
5. First-boundary 16-bit probe, when its boundary is nonzero.
6. Second-boundary 16-bit probe, when its boundary is nonzero.
7. First active-path dependency.
8. Second active-path dependency.
9. Primary-address field at byte offset 20.
10. Update dependency.
11. Final context dependency.

Direct writes are to local records. No direct write to the input range record or boundary-addressed storage is shown.

Writes, allocation, release, global-state changes, callbacks, and other effects performed by dependencies are unknown. Lack of a direct external write does not make this function observationally read-only.

### Ownership and lifetime

The function shows no explicit allocation or release operation of its own.

The local context’s initialization from the range-record address and auxiliary word does not establish copying, ownership transfer, or reference counting. Likewise, the final dependency’s position does not prove that it destroys or releases anything.

Pointers to local records are valid during their calls. Any dependency retaining those pointers beyond the lifetime of this invocation would require an additional, currently undocumented contract.

### Errors and interruption

There is no explicit validation failure branch or dedicated error return.

In particular, the function does not locally reject:

- An invalid range-record address.
- Unreadable or misaligned boundary storage.
- Reversed boundaries.
- A nonzero second boundary below two.
- An invalid primary-address field.
- An error-like value returned by F-44362605ab79.

The latter value is forwarded unchanged as a 32-bit argument.

Invalid accesses may prevent normal completion. The supplied evidence does not specify fault handling, exception behavior, or recovery. The final dependency is reached only if the earlier operations return normally; no guaranteed cleanup mechanism is established.

## Dependencies

| Opaque ID | Established role | Not established |
|---|---|---|
| F-a1e68ae27737 | Receives the selector and global-backed word; zero skips the active path, nonzero enables it. | Predicate meaning, memory effects, global preservation, failure behavior. |
| F-34348fd1b45d | Receives the local context address, original primary address, and local boundary-record address. | Whether it copies, consumes, retains, transforms, or owns any referenced data. |
| F-44362605ab79 | Receives the context address; its 32-bit result is forwarded to F-2444b7019483. | Result semantics, validity conditions, allocation behavior, error encoding. |
| F-2444b7019483 | Receives the late-read primary field and the preceding dependency’s result. | Whether it stores, consumes, releases, or otherwise interprets either argument. |
| F-289ec4311c64 | Receives the context address last. | Whether it performs cleanup, releases resources, or preserves any state. |

No dependency is assumed to preserve globals or referenced storage.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests.

### Predicate rejects

If the primary argument’s bit pattern is 0x00102030, the selector is 0x00000007, and the predicate returns zero, the result is **0x0000000700102030**.

The range-record address and auxiliary word do not affect this packed result. The function makes no direct read through the range-record address on this path.

### Both boundary code units are quotation marks

Suppose the predicate returns nonzero, the range record contains 0x00001000 and 0x0000100a, and readable 16-bit values at 0x00001000 and 0x00001008 are both 0x0022.

The boundary record passed to F-34348fd1b45d contains:

- First word: 0x00001002.
- Second word: 0x00001008.

If the dependencies leave that record unchanged and complete normally, the result is **0x0000100800001002**.

### Independent null-boundary handling

Suppose the first boundary is zero and the second is 0x00002004, with 0x0022 readable at 0x00002002.

No first-boundary probe occurs. The boundary record passed onward contains zero and 0x00002002.

Conversely, a zero second boundary suppresses only the trailing probe. It does not suppress a valid leading probe.

### A single quotation code unit

Suppose the boundaries are 0x00003000 and 0x00003002, with a stable 0x0022 at 0x00003000.

Both probes read that same address. The first boundary becomes 0x00003002 and the second becomes 0x00003000. The function passes this reversed pair onward without repairing or rejecting it.

### Full-width comparison

If a probed 16-bit value is 0x0122 rather than 0x0022, that boundary is not adjusted. A matching low byte alone is insufficient.

### Dependency-visible mutation

If F-34348fd1b45d changes the boundary record to two different words, and no subsequent operation changes them, those words determine the return.

If F-44362605ab79 changes the field at primary-address offset 20, the following field read supplies the changed value to F-2444b7019483.

## Uncertainties

- **documented_only — Data model:** The boundary pair resembles a half-open text range, but its semantic type and encoding are unknown.
- **documented_only — Arithmetic extremes:** Adjustments are two-byte operations on 32-bit representations. The second-boundary subtraction has no underflow guard. No portable success behavior is established for address overflow, underflow, invalid pointer arithmetic, or inaccessible resulting addresses.
- **documented_only — Alignment and byte order:** The accesses are 16-bit or 32-bit as described. Target alignment requirements and in-memory byte order are not established here.
- **documented_only — Global state:** The meaning, initialization, and validity of the global-backed base and offset-124 field are unknown.
- **documented_only — Dependency contracts:** Mutation, ownership, retention, error handling, and resource lifetimes require independent specifications.
- **documented_only — Concurrent mutation:** Separate reads are visible in the supplied representation, but no synchronization, atomicity, or coherent-snapshot guarantee is established.
- **documented_only — Exceptional control flow:** Faults or nonlocal exits may bypass later calls and the normal return.

## Evidence

This contract is based solely on static inspection of the supplied function representation and dependency mapping.

Established evidence includes the four argument roles, 32-bit words, 16-bit quotation comparisons, two-byte boundary adjustments, offsets 4, 20, and 124, zero/nonzero predicate branch, dependency order, local-record addresses passed to dependencies, and 64-bit result packing.

Text trimming and cleanup are possible interpretations, not established dependency semantics. No implementation bodies for the dependencies were supplied. No runtime tests have been performed.
