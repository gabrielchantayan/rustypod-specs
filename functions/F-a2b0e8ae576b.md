<!-- Trusted provenance: F-a2b0e8ae576b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a2b0e8ae576b

## Purpose

F-a2b0e8ae576b conditionally retrieves a three-word record, passes its components to three virtual operations on an object, optionally refreshes a cached lookup result, and conditionally performs further operations using temporary storage.

Its return value is branch-dependent: null, the initially associated pointer, a secondary lookup result, or the object argument. A nonnull result is not a uniform success indicator, and null does not uniquely identify an error.

The established behavior consists of memory accesses, dependency calls, comparisons, and return selection. Interpreting the record as object state or the temporary value as a region is inferred intent, not a required semantic meaning.

## Interface

### Arguments and result

The supplied representation uses 32-bit object words and represents pointers in a 32-bit address model.

| Item | Representation | Established role |
|---|---|---|
| Object argument | Object pointer | Supplies object-relative state and virtual dispatch information. Additional state is read from global locations. |
| Second argument | Opaque 32-bit value | Unused by the supplied body. |
| Third argument | Represented 32-bit integer | Its incoming value has no effect on the visible behavior. |
| Result | Pointer in the represented address model | Selected according to the branches described below. |

When the associated pointer is nonnull, the third argument’s working value is replaced with the processing gate from the object. When the associated pointer is null, the early return does not depend on the incoming third argument.

There is no object-null check. The object must support every access reached on the selected path.

### Object layout

Offsets are byte offsets from the object base.

| Offset | Access | Established use |
|---|---|---|
| `0x00` | Read for virtual dispatch | Dispatch-table pointer. |
| `0x34` | Conditional word read | Related address captured before temporary-storage initialization. |
| `0x38` | Entry read; optional later read | Associated pointer supplied to lookups. |
| `0x44` | Conditional word read | Enables processing when nonzero. |
| `0x80` | Address formation | Embedded address passed twice to a helper. Its storage extent is unknown. |
| `0xA0` | Conditional low-byte test of the represented word | Suppresses the late helper sequence when the low byte is nonzero. |
| `0xAC` | Word read | First saved comparison operand. |
| `0xB0` | Word read | Second saved comparison operand. |
| `0xB4` | Conditional 32-bit write | Receives the secondary lookup result. |
| `0xB8` | Word read after virtual updates | Enables the secondary lookup and cache assignment. |

The low-byte test at `0xA0` establishes which bits affect the branch. It does not independently establish the physical load width used by an implementation.

### Global state

The following locations are separate from object storage:

| Global location | Established access |
|---|---|
| `0x0820F284` | Supplies the primary lookup’s selector operand when that lookup is reached. |
| `0x0820F288` | Supplies the optional secondary lookup’s selector operand after the three virtual updates. |
| `0x0826EC98` | Holds a pointer that is read only after the first two late gates pass; the pointed-to byte is then read. |

The selector locations are distinct storage locations. Their contents are not established as immutable, unequal, or semantically different. The supplied representation does not independently establish their declared types or access widths.

The late-gate pointer is obtained from its global location at the time that condition is evaluated. It is not captured at entry. It is dereferenced without a null check.

### Virtual interface and record layout

Virtual slots are identified by byte offsets in the dispatch table:

| Slot | Established call shape |
|---|---|
| `0x114` | Object argument; returns a represented 32-bit comparison value. |
| `0x11C` | Object argument and record word 1. |
| `0x120` | Object argument and record word 2. |
| `0x118` | Object argument and record word 0. |

A successful primary lookup must provide storage supporting three consecutive 32-bit reads at record offsets `0`, `4`, and `8` when those reads occur. A nonnull lookup result alone does not establish that validity.

## Behavior

### 1. Entry gate

The function first reads the associated pointer at object offset `0x38`.

- If it is null, the function returns null. The processing gate at `0x44` is not needed.
- If it is nonnull, the function reads the word at `0x44`.
- If that word is zero, the function returns the associated pointer captured at entry.
- Otherwise, the primary lookup is reached.

Neither unused incoming argument can override this gate.

### 2. Primary lookup

F-5710a8628cba receives:

1. The associated pointer captured at entry.
2. The selector operand read from global location `0x0820F284`.

This is a global read, not an established constant selector.

If the dependency returns null, the function returns null immediately. It does not perform the saved-state reads, virtual operations, secondary cache assignment, or late helper sequence. Effects already performed by the lookup are not undone.

### 3. Saved operands and virtual updates

After a nonnull primary result, the following order is established:

1. Read and save the object word at `0xAC`.
2. Read and save the object word at `0xB0`.
3. Invoke virtual slot `0x114` and save its result.
4. Read record word 0.
5. Read record word 1.
6. Read record word 2.
7. Invoke virtual slot `0x11C` with captured record word 1.
8. Invoke virtual slot `0x120` with captured record word 2.
9. Invoke virtual slot `0x118` with captured record word 0.

The record reads occur after the first virtual call. If that call modifies record storage, the reads observe its resulting contents, provided the storage remains valid.

All three record words are captured before the update calls begin. Subsequent changes to record storage do not change the captured arguments or later comparison operands.

The three updates occur even when all saved operands already match the record values. The function does not test equality before calling them.

Each virtual call obtains dispatch information through the object. The body does not preserve one dispatch-table pointer for all four calls. An earlier dependency may therefore affect a later target by changing dispatch state.

### 4. Optional secondary lookup and cache write

After the update calls, the tentative return value is null. The function reads the current word at `0xB8`.

If it is zero:

- No secondary lookup occurs.
- This function makes no direct write to `0xB4`.
- The tentative return value remains null.

If it is nonzero:

- The current associated pointer at `0x38` is obtained.
- The selector operand is read from global location `0x0820F288`.
- F-5710a8628cba is called with those operands.
- Its result is written to `0xB4` and retained as the tentative return value.

Both argument reads belong to this post-update stage. The supplied representation does not establish a relative evaluation order between the associated-pointer read and the selector read.

The associated pointer may differ from the entry value and is passed without a new null check. Earlier dependencies may also have changed the secondary selector before it is read. The function neither compares the two selector contents nor requires them to differ.

A null secondary result is still written to `0xB4`.

### 5. Change decision

The function compares these captured pairs:

| Saved operand | Captured record operand |
|---|---|
| Object word from `0xAC` | Record word 1 |
| Object word from `0xB0` | Record word 2 |
| Result of virtual slot `0x114` | Record word 0 |

If all pairs are equal, the function returns the tentative result: either the secondary lookup result or null.

If any pair differs, it enters the changed path. Every normal return from that path returns the object argument, irrespective of the tentative result.

This decision uses captured values, not fresh object or record reads. It does not prove that the object’s final state changed: the virtual operations’ effects are unknown.

### 6. Late gates

Only on the changed path, F-3fac671648b8 is called without explicit arguments. Further processing requires these conditions, in order:

1. Its result is nonzero.
2. The low byte of the current object word at `0xA0` is zero.
3. The byte reached through the pointer currently held at global location `0x0826EC98` is nonzero.

Evaluation short-circuits:

- A zero dependency result skips the object low-byte test and both accesses associated with the final global gate.
- A nonzero low byte skips the global-pointer read and pointed-to byte read.
- Otherwise, the function reads the global pointer and dereferences it without checking for null.

The pointer may have been changed by any preceding dependency, including F-3fac671648b8. No preservation of the pointer or its target is established.

Only the low byte of the object word affects the second condition. For example, `0x00000100` passes that condition, while `0x00000001` fails it.

### 7. Temporary-storage helper sequence

If all late gates pass:

1. Capture the current related address at object offset `0x34`.
2. Call F-e57b05520b65 with the temporary-storage address, followed by the address at object offset `0x80` twice.
3. If the captured related address is nonzero, call F-ee8733a08210 with the temporary-storage address and the captured address plus `0xD0`.
4. Call F-ffe0d92c57b7 with the object.
5. If its result is nonzero, call F-c9b5d17d8d1e with that result and the temporary-storage address.
6. After that call returns normally, call F-4d037ecf0f82 with the same captured result and temporary-storage address.

The related address is captured before F-e57b05520b65. A change to object offset `0x34` during that call does not replace the captured operand.

There is no second context lookup or validity check between the final two calls. The second receives the same context-like value even if the first call changes associated state.

The function then returns the object argument.

## Side effects

### Direct and delegated effects

The only explicit persistent assignment in the body is the conditional 32-bit write of the secondary result to object offset `0xB4`.

The body does not directly assign the three compared state values. The virtual calls receive the record values, but their actual state changes are unknown.

All dependencies may have additional effects. They are not assumed to preserve object fields, selector globals, the late-gate pointer, its pointed-to byte, dispatch state, record storage, or ownership. Consequently, “no direct write” does not mean that a field remains unchanged across a dependency call.

No rollback is shown for earlier calls or the cache assignment if a later operation fails to return normally.

### Ownership and lifetime

The body contains no explicit allocation, release, or destruction operation. This does not establish that dependencies perform none.

Ownership and retention rules for lookup results, the object, the related address, and the context-like result are unknown. In particular, the first member of the final helper pair is not known to preserve the context-like value’s validity for the second.

Temporary storage is passed by address. Its size, alignment, initialization contract, representation, and cleanup requirements are unresolved. No explicit cleanup follows the helper sequence. The function does not return its address, but dependency retention of that address is not established either way.

### Arithmetic and failures

Record components, object words, and the saved virtual result are represented as 32-bit values. Equality compares those captured values; no arithmetic is performed on the record components.

Address formation includes object-relative offsets and the addition of `0xD0` to the captured related address. The latter is reached only when that address is nonzero. No overflow, mapping, alignment, or bounds checks are shown. Exact overflow semantics cannot be established from the supplied representation alone.

Valid storage and callable targets are preconditions wherever accessed. In particular, the final global pointer must designate a readable byte when its dereference is reached.

There is no explicit exception handler, retry, error-code protocol, or recovery path. Dependency nonreturn, invalid memory, and other abnormal behavior are outside the established normal-return contract.

## Dependencies

| Opaque ID or virtual slot | Established use | Unresolved behavior |
|---|---|---|
| F-5710a8628cba | Primary and optional secondary lookup, each using a selector read from its respective global location. | Selector interpretation, result lifetime, ownership, and side effects. |
| Virtual `0x114` | Supplies a saved 32-bit comparison operand before record reads. | Meaning and effects. |
| Virtual `0x11C`, `0x120`, `0x118` | Receive captured record words 1, 2, and 0, respectively. | Update semantics and effects. |
| F-3fac671648b8 | Supplies the first late zero/nonzero gate. | Meaning, observed state, and effects. |
| F-e57b05520b65 | Receives temporary storage and the same embedded address twice. | Construction semantics, layout, and ownership. |
| F-ee8733a08210 | Receives temporary storage and a captured related address plus `0xD0`. | Transformation and lifetime effects. |
| F-ffe0d92c57b7 | Receives the object and supplies a context-like zero/nonzero result. | Result type, lifetime, and ownership. |
| F-c9b5d17d8d1e | First operation using the captured nonzero context-like result and temporary storage. | Effects and validity guarantees. |
| F-4d037ecf0f82 | Second operation using those same captured operands. | Effects and ownership consequences. |

No opaque identities are supplied for the virtual targets.

## Acceptance examples

These are conditional static obligations, not executed tests. Unless stated otherwise, memory is valid and dependencies return normally.

1. **Null association:** A null pointer at `0x38` produces a null return without reading the processing gate or either selector global.

2. **Disabled processing:** A nonnull association and zero word at `0x44` return the captured association. No lookup occurs.

3. **Primary miss:** With processing enabled, the primary lookup uses the selector read from `0x0820F284`. A null result returns null before snapshots or virtual calls.

4. **Equal selector contents:** If both selector globals contain the same value when read, each reached lookup receives that value. There is no requirement that the contents differ.

5. **Secondary selector mutation:** If an earlier dependency changes `0x0820F288`, a subsequently enabled secondary lookup uses the changed contents, not an entry-time snapshot.

6. **Equal comparisons without secondary lookup:** If all captured comparison pairs match and post-update `0xB8` is zero, all three updates still occur. The return is null, with no direct cache assignment.

7. **Equal comparisons with secondary lookup:** If all pairs match and post-update `0xB8` is nonzero, the secondary result is stored and returned, including when null.

8. **Changed comparison with first late gate false:** The function returns the object. Neither the low-byte condition nor the global-pointer gate is evaluated.

9. **Low-byte suppression:** With a nonzero first late gate and a nonzero low byte at `0xA0`, the global pointer is not read. A word of `0x00000100` does not cause this suppression.

10. **Late pointer replacement:** If preceding dependencies replace the pointer at `0x0826EC98`, the final gate reads through the replacement. A zero target byte suppresses the helper sequence. A null replacement is dereferenced if reached; no safe fallback is established.

11. **No related address:** When all late gates pass but captured `0x34` is zero, F-e57b05520b65 and F-ffe0d92c57b7 still run; F-ee8733a08210 does not.

12. **Context-like result:** A zero result from F-ffe0d92c57b7 skips both final calls. A nonzero result causes both in order, provided the first returns normally, with no intervening relookup.

13. **Mutation timing:** A change to record storage by virtual slot `0x114` precedes the record reads. Later update-call changes to record storage do not alter captured operands. Changes to `0x38` or `0xB8` during those updates can affect the secondary stage.

## Uncertainties

### Established static evidence

The supplied body establishes object offsets, represented word widths, record-read order, virtual-call order, global selector reads at their respective lookup stages, the two-stage global-pointer dereference, short-circuit conditions, cache assignment, comparison pairings, and normal-return selection.

### Inferred intent

The three-word record may describe object state. The temporary storage may hold a region-like value, and the final helper pair may propagate a change. These interpretations are not required for conformance.

### Documented-only unknowns

The following remain **documented_only**:

- Actual ABI, declared types, signedness, and selector-global access widths.
- Selector meanings, values, mutability rules, and any relationship between their contents.
- The identity and application meaning of the late-gate pointer and target byte.
- Dependency implementations, virtual-target semantics, and global-preservation guarantees.
- Ownership, retention, and lifetime of all externally supplied or returned storage.
- Temporary-storage layout, initialization, and cleanup obligations.
- Exact address-overflow behavior, synchronization, concurrency, reentrancy, and fault handling.
- Whether any return value denotes application-level success.

## Evidence

This contract derives from static inspection of the supplied function representation and opaque dependency mapping. No dependency implementations were available.

No runtime tests have been performed. The acceptance examples are conditional consequences of the visible accesses and control flow, not observed execution results.
