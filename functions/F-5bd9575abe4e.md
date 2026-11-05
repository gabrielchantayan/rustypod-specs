<!-- Trusted provenance: F-5bd9575abe4e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5bd9575abe4e

## Purpose

F-5bd9575abe4e dispatches processing to a subset of eight context-associated records. It prepares a transient three-word value, selects an address within a second object, invokes a preparation dependency, visits enabled records in ascending index order, and invokes a final dependency on the selected address.

The enabled subset is not necessarily fixed at entry: the function reads the context’s selection mask separately for each record. Changes made by dependencies can therefore affect later dispatch decisions.

The interpretation of the three-word value, the selected address, and the eight records is not established. Terms such as “vector,” “transform,” “constructor,” and “destructor” would imply semantics that the supplied evidence does not prove.

## Interface

The function has three arguments and no returned value:

| Argument | Established role |
|---|---|
| Context address | Base address for an entry flag, a selection mask, two dependency arguments, and eight indexed records. |
| Input address | Base address for a three-word input value and an address selected using the third argument. Also forwarded unchanged to each record-processing call. |
| Selector | Integer used both for a branch on equality to 1 and for address calculation. |

The recovered interface uses 32-bit integer arguments and 32-bit words. The first two arguments are used as addresses rather than as ordinary numeric quantities. No pointer validation is visible.

### Context-relative layout

All offsets are byte offsets from the context address.

| Offset | Access or use |
|---|---|
| `0x903` | One-byte flag read at entry. Any nonzero value triggers an additional dependency call. |
| `0x89C` | 32-bit selection mask, read once per loop iteration. Only bits 0 through 7 affect dispatch. |
| `0x88C` | Address passed as the second argument of the preparation dependency. |
| `0x838` | Address passed as the fourth argument of every record-processing call. |
| `0x2F8 + 0xA8 × index` | Address passed as the first argument of the record-processing call for indices 0 through 7. |

The eight record addresses have a stride of 168 bytes. The last begins at offset `0x790`. The stride establishes address spacing, not the size or layout of a record.

### Input-relative layout

| Offset or expression | Access or use |
|---|---|
| `0x30`, `0x34`, `0x38` | Three successive 32-bit words used directly when the selector equals 1. |
| `0x30` | Address passed to the alternate value-preparation dependency when the selector does not equal 1. |
| `0x3C + 0x10 × selector` | Selected address passed to preparation, record-processing, and final dependencies. |

The selector is not restricted to 0 or 1. Every value other than 1 takes the alternate value-preparation path, while its full value still participates in the selected-address calculation.

### Preconditions

For ordinary successful execution:

- Every directly accessed byte and word must be accessible with the target’s required alignment and access permissions.
- Address arithmetic must produce meaningful addresses for all accesses performed by this function and its dependencies.
- Each dependency’s own preconditions must hold.
- Stack-backed temporary storage passed to dependencies must be used consistently with its temporary lifetime.

The function does not check these preconditions. The extent of memory required by the dependencies is unknown.

## Behavior

### 1. Entry flag handling

The first established memory access is a one-byte read at context offset `0x903`.

If that byte is nonzero, F-e1c4e131e7b7 is invoked without explicit arguments. If the byte is zero, that invocation is omitted.

This flag is read only once by the visible function. No later recheck occurs. The absence of explicit arguments does not establish that F-e1c4e131e7b7 is free of state changes: it may access global or otherwise reachable state. Any changes it makes before returning precede all subsequent input preparation and selection-mask reads.

### 2. Preparing the transient three-word value

The selector determines the source of three 32-bit words:

- When the selector equals 1, the source begins at input offset `0x30`.
- Otherwise, F-b9d144e0e393 is called with a temporary output address as its first argument and input offset `0x30` as its second argument. The temporary output then becomes the source.

The function copies three words from the chosen source into a separate transient record. The source reads occur in ascending word order: offsets 0, 4, and 8. The corresponding record consists of three consecutive 32-bit words, totaling 12 bytes.

These are raw word transfers. The visible function does not interpret the words as signed integers, floating-point values, pointers, or any other semantic type.

On the alternate path, the temporary output is not visibly initialized before F-b9d144e0e393 is called. Whether that dependency writes all 12 bytes subsequently read is part of its contract and is not established here.

The record-processing dependency receives the address of the copied record, not the original input words or the alternate output buffer. This prevents later changes to the original source alone from changing the already-copied record. It does not make the copied record immutable: a dependency receiving its address may modify it.

### 3. Selecting and preparing an input-relative address

The selected address is:

input address + `0x3C` + 16 × selector.

The same resulting address is used throughout the remaining visible execution. For example, selectors 0, 1, and 2 select input offsets `0x3C`, `0x4C`, and `0x5C`, respectively.

After preparing the transient record, the function invokes F-3f35b3cd17e1 with:

1. The selected address.
2. Context address plus `0x88C`.

No selection-mask read precedes this call. Consequently, any mask change caused by this dependency can affect the first dispatch decision.

The visible function does not establish what this call reads or writes, or whether it initializes an object at either address.

### 4. Visiting the eight record slots

The function considers exactly eight indices, in order from 0 through 7. For each index:

- It reads the 32-bit word at context offset `0x89C`.
- It tests the bit corresponding to that index.
- If the bit is set, it invokes F-3df10f8356a4.
- If the bit is clear, it makes no record-processing call for that index.

Each invocation of F-3df10f8356a4 receives these five arguments, in this order:

1. Context address plus `0x2F8 + 0xA8 × index`.
2. The original input address.
3. The address of the transient three-word record.
4. Context address plus `0x838`.
5. The selected input-relative address.

The tested bits are, successively, `0x01`, `0x02`, `0x04`, `0x08`, `0x10`, `0x20`, `0x40`, and `0x80`. Bits above bit 7 have no direct effect.

The index and bit advance regardless of whether a call occurs. There is no early exit based on the number of enabled records and no revisit of an earlier index.

The mask is not snapshotted. A record-processing call can affect later dispatches if it changes the mask. Changing an already-tested bit does not cause a repeat call. Setting a future bit before its read can enable that future call; clearing it can suppress the call.

All enabled calls receive the same transient-record address. No restoration or fresh copy occurs between calls. If one call modifies that record, a later call receives the modified storage.

### 5. Final dependency invocation

After all eight indices have been considered, F-3e0c716bf179 is invoked with the selected address as its sole explicit argument.

This call occurs even when no record-processing calls occurred, provided execution reaches it normally. The function then returns without a value.

The role of this final call is unknown. Its position alone does not establish that it releases memory, destroys an object, commits state, or reverses the preparation call.

## Side effects

### Direct memory operations

The visible function directly:

- Reads the entry flag.
- Reads three source words.
- Writes temporary stack-backed storage.
- Reads the selection mask eight times during a complete normal execution.

It contains no visible direct write to context-relative or input-relative memory. Dependencies receive addresses into those objects and may write through them.

The exact effects of dependency calls must not be replaced by an assumption that they preserve globals, preserve the mask, preserve the input, or preserve any passed storage.

### Ordering

The established normal-execution ordering is:

1. Entry flag read.
2. Optional F-e1c4e131e7b7 call.
3. Optional F-b9d144e0e393 call.
4. Three-word copy into the transient record.
5. F-3f35b3cd17e1 call.
6. Eight ordered mask reads, interleaved with any enabled F-3df10f8356a4 calls.
7. F-3e0c716bf179 call.
8. Return.

This describes the order visible in the supplied representation. It does not establish atomicity, synchronization, or safe behavior under concurrent mutation.

### Ownership and lifetime

No direct allocation, deallocation, ownership transfer, or persistent pointer storage is visible.

The two temporary areas are local to this invocation. The function itself provides no post-return lifetime for pointers into them. Whether a dependency retains such a pointer is unknown; retention cannot be treated as safe without an additional lifetime contract.

Passing an address does not establish ownership transfer or permission to release it. Likewise, the final dependency cannot be assumed to release the selected object.

### Errors and abnormal completion

There is no visible error return, validation failure, retry, or recovery path. Dependency results are not visibly inspected.

If a dependency faults, does not return, or otherwise interrupts normal execution, later calls are not guaranteed to occur. In particular, no visible mechanism guarantees the final dependency call after an earlier abnormal exit.

## Dependencies

| Opaque ID | Invocation contract visible here |
|---|---|
| F-e1c4e131e7b7 | Called once without explicit arguments if the entry flag is nonzero. Internal state effects and failure behavior are unknown. |
| F-b9d144e0e393 | Called once only when the selector differs from 1. Receives a temporary destination and the address at input offset `0x30`. The destination’s first 12 bytes are subsequently read. |
| F-3f35b3cd17e1 | Called once after transient-value preparation. Receives the selected address and context offset `0x88C`. |
| F-3df10f8356a4 | Called zero through eight times, in ascending slot order, according to separately read mask bits. Receives the five addresses described above. |
| F-3e0c716bf179 | Called once after all eight mask checks on normal execution. Receives the selected address. |

No dependency’s return value contributes to a visible decision or output. This does not establish the dependencies’ declared return types.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not reports of executed tests. They assume valid addresses and normally returning dependencies.

### Direct-source path with no enabled records

Given selector 1, a zero entry flag, and a mask that remains zero:

- Neither F-e1c4e131e7b7 nor F-b9d144e0e393 is called.
- The words at input offsets `0x30`, `0x34`, and `0x38` are copied into the transient record.
- The selected address is input plus `0x4C`.
- F-3f35b3cd17e1 is called before any mask check.
- Eight mask checks occur, with no F-3df10f8356a4 calls.
- F-3e0c716bf179 is called with input plus `0x4C`.

### Sparse enabled mask

Given a mask that remains `0x85` throughout dispatch, record-processing calls occur for indices 0, 2, and 7, in that order.

Their first arguments are context offsets `0x2F8`, `0x448`, and `0x790`. Their other four arguments use the same input address, transient-record address, context offset `0x838`, and selected address.

### High bits only

Given a mask that remains `0xFFFFFF00`, none of the eight record-processing calls occurs. Preparation and the final dependency invocation still occur.

### Alternate preparation and unnormalized selector

Given selector 2 and an F-b9d144e0e393 invocation that supplies all three output words:

- The alternate preparation call occurs.
- Its three output words are copied into the record passed to processing.
- The selected address is input plus `0x5C`, not the address associated with selector 0 or selector 1.

### Mask mutation during processing

Given bit 0 is set when index 0 is checked, and its processing call sets bit 3 before returning:

- Index 3 is processed if bit 3 remains set when its own mask read occurs.
- Earlier indices are not revisited.
- A cleared future bit suppresses its call if it remains clear at its read.

### Transient-record mutation

If the processing call for one enabled index changes the transient record, the next enabled call receives that same storage without the function restoring its original contents.

## Uncertainties

The following are **documented_only**:

- Semantic types and meanings of the three words and addressed objects.
- Internal behavior, access widths, and memory requirements of every dependency.
- Whether alternate preparation writes all three words.
- Whether any dependency changes globals, the mask, source memory, or transient storage.
- Whether any dependency retains pointers or performs allocation or release.
- Valid selector range beyond the visible equality test.
- Target behavior for negative selectors, overflowing arithmetic, or address-space wraparound.
- Alignment requirements and fault behavior.
- Concurrency guarantees and visibility of unsynchronized external writes.
- Exception, unwinding, and nonlocal-exit behavior.

The recovered arithmetic uses 32-bit quantities. Multiplication by 16 and address addition can exceed a signed 32-bit range. The supplied representation does not establish a portable signed-overflow contract or validate wrapped addresses. Such cases must remain unspecified here rather than being assigned guessed modulo behavior.

The eight-iteration bit progression itself needs only bits 0 through 7 and does not approach a 32-bit shift boundary.

## Evidence

**Established static evidence:** The supplied function representation supports the offsets, argument ordering, equality branch, three 32-bit transfers, eight ascending mask checks, conditional dispatches, and final normal-path dependency call described above.

**Inferred intent:** The structure suggests preparation, selective per-record processing, and finalization. These are organizational descriptions, not evidence of construction, cleanup, ownership, geometry, or another application-specific meaning.

**Unknowns:** Dependency implementations and a complete target-language and ABI contract were not supplied. Their effects are not inferred from naming, argument positions, or call order.

No runtime tests have been performed. The acceptance examples are static behavioral criteria only.
