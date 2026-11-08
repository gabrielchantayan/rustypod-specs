<!-- Trusted provenance: F-f68f8fcdbc7a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f68f8fcdbc7a

## Purpose

F-f68f8fcdbc7a performs a mode-selected sequence of indirect calls using a supplied object. It returns a 32-bit success or failure indicator and, on one successful path, explicitly replaces the first 32-bit word of a supplied record.

The established behavior consists of an initial status call followed, when permitted, by one of two sequences:

- Mode 2: two calls with numeric arguments 3 and 2, followed by construction and storage of a 24-bit value.
- Modes 3 and 4: identical sequences with numeric argument pairs 4/1, 4/1, and 2/2. The middle call receives the record address.

The apparent intent is mode-dependent processing of serialized data. The available evidence does not establish transfer direction, unit size, selector meanings, or the concrete identities of the indirect operations. This contract therefore describes their arguments and required results without assigning them read, write, or decoding semantics.

## Interface

### Arguments

| Position | Established role |
|---|---|
| First | Address of a record. Its second 32-bit word supplies the mode; its first 32-bit word is explicitly assigned on successful completion of mode 2. The record address is also passed to an indirect call in modes 3 and 4. |
| Second | Address of an object whose first 32-bit field supplies a method-table address. Entries at byte offsets 0x14 and 0x10 are invoked indirectly. Every invocation receives this object address as its first argument. |
| Third | A 32-bit value with no explicit use in the supplied body. |
| Fourth | A 32-bit value with no explicit use in the supplied body. |
| Fifth | An unsigned 32-bit value assigned to temporary storage before the first mode-specific call and used in a continuation check. |

The third and fourth arguments do not participate in the visible branch conditions, indirect-call arguments, or output calculation. Their wider interface purpose is unknown.

### Record layout

Offsets are measured in bytes from the first argument.

| Offset | Width | Established use |
|---|---:|---|
| 0 | 32 bits | Explicit output word on successful mode 2. In modes 3 and 4, its address, also the record address, is supplied to the second mode-specific call. |
| 4 | 32 bits | Mode-bearing word. Its numeric low eight bits determine the selected path. |

Passing the record address does not establish whether the dependency reads it, writes it, or accesses a larger region. In particular, the numeric argument 4 does not independently establish a four-byte access boundary.

Recognized mode values are 2, 3, and 4. Upper bits of the mode-bearing word do not affect selection. Character signedness does not affect recognition of these three values.

The low-eight-bit description concerns the numeric word value. It does not establish which physical byte contains those bits on the target machine.

### Return value

The function returns a 32-bit value:

- **0:** Every visible condition on the selected successful path was satisfied.
- **1:** The initial status was nonzero, the mode was unsupported, a continuation gate failed, or an indirect-call result differed from its required value.

There are no distinct public error codes for individual failure stages.

### Preconditions

For an execution to remain within the ordinary return-value contract:

- The object must support each reached method-table lookup and invocation.
- After a zero initial status, the record’s second word must be readable.
- On the successful mode-2 path, the record’s first word must be writable.
- Memory supplied to indirect calls must satisfy those callees’ actual requirements, which are not available here.
- Any object or table mutation performed by a dependency must leave later reached lookups valid.

No explicit null, alignment, size, or callable-entry validation appears. Invalid memory or an incompatible callee need not produce return value one; faults and nonlocal exits are outside the established normal-return behavior.

## Behavior

### Initial status and selection

Before invoking a dependency, the function initializes a 32-bit temporary word to zero.

It resolves F-unknown-001 through the object’s method-table entry at byte offset 0x14 and invokes it with the object address as its sole visible argument.

After that invocation, the function captures the temporary word’s value for possible use by modes 3 and 4. The explicit initialization establishes zero; no address of that temporary is explicitly passed to F-unknown-001. Preservation against any otherwise accessible alias is not independently guaranteed.

If the status result is nonzero, the function returns one. There is no explicit mode read or mode-specific call on this path.

If the status result is zero, the function reads the record’s second 32-bit word and selects a path using its low eight bits:

- 2 selects mode 2.
- 3 selects mode 3.
- 4 selects mode 4.
- Any other value produces return value one without a mode-specific call.

The status invocation precedes the mode read. A dependency mutation affecting the mode-bearing word before that read can therefore affect selection. Once selected, the mode is not explicitly reread.

### Mode 2

The function assigns the fifth argument to the 32-bit temporary word.

It then resolves F-unknown-002 and invokes it with:

| Argument | Value |
|---|---|
| First | Object address |
| Second | An address in temporary stack storage |
| Third | 3 |
| Fourth | 2 |

The relationship between this first temporary-storage address and the initialized 32-bit temporary is unresolved.

The first result must equal exactly 3. Any other result returns one without a second mode-specific call or a subsequent explicit record assignment.

After an exact first result of 3, the fifth argument must equal zero. A nonzero fifth argument returns one. The first invocation has already occurred, so this gate does not prevent its side effects.

If the gate passes, the function resolves F-unknown-002 again and invokes it with:

| Argument | Value |
|---|---|
| First | Object address |
| Second | Address of the 32-bit temporary word |
| Third | 3 |
| Fourth | 2 |

The second result must also equal exactly 3. Otherwise, the function returns one without explicitly assigning the record.

Following an exact second result, the function reads the current temporary word and constructs the record output as follows:

- Temporary bits 0–7 become output bits 16–23.
- Temporary bits 8–15 become output bits 8–15.
- Temporary bits 16–23 become output bits 0–7.
- Temporary bits 24–31 do not contribute.
- Output bits 24–31 are zero.

The resulting value lies between zero and 0x00FFFFFF, inclusive. The operations use the unsigned 32-bit temporary; no sign extension contributes to the result.

The function writes this value to the record’s first 32-bit word and returns zero.

This is a numeric bit-position contract. Calling it a particular wire-endian conversion would require additional evidence about memory representation and dependency behavior.

### Modes 3 and 4

Modes 3 and 4 have identical visible behavior.

The function assigns the fifth argument to the 32-bit temporary word, then resolves F-unknown-002 and invokes it with:

| Argument | Value |
|---|---|
| First | Object address |
| Second | An address in temporary stack storage |
| Third | 4 |
| Fourth | 1 |

Continuation requires both:

1. The result equals exactly 4.
2. The fifth argument equals the temporary value captured immediately after the initial status call.

With the explicitly established zero initialization and no intervening modification of that temporary before capture, the second requirement is a zero fifth argument. The source does not establish a comparison against data supplied by this first mode-specific call: the compared value was captured earlier.

If either condition fails, the function returns one without further mode-specific calls.

If both conditions pass, it resolves F-unknown-002 again and invokes it with:

| Argument | Value |
|---|---|
| First | Object address |
| Second | Record address |
| Third | 4 |
| Fourth | 1 |

An exact result of 4 is required. Any other result returns one without the final invocation.

After an exact second result of 4, it resolves F-unknown-002 once more and invokes it with:

| Argument | Value |
|---|---|
| First | Object address |
| Second | Address of the 32-bit temporary word |
| Third | 2 |
| Fourth | 2 |

An exact final result of 2 produces return value zero. Any other result produces return value one.

The wrapper does not explicitly inspect the final temporary contents or assign the record after the second invocation. It performs no explicit rollback. These facts do **not** establish that any record contents produced by the second invocation persist: the final dependency may modify the record through aliases or other accessible state, whether it succeeds or fails.

### Result matching and short-circuiting

All required dependency results are exact equalities. Larger values are not accepted as sufficient, and smaller values are not accumulated.

The initial status and most invocation results are represented as 32-bit signed values; the first result in modes 3 and 4 is handled as an unsigned 32-bit value. This distinction does not change equality with the required positive constants. A negative signed result cannot satisfy any required result.

No retry occurs. Each failed condition prevents all subsequent calls on that path.

## Side effects

### Explicit access order

The observable wrapper operations are ordered as follows:

1. Initialize the temporary word.
2. Resolve and invoke the status dependency.
3. Capture the temporary word’s post-status value.
4. If status permits, read the mode-bearing record word.
5. For a supported mode, assign the fifth argument to the temporary word.
6. Perform the selected calls and intervening continuation checks.
7. Only for successful mode 2, read the temporary value and explicitly assign the first record word.

The wrapper does not explicitly read the record’s first word before assigning it in mode 2.

Each indirect invocation resolves its entry through the object’s current method-table reference. The function does not visibly cache a single transfer entry for the complete sequence. Earlier dependency effects on the object or table can therefore affect later dispatch.

### Failure is not transactional

Return value one does not imply unchanged inputs or unchanged external state.

- The status dependency can have effects before status failure.
- The first mode-specific call occurs before the fifth-argument gate.
- A dependency receiving the record address can modify accessible state even if its result is unexpected.
- The final call in modes 3 and 4 can have effects before either a zero or one return.

For mode 2, the wrapper’s own record assignment occurs only after both exact results and the zero fifth-argument gate. This does not exclude record changes made independently by dependencies.

### Ownership and lifetime

No explicit allocation, release, reference-count adjustment, or ownership-transfer operation appears in the wrapper.

Temporary stack addresses are supplied to dependencies during this invocation. The wrapper provides no durable lifetime for those addresses after return. Whether dependencies retain them, access globals, acquire resources, or release resources is unknown.

Neither dependency has an established preservation guarantee for the object, record, globals, or other accessible state.

## Dependencies

The following opaque IDs identify dynamic call roles, not recovered concrete implementations.

| Opaque function ID | Resolution | Visible arguments | Result required for continuation |
|---|---|---|---|
| F-unknown-001 | Object method-table entry at byte offset 0x14 | Object address | Zero |
| F-unknown-002 | Object method-table entry at byte offset 0x10, resolved separately for each invocation | Object address, storage address, numeric amount, numeric selector | Exactly 3, 4, or 2, depending on the stage |

“Amount” and “selector” are descriptive labels for numeric argument positions, not established semantic types. The evidence does not prove byte counts, transfer direction, byte order, complete-access behavior, or ownership rules.

Because dispatch is repeated, invocations assigned the same documentation-local ID need not reach the same concrete implementation if earlier calls alter dispatch state.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Unless specified otherwise, they assume valid memory, normal dependency returns, and no additional dependency mutations affecting the stated conditions.

1. **Status failure precedes record inspection.**  
   If F-unknown-001 returns 7, the function returns one without explicitly reading the mode or making a mode-specific call.

2. **Selection uses the post-status record value.**  
   If the status dependency changes the mode-bearing word to 0xABCDEF02 and returns zero, mode 2 is selected.

3. **Unsupported low byte.**  
   With status zero and a mode-bearing word of 0x00000105, the function returns one without invoking F-unknown-002.

4. **Mode-2 successful construction.**  
   With mode 2, fifth argument zero, and results 3 then 3, suppose the temporary word read for final construction is numerically 0xA1563412. The wrapper writes 0x00123456 to the first record word and returns zero. This condition specifies the resulting temporary value without assuming how the dependency produced it.

5. **Mode-2 nonzero gate.**  
   With mode 2 and fifth argument 9, the first invocation still occurs. If it returns 3, the function nevertheless returns one without the second invocation or a wrapper record assignment.

6. **Exact result, not minimum count.**  
   With mode 2, a first result of 4 returns one and prevents the second invocation. An oversized result is not accepted.

7. **Mode-3 success.**  
   With captured post-status value zero and fifth argument zero, results 4, 4, and 2 produce return value zero. The second invocation receives the record address. The wrapper performs no later explicit record assignment.

8. **Mode-4 equivalence.**  
   Replacing mode 3 with mode 4 in the preceding conditions leaves the call arguments, checks, and return value unchanged.

9. **Unexpected second result.**  
   In mode 3 or 4, after the first result and gate permit continuation, a second result of 3 returns one and prevents the final invocation. Any dependency effects already performed are not explicitly rolled back.

10. **Late failure does not determine final record contents.**  
    Suppose the second invocation changes the record and returns 4, then the final invocation returns 1. The wrapper returns one and performs no restoration. Whether the earlier record value persists depends on effects of the final invocation and other accessible state.

11. **Final success does not guarantee persistence either.**  
    If the final invocation changes the record through an alias and returns 2, the wrapper returns zero without restoring the second invocation’s earlier record value.

12. **First temporary contents are not directly compared.**  
    In mode 3 or 4, with captured value zero and fifth argument zero, an exact first result of 4 passes the visible gate regardless of the contents at its storage argument, provided dependency effects do not otherwise alter the relevant state.

## Uncertainties

### Established static evidence

The supplied representation establishes argument widths, explicit accesses, dispatch offsets, supported modes, call order, numeric arguments, exact-result checks, the mode-2 bit arrangement, and normal return values.

### Inferred intent

Matching numeric amounts and results suggests a completed-unit-count interface. The mode-2 construction suggests processing a three-byte quantity. Neither interpretation is required by the behavioral contract.

### documented_only unknowns

- **Temporary-storage relationship:** The extent and relationship of the first mode-specific storage address to the initialized 32-bit temporary are unresolved. Independence or identity must not be assumed.
- **Dependency semantics:** Direction, unit size, selector meanings, memory-access extent, and partial-result conventions are unavailable.
- **External effects:** Dependencies may modify accessible state. No general preservation, ownership, or resource-management contract is established.
- **Captured temporary preservation:** Zero is established by explicit initialization. Any modification through an otherwise available alias before the post-status capture is not characterized.
- **Machine representation:** The represented words and arithmetic are 32-bit. Byte order, alignment requirements, and the complete calling convention are not independently established.
- **Exceptional behavior:** Faults, exceptions, nonlocal exits, and invalid-input behavior are not specified.
- **Unused arguments:** The wider purpose of the third and fourth arguments is unknown.

## Evidence

This contract derives solely from the supplied static function representation:

- Temporary initialization precedes the status invocation.
- Status success precedes the mode read.
- The numeric low eight bits select modes 2, 3, or 4.
- Mode 2 uses two 3/2 invocations and then explicitly stores a zero-extended, rearranged 24-bit value.
- Modes 3 and 4 use equivalent 4/1, 4/1, and 2/2 sequences.
- Their middle invocation receives the record address.
- Their continuation gate uses a value captured before the first mode-specific invocation.
- All normal returns are zero or one.
- No explicit record assignment or rollback follows the record-address invocation in modes 3 and 4.

No runtime tests have been performed. The acceptance examples express conditional obligations, not execution evidence.
