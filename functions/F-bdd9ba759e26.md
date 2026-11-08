<!-- Trusted provenance: F-bdd9ba759e26; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bdd9ba759e26

## Purpose

F-bdd9ba759e26 prepares an operation record and a temporary descriptor, conditionally obtains and initializes a context-associated object, selects an operation-specific object, installs descriptor information, and invokes an execution callback. If that callback returns zero, the function invokes finalization and returns its result unchanged.

The established behavior is an ordered sequence of memory accesses, calls, and zero/nonzero decisions. Construction, caching, initialization, installation, and cleanup are descriptive roles inferred from this sequence; they do not establish the implementations or ownership contracts of dependencies.

A missing required associated object produces the fixed result 6. Zero permits continuation at several intermediate stages. The finalization result is not checked: a zero finalization result produces a zero function result, but its operational meaning is unknown.

## Interface

### Arguments and result

| Argument | Representation | Observed use |
|---|---|---|
| Operation record | Address of storage accessed in 4-byte words | Receives context, associated-object, and selected-object values; supplies another dependency argument and an execution argument address. |
| Context | 32-bit address value | Supplies fields and receives conditional cache updates. |
| Selector | Uninterpreted 32-bit value | Passed unchanged to selection and finalization. |
| Flags | Unsigned 32-bit value | Supplies a descriptor nibble and transformed finalization flags. |

The result has a pointer-shaped, 32-bit representation. This function tests certain intermediate results for zero but does not dereference its returned value. Nonzero results are not established to be valid addresses.

Neither input address receives an explicit null check. Every reached access requires suitable readable or writable storage. Dependencies may require additional storage or alignment not established here.

### Operation-record layout

| Byte offset | Width | Direct behavior |
|---|---:|---|
| `0x00` | 4 bytes | Receives the context address. |
| `0x04` | 4 bytes | Receives the initial context cache value; is subsequently tested, possibly replaced, and passed to selection. |
| `0x08` | 4 bytes | Read as the argument to F-1cc54a4c66f8; not directly initialized here. |
| `0x0C` | Address only | Passed to the execution callback. The callback-accessed extent is unknown. |
| `0x80` | 4 bytes | Receives the selection output after selection returns zero. |

Direct record accesses extend through byte `0x83`; this is not a bound on dependency accesses.

### Context and associated-object layouts

| Context offset | Width | Direct behavior |
|---|---:|---|
| `0x10` | 4 bytes | Read as a signed size component for acquisition; reread for a candidate header. |
| `0x58` | 4 bytes | Supplies the required associated-object address. |
| `0x64` | 4 bytes | Supplies an acquisition argument. |
| `0x74` | 4 bytes | Supplies the initial cache value and conditionally receives the candidate address. |
| `0x78` | 4 bytes | Conditionally receives a designated global value. |

The object initially obtained from context offset `0x58` supplies 4-byte fields at offsets `0x10` and `0x14`.

On the acquisition path, a candidate may receive these 4-byte header writes:

| Candidate offset | Value |
|---|---|
| `0x00` | Context address. |
| `0x04` | Freshly read context field at `0x10`. |
| `0x08` | Candidate address plus 24 bytes. |

Candidate bytes `0x0C`–`0x17` receive no explicit initialization from this function.

### Invocation-local storage

The function uses a 28-byte descriptor:

| Descriptor offset | Width | Explicit assignment |
|---|---:|---|
| `0x00` | 4 bytes | Context address. |
| `0x04` | 4 bytes | Initial associated object’s field at `0x10`. |
| `0x08` | 4 bytes | Initial associated object’s field at `0x14`. |
| `0x0C` | 4 bytes | Zero. |
| `0x10` | 4 bytes | Zero. |
| `0x14` | 1 byte | Original flags bits 16–19, placed in bits 0–3. |
| `0x18` | 4 bytes | Zero. |

F-unknown-01 receives the descriptor address and length 28 before these assignments. Its effects are unknown. Descriptor bytes `0x15`–`0x17` have no explicit assignment by this function; they must not be described as zero merely because this helper runs.

A separate 4-byte temporary slot serves successively as acquisition-status storage, initialization-status storage, and selection-output storage. It has no explicit initialization before its first dependency call.

The descriptor and slot are local to this invocation, but their addresses escape to dependencies. Their contents cannot be assumed preserved across those calls.

## Behavior

### Required associated-object check

The first direct access reads context offset `0x58`.

If that value is zero, the function returns exactly 6. This path makes no direct operation-record or context writes and invokes none of the listed dependencies.

Otherwise, the value is retained as the address supplying the initial descriptor fields. The function does not reread context offset `0x58` to obtain those fields after F-unknown-01 returns. The object’s contents, however, are read after that call and are not established to remain unchanged during it.

### Descriptor and record preparation

The explicit preparation order is:

1. Call F-unknown-01 with the descriptor address and 28.
2. Read the retained associated object at `0x10` and assign descriptor offset `0x04`.
3. Assign zero to descriptor offset `0x0C`.
4. Read the retained associated object at `0x14` and assign descriptor offset `0x08`.
5. Assign zero to descriptor offset `0x18`.
6. Assign the extracted flags nibble to descriptor byte `0x14`.
7. Assign zero to descriptor offset `0x10`.
8. Write the context address to record offset `0x00`.
9. Read context offset `0x74` and write that value to record offset `0x04`.
10. Assign the context address to descriptor offset `0x00`.
11. Read record offset `0x08` and call F-1cc54a4c66f8 with that value.

The result of F-1cc54a4c66f8 is ignored.

The subsequent acquisition decision reads record offset `0x04`, not context offset `0x74`. Consequently, the decision observes the record field after F-1cc54a4c66f8 returns. The dependency is not established to preserve that field or any other reachable state.

### Conditional acquisition

If the decision reads a nonzero record field, acquisition is skipped. The function proceeds to selection without explicitly initializing the temporary slot.

If the decision reads zero, F-3ce007b692cc receives:

- The 4-byte value from context offset `0x64`.
- The signed 32-bit field from context offset `0x10`, increased by 24.
- The temporary slot’s address.

The supplied representation does not establish an evaluation order between the two context-field argument reads.

The addition has no local range or sign check. When the signed addition is representable, its result is passed to acquisition, including a negative result. Overflow behavior is unresolved; the absence of an established overflow interpretation does not imply rejection.

The direct acquisition result becomes the candidate address. The temporary slot is then read and tested as a status.

**Nonzero acquisition status:** the function skips candidate-header initialization and the initialization dependency. It still carries the returned candidate forward for a record-field write. There is no cleanup call solely because acquisition reports nonzero status.

**Zero acquisition status:** the function writes the candidate header in this order:

1. Context address at candidate offset `0x00`.
2. A fresh read of context offset `0x10` at candidate offset `0x04`.
3. Candidate address plus 24 bytes at candidate offset `0x08`.

There is no explicit candidate-null check before these writes. Valid writable candidate storage is a precondition for reaching them safely. The fresh size-field read need not equal the value used for the acquisition request.

The function then calls F-468828a44d0c with the candidate and stores its result into the temporary slot.

If that stored result is nonzero, F-ba86177221f0 is called with the candidate. After this dependency returns, the candidate value used by the function is unconditionally replaced with zero.

The earlier initialization result is not restored after cleanup. Acquisition has received the slot’s address, so a retained alias may allow cleanup to change it. Whether this occurs is unknown.

### Acquisition-path completion

After the acquisition alternatives converge, the function:

1. Writes the current candidate value to record offset `0x04`.
2. Reads a designated 32-bit global.
3. Reads the temporary slot as the pending result.
4. Tests the slot for zero to decide whether to update the context cache.

On the zero-status branch, it writes the candidate to context offset `0x74`, then the previously read global value to context offset `0x78`.

On the nonzero-status branch, selection is skipped and the pending nonzero result is returned.

The global read therefore occurs on acquisition paths with either zero or nonzero status. It occurs after the record write and before the conditional context writes.

A cleanup call following initialization failure may leave the original nonzero status, replace it with another nonzero value, or replace it with zero. In the last case, the candidate has nevertheless been reset to zero: the function writes zero into the record, writes zero into the context cache, writes the companion global value, and proceeds to selection.

### Operation-specific selection

Selection is reached either by skipping acquisition or by obtaining a zero pending acquisition-path result.

F-ec82dd9bf147 receives:

- The current value read from record offset `0x04`.
- The original selector.
- The temporary slot’s address.

Its direct result controls continuation. A nonzero result is returned without a direct write to record offset `0x80`.

If the result is zero, the function reads the temporary slot as an object address and writes it to record offset `0x80`.

No null or validity check follows. A zero selection result does not establish that the dependency wrote the slot. If it leaves the slot unchanged, the function uses the incoming contents, including a prior status value or otherwise uninitialized contents.

### Descriptor installation

The object obtained from the temporary slot supplies its first 4-byte word as a dispatch-table address. The function tests the 4-byte entry at table offset `0x10`.

- If the entry is zero, F-unknown-02 receives an address four bytes beyond the selected object, the descriptor address, and 28.
- If the entry is nonzero, F-unknown-03 is invoked through that entry with the selected-object address and descriptor address.

These calls suggest alternative descriptor-installation mechanisms. No byte-copy, initialization, ownership-transfer, or descriptor-preservation guarantee is established. Neither alternative contributes a checked status to the next stage.

The slot is not converted into a stable snapshot for all later dispatch. Selection has received its address, and subsequent dependencies may affect it through retained aliases.

### Execution and finalization

After installation returns, the function reads the temporary slot again as an object address. It reads that object’s first word as a dispatch-table address, then resolves the entry at table offset `0x18`.

Thus execution may use:

- A different object if the slot changed.
- A different table if the object’s first word changed.
- A different callback if the table entry changed.

The earlier write to record offset `0x80` does not determine this lookup.

F-unknown-04 is invoked through the execution entry with record address plus `0x0C`. The selected object is not an explicit argument. No explicit null-entry check is shown for this execution callback.

If execution returns nonzero, that result is returned and finalization is skipped.

If execution returns zero, F-da8f18e4b818 receives, in argument order:

1. Operation-record address.
2. Descriptor address.
3. Original selector.
4. Transformed 32-bit flags.
5. Zero.

The transformation clears original bit 2 and forces bits 0 and 11 to one. Every other bit retains its original value.

The finalization result is returned unchanged, without a zero/nonzero test. Its success or failure semantics are not established.

## Side effects

Direct writes may affect the descriptor, operation record, candidate header, temporary slot, and context cache fields.

There is no explicit general rollback:

- Preparation writes precede acquisition and selection outcomes.
- Context cache writes precede selection and execution.
- A later nonzero result does not trigger direct restoration of earlier field values.
- Selection failure causes no direct write to record offset `0x80`.
- Returning cleanup after initialization failure causes a zero candidate value to be written to record offset `0x04`, regardless of the slot’s subsequent status.

These statements describe this function’s direct actions, not preservation guarantees. Dependencies may change records, context, globals, descriptor contents, objects, tables, or exposed temporary storage.

No allocation, release, reference-count, or ownership-transfer contract is established. F-ba86177221f0 is a cleanup-path dependency, not a proven deallocator. Invocation-local addresses are exposed, but safe retention or use after the invocation ends is not established.

No disjointness, atomicity, locking, or concurrency guarantee is supplied. Aliasing may make an earlier direct write affect a later read.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-1cc54a4c66f8 | Receives record field `0x08`; result ignored. |
| F-3ce007b692cc | Receives context field `0x64`, requested size, and slot address; returns the candidate address. |
| F-468828a44d0c | Receives candidate; result is stored in the slot and tested. |
| F-ba86177221f0 | Receives candidate after nonzero initialization result; result ignored; slot preservation unknown. |
| F-ec82dd9bf147 | Receives current record field `0x04`, selector, and slot address; direct result controls continuation. |
| F-da8f18e4b818 | Receives record, descriptor, selector, transformed flags, and zero; result returned unchanged. |
| F-unknown-01 | Receives descriptor address and 28 before explicit descriptor assignments. |
| F-unknown-02 | Receives selected-object address plus four bytes, descriptor address, and 28. |
| F-unknown-03 | Optional indirect installation dependency at table offset `0x10`. |
| F-unknown-04 | Indirect execution dependency at table offset `0x18`; result controls finalization. |

The unknown IDs are document-local opaque identifiers. No dependency is assumed to preserve globals or to own, allocate, copy, or release memory.

## Acceptance examples

These are conditional static examples, not runtime results. Each assumes valid storage and dependency behavior sufficient to reach the stated accesses.

1. **Missing required object.** Context field `0x58` is zero. The result is 6. No listed dependency is called and no direct record or context write occurs.

2. **Cached path after preparation.** Record field `0x04` is nonzero after F-1cc54a4c66f8 returns. Acquisition is skipped. Selection receives the slot without prior explicit initialization by this function, even if the original context cache differed before that dependency ran.

3. **Negative acquisition size.** The signed size field is −25 when read for acquisition. Adding 24 is representable and yields −1. That value is passed to acquisition without a local rejection branch. The dependency’s handling of it remains unknown.

4. **Zero acquisition size.** The signed size field is −24. The requested size is zero and is passed without a local rejection branch. This does not establish that acquisition accepts it or returns usable storage.

5. **Acquisition reports failure.** Acquisition returns candidate address A and leaves status 9. With no intervening slot changes, the record receives A, the global is read, and 9 is returned. Candidate-header and context-cache writes are skipped.

6. **Initialization failure and cleanup.** Acquisition reports zero and initialization returns 7. Cleanup receives the candidate; the candidate value is then reset to zero:
   - Slot remains 7: the record receives zero and the function returns 7.
   - Slot becomes 12: the record receives zero and the function returns 12.
   - Slot becomes zero: the record and context cache receive zero, the companion context field receives the global value, and selection is attempted.

7. **Selection does not define its output.** The incoming slot contains zero and selection returns zero without changing it. The function writes zero to record offset `0x80` and attempts dispatch through zero. A zero selection status alone does not ensure safe continuation.

8. **Installation changes later dispatch.** Selection supplies object A, which is stored in the record. Installation changes the slot to valid object B. Execution resolves through B, not the stored record value. Changes only to A’s table pointer or its execution entry likewise affect the later lookup.

9. **Flags.** Original flags `0x000A0004` produce descriptor byte `0x0A` and finalization flags `0x000A0801`. Dependencies may subsequently change descriptor contents.

10. **Execution result.** Execution returns nonzero value 13. The function returns 13 and does not invoke finalization.

11. **Unchecked finalization result.** Execution returns zero and finalization returns value R. The function returns R unchanged. In particular, R equal to zero produces a zero function result; no operational success meaning follows from this function alone.

## Uncertainties

The following remain **documented_only**:

- Exact calling conventions, alignment requirements, and dependency-accessed extents.
- Status meanings beyond observed branching and the fixed result 6.
- Finalization’s success or failure semantics.
- Incoming-slot requirements and whether dependencies define outputs on every path.
- Helper initialization or copying behavior.
- Allocation, ownership, release, and reference-count contracts.
- Dependency mutations through globals, shared storage, or retained aliases.
- Valid selector values and dependency-supported size ranges.
- Address-arithmetic validity, concurrency, and reentrancy guarantees.

The requested-size addition uses a signed 32-bit field plus 24. Representable results, including negative values, are passed without a local check. Overflow behavior is unestablished; neither wrapping nor rejection is established. Candidate-plus-24 address validity is also unchecked.

## Evidence

Established static evidence is the supplied function body interpreted with the supplied opaque dependency mapping. It supports the stated access widths, offsets, explicit assignments, call arguments, branch conditions, reused slot, repeated dispatch lookup, flag transformations, and result propagation.

Construction, caching, initialization, installation, and cleanup are inferred roles, not recovered dependency contracts. Retained-alias examples identify outcomes not excluded by the supplied evidence; they do not assert that such mutations occur.

No dependency implementations, callback bodies, helper definitions, or global definition were supplied. No runtime tests have been performed.
