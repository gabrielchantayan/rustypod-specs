<!-- Trusted provenance: F-382bffe8f3c1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-382bffe8f3c1

## Purpose

F-382bffe8f3c1 conditionally performs module-related preparation, examines an object’s flag and signed state, and may resolve and initialize that object. Its normal return is a 64-bit value containing a zero-or-one outcome in the low 32 bits and an address in the high 32 bits.

Established behavior includes diagnostic calls, lookup, temporary context changes, ordered transfer and clearing of two result fields, and a 16-bit flag update.

Module preparation, lazy initialization, and circular-definition detection are inferred purposes. They do not establish an object lifecycle or ownership model.

Two ordering distinctions are essential:

- Preparation selection uses an initially sampled object flag, captured **before** the fourth argument is written to local temporary storage. A later flag read controls a separate shortcut.
- Initialization writes a negative state marker before temporarily clearing another location. If those locations overlap, the object state need not remain negative when the initialization dependency begins.

## Interface

### Arguments and representation

The supplied representation exposes four arguments:

| Argument | Established use |
|---|---|
| Context address | Identifies storage containing a 32-bit address at offset `0x00` and a 32-bit field at offset `0x44` that is saved and restored during initialization. |
| Object address | Identifies storage accessed through 8-, 16-, and 32-bit operations, including fields containing further addresses. |
| Auxiliary value | An opaque 32-bit value passed to F-f4457ebb35af. |
| Initial temporary value | A 32-bit value written to local temporary storage after the initial object-flag sample. Before this storage is passed to F-f4457ebb35af, it is overwritten with zero. |

Addresses in this representation are 32 bits. Address validity, alignment, byte order, and address-overflow behavior are not established.

The first context word is read at entry and retained as an address, called the **entry-retained address** below. Later changes to the context’s first word do not replace that retained value.

For a nonnull object, its byte at offset `0x39` is also read and retained at entry, after the first context read but before the initial temporary write. This **initial flag sample** is distinct from the later flag read.

### Object layout

Offsets are byte offsets from the object address.

| Offset | Access width | Established use |
|---|---:|---|
| `0x00` | 32 bits | Argument for the negative-state diagnostic. |
| `0x04` | 32 bits | State, interpreted as signed for decisions; also an initialization-transfer destination. |
| `0x08` | 32 bits | Second initialization-transfer destination. |
| `0x18` | 32 bits | Lookup argument. |
| `0x39` | 8 bits | Initially sampled preparation flag, then independently reread for a shortcut. |
| `0x3C` | 32 bits | Module descriptor address. |
| `0x40` | 32 bits | Preparation gate; zero permits preparation. |
| `0x48` | 32 bits | Address dereferenced for the missing-descriptor diagnostic argument. |
| `0x4C` | 32 bits | Address leading to a 16-bit field at offset `0x5A`. |

For signed state comparisons, `0x00000000` is zero, `0x00000001` through `0x7FFFFFFF` are positive, and `0x80000000` through `0xFFFFFFFF` are negative.

Other accessed storage includes:

- The descriptor’s first 32-bit word, interpreted as an address, and a 32-bit value at offset `0x08` from that address.
- A 32-bit field at offset `0x0C` in a nonzero lookup result.
- A 32-bit field at offset `0xE0` from the entry-retained address.
- Two 32-bit fields at offsets `0x04` and `0x08` in a nonzero initialization result.
- A 16-bit field at offset `0x5A` from the address subsequently read at object offset `0x4C`.

### Return value

Every established normal return has this layout:

| Bits | Meaning |
|---|---|
| `0–31` | Exactly zero or one, selected by the path taken. |
| `32–63` | A 32-bit address placed in the upper half of the result. |

The high-word address initially equals the object address. When descriptor-backed preparation is entered, it changes to the address of the local 32-bit temporary, before F-f4457ebb35af is called. It is not subsequently changed back.

Thus the high word is not consistently an object handle. Following descriptor-backed preparation, it identifies local storage whose lifetime ends when this invocation returns. No valid post-return dereference is established.

A nonzero preparation status produces low word one, not the original status value. A nonzero initialization-result address produces low word zero regardless of the signed state value transferred from that result.

### Preconditions

The context’s first word must be readable even on early-return paths.

A null object is not a supported no-op input. It bypasses initial flag sampling and preparation, but the later flag read still uses the object address without a null guard.

Every address reached on the selected path must support the corresponding access. No pointer validation, bounds checking, synchronization, or recovery from invalid accesses is established.

Non-aliasing is not an established precondition, including for local temporary storage. Retained values, access order, and final memory contents must therefore be distinguished.

## Behavior

### Entry ordering and preparation selection

Entry establishes the following order:

1. Read and retain the context’s first 32-bit word.
2. If the object is nonnull, read and retain its byte at offset `0x39`.
3. Select the object address as the initial return high-word address.
4. Write the fourth argument to the local 32-bit temporary.
5. Evaluate preparation eligibility using the object’s nonnull status and the retained initial flag sample. Only if both permit preparation is the 32-bit gate at object offset `0x40` read.

Preparation is selected when the object is nonnull, the retained initial flag sample is nonzero, and the subsequently read gate is zero.

The initial flag is not reread for this condition. If the temporary write overlaps the object flag and changes it, preparation selection still uses the earlier sample. Conversely, the gate is read after the temporary write, so an overlapping write can affect the gate value used by this decision. The entry-retained address likewise remains the value captured before the temporary write.

A nonzero initial flag sample does not itself imply an immediate successful return: preparation and its diagnostic paths take precedence when the gate is zero.

### Preparation without a descriptor

After preparation is selected, the descriptor address at object offset `0x3C` is read.

If it is zero, the function reads the address at object offset `0x48`, reads a 32-bit value through that address, and calls F-2022b0f73f3b with:

- The context address.
- A missing-module diagnostic template.
- The dereferenced value.

If the diagnostic returns normally, the function returns low word one and high word equal to the object address. Neither F-f4457ebb35af nor F-94cb53877c15 is called on this path.

### Preparation with a descriptor

For a nonzero descriptor address, the function overwrites the local temporary with zero and changes the return high-word address to that temporary’s address.

F-f4457ebb35af receives six arguments:

1. The context’s first word, read again for this call.
2. The object address.
3. The descriptor address.
4. The 32-bit value at offset `0x08` from the address contained in the descriptor’s first word.
5. The local temporary’s address.
6. The auxiliary value.

The newly read context word need not equal the entry-retained address. The supplied representation does not establish a relative evaluation order among the independent argument reads.

A nonzero preparation result causes a call to F-2022b0f73f3b with the context and a second diagnostic-template address. No third argument is explicitly supplied on that call.

After a zero preparation result, or after the nonzero-result diagnostic returns normally, F-94cb53877c15 receives the temporary’s current 32-bit contents. F-f4457ebb35af can write through the supplied temporary address; its contents are not guaranteed to remain zero. No preservation by the diagnostic dependency is assumed either.

After F-94cb53877c15 returns, a nonzero preparation result produces low word one. A zero preparation result continues to the later flag check. Both paths retain the temporary’s address in the return high word.

### Later flag and signed-state decisions

After successful preparation, or when preparation was not selected, the function independently reads the object byte at `0x39` again.

If this later flag value is nonzero, it returns low word zero without reading the state for the shortcut. The later flag may differ from the initial sample because of overlapping writes, dependency effects, or external mutation.

Otherwise, the state at `0x04` is read and compared as signed. A positive value returns low word zero.

If that shortcut does not apply, the state is read again for a signed nonnegative comparison. These reads do not form an atomic snapshot. Under stable storage, the remaining nonnegative case is zero; if storage changes between reads, the second value controls whether lookup is entered.

A negative value at the second comparison causes F-2022b0f73f3b to receive the context, a circular-definition diagnostic template, and the object’s first 32-bit word. Normal diagnostic return produces low word one. This branch performs no direct state reset.

### Lookup

The nonnegative branch calls F-5f042ee1ac4e with:

- The entry-retained address.
- The current 32-bit value at object offset `0x18`.

A zero lookup result returns low word one without a direct state write and without calling F-6c7ac9769d41.

A nonzero lookup result enters initialization.

### Initialization setup and restoration

The function saves the context field at `0x44`, then calls F-787923a5a4df with the context and the 32-bit field at lookup-result offset `0x0C`.

After that call returns, these operations are ordered:

1. Write `0xFFFFFFFF` to the object state.
2. Read and save the 32-bit field at entry-retained address plus `0xE0`.
3. Write zero to that `0xE0` location.
4. Call F-892de4fc8469 with the context, a zero second argument, and the lookup-result address.

The marker destination and the `0xE0` location are not proven disjoint. If they exactly coincide, the saved value is the just-written `0xFFFFFFFF`, and the zero write changes the state to zero before F-892de4fc8469 begins. Partial overlap can also alter the state.

Absent intervening mutation, the `0xE0` location has received zero at call entry. The object state remains `0xFFFFFFFF` only when the later zero write does not affect it.

Following a normal return from F-892de4fc8469, restoration occurs before its result is handled:

1. Write the saved value back to the `0xE0` location.
2. Write the saved context value back to context offset `0x44`.

These are ordered writes, not independent guarantees that overlapping storage regains its original contents. The context value was captured before F-787923a5a4df; the `0xE0` value was captured after that dependency and after the marker write.

No restoration is established for an abnormal or nonlocal exit.

### Zero initialization result

A zero result from F-892de4fc8469 selects low word one. After restoration, the function writes zero to the object state, calls F-6c7ac9769d41 with the lookup-result address, and returns.

There is no separate direct assignment to object offset `0x08` on this branch. Aliasing or dependency effects can nevertheless alter it. The state-zero write likewise does not establish that the final dependency preserves zero.

### Nonzero initialization result

A nonzero result address selects low word zero without validating its fields.

The transfer and clearing order is:

1. Read result offset `0x04`, then write that value to object offset `0x04`.
2. Read result offset `0x08`, then write that value to object offset `0x08`.
3. Write zero to result offset `0x04`.
4. Write zero to result offset `0x08`.

This is not an atomic two-field move or snapshot. Overlap can make the first destination write affect the second source read, and source clearing can affect the object.

Next, F-da1a289a3277 is called without explicit arguments. After it returns, the address currently held at object offset `0x4C` is used to reach the 16-bit field at offset `0x5A`. That field is read and written with mask `0x0002` set, preserving the other bits of the value read.

Finally, F-6c7ac9769d41 receives the lookup-result address, and the function returns low word zero. Neither positive transferred state nor a nonzero final state is required.

## Side effects

Direct writes affect:

- The local 32-bit temporary, initially after entry sampling and again before preparation.
- Object state at `0x04`.
- Object offset `0x08` on nonzero initialization result.
- Initialization-result offsets `0x04` and `0x08`.
- The 32-bit location at entry-retained address plus `0xE0`.
- Context offset `0x44`.
- The indirectly reached 16-bit field at offset `0x5A`.

The state marker is exactly the 32-bit pattern `0xFFFFFFFF`. The 16-bit flag operation is bitwise OR with `0x0002`, not addition. No size multiplication or numeric accumulation is established.

All final-memory claims are subject to overlap and later writes. In particular, the initial temporary write cannot retroactively alter either the entry-retained address or the retained flag sample, but it can affect subsequent memory reads if storage overlaps.

Dependencies may modify accessible memory or global state. Their allocation, release, retention, and ownership behavior is unknown. Clearing result fields is consistent with ownership transfer but does not prove it. Calls to F-94cb53877c15 and F-6c7ac9769d41 do not alone establish deallocation.

## Dependencies

| Opaque function ID | Established local contract |
|---|---|
| F-f4457ebb35af | Six-argument preparation call with writable local temporary storage. Zero permits continuation; nonzero triggers a diagnostic and eventual low word one after normal intervening returns. |
| F-2022b0f73f3b | Diagnostic dependency used on three paths with differing explicit arguments. Normal return is not established. |
| F-94cb53877c15 | Receives the temporary’s current contents after preparation, including after a nonzero preparation result if its diagnostic returns. |
| F-5f042ee1ac4e | Lookup using the entry-retained address and object field `0x18`. Zero bypasses initialization. |
| F-787923a5a4df | Receives the context and lookup-result field `0x0C`, after the context-field save and before the state-marker write. |
| F-892de4fc8469 | Receives the context, zero, and lookup result after marker placement and the `0xE0` read-and-clear sequence. Negative object state at call entry is conditional, not guaranteed. Zero selects failure; nonzero supplies a result address. |
| F-da1a289a3277 | Called without explicit arguments after result-field transfer and clearing, before the 16-bit update. |
| F-6c7ac9769d41 | Receives the nonzero lookup result after either normally completed initialization outcome; not called on lookup miss. |

No dependency is assumed to preserve globals, object fields, context contents, local storage reachable through supplied addresses, or ownership.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless stated otherwise, assume valid disjoint storage, stable reads, normal dependency returns, and no dependency mutations beyond those specified.

1. **Positive state.** Flag zero and state `7` produce low word zero and the object address in the high word. No dependency is called.

2. **Preparation gate blocks preparation.** A nonzero initial flag and nonzero gate produce low word zero if the later flag remains nonzero. No state read is needed for that shortcut.

3. **Initial flag survives an overlapping temporary write.** Suppose the initial sample is nonzero, but writing the fourth argument to local storage also changes the object flag to zero. With the subsequently read gate zero, preparation is still selected. Selection uses the retained nonzero sample, not the changed flag.

4. **A later nonzero flag does not retroactively select preparation.** Suppose the initial sample is zero, but the initial temporary write changes the flag to nonzero. Preparation is skipped and its gate is not read. The later flag check returns low word zero with the object address in the high word.

5. **Missing descriptor.** A nonzero initial flag, zero gate, and zero descriptor cause the missing-module diagnostic. Normal return produces low word one with the object address in the high word.

6. **Preparation failure.** A nonzero preparation result causes the second diagnostic, then F-94cb53877c15 with the current temporary value. Normal completion returns low word one with the temporary’s address in the high word.

7. **Preparation leaves the later flag set.** If preparation returns zero and the later flag is nonzero after F-94cb53877c15, the function returns low word zero without lookup. The high word remains the temporary’s address.

8. **Negative state.** With flag zero and stable state, both `0xFFFFFFFF` and `0x80000000` select the circular-definition diagnostic. Normal diagnostic return produces low word one.

9. **Lookup miss.** With flag zero and stable state zero, a zero lookup result produces low word one without a direct state write or F-6c7ac9769d41 call.

10. **Disjoint initialization failure.** A nonzero lookup result and zero initialization result cause ordered restoration, a state-zero write, and F-6c7ac9769d41. The low word is one.

11. **Disjoint initialization success.** Result fields `3` and `0x12345678` are transferred to the object and then cleared in the result. A subsequent 16-bit read of `0x0040` produces a write of `0x0042`. The low word is zero.

12. **State aliases the temporarily cleared location.** If entry-retained address plus `0xE0` exactly equals object offset `0x04`, initialization setup writes the marker, saves it, then clears that same location. F-892de4fc8469 begins with state zero, absent other mutation. If it returns zero without mutation, restoration writes the marker back before the failure branch writes state zero.

13. **Initialization result equals the object.** Transfer writes target their own source fields; subsequent clearing sets object offsets `0x04` and `0x08` to zero. The nonzero result address still selects low word zero.

## Uncertainties

The following remain **documented_only**:

- Complete argument and dependency types beyond observed widths and accesses.
- Address validity, alignment, byte order, and address-overflow behavior.
- Whether the execution environment permits particular overlaps, including overlaps with local temporary storage.
- Exact byte-level outcomes of unspecified partial overlaps.
- The intended consumer and usable lifetime of the return high word.
- Exact diagnostic formatting and whether diagnostics return or transfer control nonlocally.
- Dependency effects on memory, globals, retention, ownership, and resource lifetime.
- Reentrancy, concurrent mutation, and synchronization requirements.
- External invariants concerning positive state after initialization.
- Any intended external significance of the fourth argument’s initial temporary value.

Unknown aliasing does not erase the established access order. The initial flag sample precedes the temporary write; the gate and later flag reads follow it.

## Evidence

This contract is based exclusively on the supplied static representation and opaque dependency mapping.

Established evidence includes access widths and offsets, signed comparisons, short-circuit conditions, explicit dependency arguments, ordered transfers and restorations, and construction of the 64-bit return.

Entry evidence establishes the context-word read first, the nonnull-object flag sample second, and the initial temporary write afterward. Preparation uses the retained flag sample; it does not obtain a replacement flag value after that write. The gate is conditionally read later, and the subsequent shortcut independently rereads the flag.

The initialization aliasing case follows from the marker write preceding the `0xE0` read and zero write. It establishes why negative state at dependency entry is not unconditional.

Module preparation, lazy initialization, circularity detection, and ownership transfer remain inferred intent wherever they exceed those explicit observations.

No runtime tests have been performed.
