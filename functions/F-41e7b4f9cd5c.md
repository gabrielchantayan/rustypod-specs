<!-- Trusted provenance: F-41e7b4f9cd5c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-41e7b4f9cd5c

## Purpose

F-41e7b4f9cd5c performs an ordered sequence of object-word replacement, dependency calls, traversal, conditional linked-storage processing, and a final dependency call.

The established behavior is the explicit access and call sequence described below. The pattern suggests object teardown or lifecycle transition, but those meanings are **inferred intent**, not an ownership or release contract.

The two leading-word writes use values read from two separate opaque global locations; equality is unknown. Likewise, copying a selected node’s first word into the head location does not establish that the node is removed from a list.

The function has no explicit status result. Normal completion requires its dependencies to return normally, the traversal dependency eventually to return exactly `1`, and any entered head-processing loop eventually to observe a zero head.

## Interface

### Input object

The function accepts one object address. The supplied representation uses four-byte scalar words and 32-bit address-like values.

| Object byte offset | Access width | Established use |
|---|---:|---|
| `0x00` | 32 bits | Written twice using values read from two separate opaque global locations; equality is unknown. |
| `0x94` | Address only | Its address is passed to F-1b9e6fff9bc9. No direct read of this object word is shown. |
| `0xB0` | 32 bits | Read separately for two consecutive dependency calls. |
| `0xB4` | 32 bits | Read separately for two consecutive dependency calls. |

The input must support these accesses and any additional accesses required by the dependencies. There is no visible null check, bounds check, alignment check, or object-type validation.

The two later object fields need not contain different values. The caller establishes neither independent allocations nor independent ownership for them.

### External state

Four separate opaque global locations supply:

- The value for the first leading-word write.
- An address-like base for a zero write at byte offset `4`.
- The value for the second leading-word write.
- A context value used at two different callsites.

The global locations are separate; their stored values may be equal or may identify overlapping storage. Their initialization and mutability are unknown. The context is accessed separately at its two uses, rather than captured once for the whole invocation.

### Local storage

The function uses:

- A two-slot address array, with only its first slot explicitly initialized.
- A 32-bit seed temporary.
- A three-word traversal state, occupying twelve bytes.
- A separate 32-bit traversal-output temporary.
- Two 32-bit argument-copy temporaries.
- A four-byte result buffer shared by two later calls.

The supplied representation treats the address-array slots as 32-bit slots. The second slot has no explicit initialization. The seed, traversal output, and result buffer also lack explicit initialization before their first dependency use.

Several dependencies receive addresses of this invocation-local storage. Retention, access extent, and any lifetime requirements are **documented_only**. No safe use of retained local addresses after return is established.

### Result and address arithmetic

There is no explicit return value.

Let **R** denote the 32-bit result supplied by F-1b9e6fff9bc9 and subsequently used in address calculations. The relevant byte-relative locations are:

| Expression | Role |
|---|---|
| **R − `0x1C`** | Head-word location |
| **R − `0x0C`** | Conditional-field location |
| **R − `0x94`** | Final-call argument |

These are byte displacements, not counts of words. The supplied representation does not establish valid behavior for address overflow, underflow, invalid address conversion, or inaccessible derived storage. It does not justify assigning portable wraparound semantics to the calculations.

## Behavior

### 1. Initial object and external-state operations

The function reads the first replacement global and writes its value to the object’s leading 32-bit word.

It then performs four dependency calls in this order:

1. F-936a8728139d receives a read of the word at object offset `0xB0`.
2. F-a3740f6ffd69 receives a new read of that same word.
3. F-936a8728139d receives a read of the word at object offset `0xB4`.
4. F-a3740f6ffd69 receives a new read of that same word.

Each argument is obtained independently at its callsite. If the first dependency changes the field at `0xB0`, the second dependency receives the changed value. Changes made by earlier dependencies may also affect either later read at `0xB4`.

The caller does not explicitly clear either field.

After all four calls return, the function reads the external base value and writes a 32-bit zero at that base plus four bytes. It then reads the second replacement global and writes that value to the object’s leading word.

The two replacement globals may contain equal values. Two ordered writes still occur; no inequality test or conditional suppression is shown.

Aliasing can make the order significant. For example, the external zero-write destination may overlap an object field, and the zero write may affect state later read for the second leading-word assignment. Independence of these locations is not established.

### 2. Seed setup and traversal

The function stores the original input object address in the first slot of the two-slot address array. It does not initialize the second slot.

F-f35532238f81 receives, in argument order:

1. The address of the seed temporary.
2. The context value read at this callsite.
3. The address of the two-slot array.

The caller does not establish whether the dependency reads the second array slot or which locations it initializes.

After the call returns, the function initializes the traversal state in this write order:

1. Third word: zero.
2. Second word: zero.
3. First word: the current seed value.

The resulting three-word layout is therefore the current seed followed by two zero words.

The traversal repeatedly invokes F-a1ecaae6738a with the traversal-state address and the separate traversal-output address. Its 32-bit result controls continuation:

- Exactly `1` terminates the traversal.
- Every other result causes F-dcbd5e1e578f to be called with the traversal-state address, followed by another traversal attempt.

Zero, negative results, and positive results other than `1` all continue. No distinct error handling, retry bound, timeout, or alternate exit is visible.

Neither the caller nor this contract guarantees that the output temporary is initialized when the terminal result occurs. A meaningful subsequent read depends on the dependency’s external contract.

### 3. Post-traversal operation

After the terminating traversal call, the function:

1. Copies the current traversal output into one 32-bit argument temporary.
2. Copies the current seed into another 32-bit argument temporary.

The seed is read again here. It is not necessarily equal to the value originally copied into the traversal state: dependencies may have changed it through an address made available earlier.

F-7fa3e55b7211 then receives:

1. The address of the four-byte result buffer.
2. A new read of the external context.
3. The address of the seed copy.
4. The address of the traversal-output copy.

The two context arguments used during the invocation are not guaranteed equal. Intervening dependencies may modify the context’s global storage.

The caller does not directly inspect the result buffer. Its storage is passed again later if conditional processing is entered.

### 4. Conditional linked-storage setup

F-1b9e6fff9bc9 receives the input object address plus `0x94`. Its returned value becomes **R**, which is retained for the remaining derived-address operations.

The function derives the head location at **R − `0x1C`** and reads the 32-bit conditional field at **R − `0x0C`**.

If that field is zero, the entire conditional block is skipped. This skips both conditional setup calls and the head-processing loop, regardless of what the head location contains. The final call remains reachable.

If the field is nonzero, the function performs these operations:

1. Reads the conditional field again as an address-like value.
2. Reads a 32-bit word eight bytes beyond that value and stores it in the temporary previously used for the seed copy.
3. Reads the conditional field again and stores it in the temporary previously used for the traversal-output copy.
4. Calls F-d0acc006efef with the result-buffer address, head-location address, address of the first copied value, and address of the second copied value.
5. Reads the conditional field again after that call.
6. Calls F-88d9235286f8 with the head-location address, the freshly read field value, and scalar zero.

The initial nonzero test is not a retained snapshot. The later field reads are separate accesses. In particular, F-d0acc006efef may change the value subsequently passed to F-88d9235286f8.

The same result-buffer storage and the same two argument-copy locations are reused across the post-traversal and conditional operations. They are not independent fresh buffers. If a dependency retains an address to one of those locations, later caller writes may change the contents reachable through that retained address; retention itself remains unspecified.

### 5. Head-processing loop

This loop runs only after the conditional block has been entered and F-88d9235286f8 has returned.

Each iteration reads the 32-bit head word at **R − `0x1C`**. Zero ends the loop. A nonzero value selects an address-like node with the following accessed fields:

| Node byte offset | Width | Use |
|---|---:|---|
| `0x00` | 32 bits | Copied into the head location |
| `0x04` | 32 bits | Second argument of the first node call |
| `0x08` | 32 bits | First argument of the first node call |

For each selected node:

1. Its offset-zero word is read and written to the head location.
2. F-ff1bdc376dcd receives the node’s offset-`0x08` word, its offset-`0x04` word, and zero.
3. F-ff1bdc376dcd receives the selected node address, `1`, and zero.
4. The head location is read again for the next iteration.

The head write occurs before either dependency call. This establishes only a copy of the selected node’s first word into the head location. It does **not** establish removal, advancement, or any structural list invariant. A self-referential node can leave the head unchanged.

The relative evaluation order of the two node-field reads used for the first call is not established. Both supply that call’s arguments.

The selected node address is retained for the second call even if the first call changes the head. The next iteration instead uses a fresh head read, so either dependency can affect continuation by modifying that location.

### 6. Final operation

After the conditional block is skipped or its head loop terminates, F-57d76f0801f8 receives **R − `0x94`**.

There is no visible comparison between that derived value and the original input object address. Their equality is unknown.

If the final dependency returns normally, the function returns without an explicit result.

## Side effects

Directly established writes comprise:

- Two ordered 32-bit writes to the object’s leading word.
- One 32-bit zero write through the external base plus four bytes.
- Initialization and reuse of invocation-local storage.
- One head-word write for each selected nonzero node.

The caller makes no direct stores to the object fields at `0xB0` or `0xB4`, to the conditional field, or to node fields. Dependencies or aliasing may nevertheless change those locations.

Dependency side effects are not bounded by the supplied body. Dependencies may modify globals, alter object or traversal state, retain addresses, invalidate memory, or change the data controlling later loops.

Ownership and release semantics are **documented_only**:

- Neither initial field-processing dependency is established to release a resource.
- The two calls per selected node are not established to free their arguments.
- Writing the head is not established to remove the selected node.
- The final call is not established to destroy or release the input object.
- No passed addresses are guaranteed to identify separate allocations.

If any dependency invalidates memory, validity of subsequent accesses requires an external contract not supplied here. No rollback, recovery branch, or exception handling is visible.

## Dependencies

| Opaque function ID | Established callsite role |
|---|---|
| F-936a8728139d | First operation on each of two independently read object fields |
| F-a3740f6ffd69 | Second operation on each field, using a fresh read |
| F-f35532238f81 | Receives seed storage, context, and the partially initialized address array |
| F-a1ecaae6738a | Receives traversal state and output storage; exactly `1` terminates traversal |
| F-dcbd5e1e578f | Receives traversal state after each nonterminal traversal result |
| F-7fa3e55b7211 | Receives the result buffer, context, and two post-traversal copies |
| F-1b9e6fff9bc9 | Receives object address plus `0x94`; supplies **R** |
| F-d0acc006efef | Receives the reused result buffer, head address, and conditional copies |
| F-88d9235286f8 | Receives the head address, a fresh conditional-field value, and zero |
| F-ff1bdc376dcd | Called twice for each selected node with the argument forms described above |
| F-57d76f0801f8 | Final operation on **R − `0x94`** |

These are caller-visible roles, not independent dependency contracts.

## Acceptance examples

The following are conditional acceptance criteria from static evidence, not executed tests. Unless otherwise stated, dependencies return normally and all accessed storage remains valid.

1. **Equal replacement values.** If both replacement globals contain the same word, that word is written at both specified stages. Equality does not suppress either write or the intervening operations.

2. **Fresh object-field read.** If F-936a8728139d changes the field at `0xB0` from A to B, the following F-a3740f6ffd69 receives B.

3. **Immediate traversal completion.** If the first F-a1ecaae6738a result is `1`, F-dcbd5e1e578f is not called. The current traversal output is then copied for the follow-on call.

4. **Exact sentinel.** Results `0`, `2`, and then `1` cause exactly two F-dcbd5e1e578f calls, one after each nonterminal result.

5. **Changed context.** If intervening calls change the context global, F-7fa3e55b7211 receives the value read at its own callsite, not the earlier context snapshot.

6. **Zero conditional field.** If the initial read at **R − `0x0C`** is zero, neither conditional setup dependency nor either per-node call is reached. A nonzero head alone does not override this gate. The final call receives **R − `0x94`**.

7. **Conditional-field mutation.** If F-d0acc006efef changes the conditional field to Q, the subsequent F-88d9235286f8 receives Q as its second argument.

8. **One node followed by zero.** If the head selects a node whose offset-zero word is zero, the caller writes zero to the head before the two node calls. If neither call changes the head, the next loop check terminates.

9. **Self-referential node.** If a selected node’s offset-zero word equals its own address, the head write leaves the head selecting that node. If both node calls preserve the head and relevant node contents, the next iteration selects it again. Removal and termination must not be inferred.

10. **Dependency changes the head.** If the first node call changes the head to another node, the second call still receives the originally selected node address. The next iteration selects from the changed head.

11. **Nontermination.** If traversal never returns `1`, no post-traversal, conditional, or final operation is reached. An entered head loop that never observes zero likewise prevents the final call.

## Uncertainties

The following remain **documented_only**:

- Semantic types and lifecycle meaning of the object, globals, traversal state, and node fields.
- Initialization guarantees for dependency-written storage.
- Whether the second address-array slot is read.
- Dependency access extents, address retention, ownership, and release behavior.
- Whether the replacement values differ.
- Whether a head write actually removes a node or advances through a valid list.
- Whether **R − `0x94`** identifies the original object or another region.
- Failure conventions and abnormal control-flow behavior of dependencies.
- Address-arithmetic behavior at representation boundaries.
- Endianness, alignment requirements, synchronization, and concurrent-access semantics.
- Whether dependency effects preserve storage required by later operations.

No idempotence, reentrancy, thread-safety, or repeated-invocation guarantee is established.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping. Established claims concern explicit widths, offsets, reads, writes, storage reuse, call arguments, ordering, branch conditions, and loop conditions.

Teardown, lifecycle transition, removal, and release are not established semantics. Where useful as interpretations, they remain inferred intent rather than acceptance requirements.

No runtime tests have been performed.
