<!-- Trusted provenance: F-9fad0532dfe3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9fad0532dfe3

## Purpose

F-9fad0532dfe3 conditionally obtains 32-bit values through dynamic dispatch and performs two ordered, append-like operations for each nonzero value. A zero result from the dispatched operation ends acquisition without either append operation for that result.

An entry dependency determines whether acquisition is attempted. A nonzero low result word from F-128132bf95c0 produces the normal return value 0. A zero low result word admits processing, which returns 1 if it subsequently reaches either visible termination condition. Returning 1 does not imply that any value was appended.

The interpretation of the destination as two coordinated collections is **inferred intent**. The explicit field accesses and append-like actions are established static evidence; concrete collection types, second-region element contents, and cross-region consistency are not established.

## Interface

### Inputs

The supplied representation exposes four 32-bit arguments.

| Input | Established use |
|---|---|
| Destination address | Base address for two groups of fields used during append processing. |
| Source token | Passed to F-6e90825b1a9f on every acquisition attempt and to F-148222599059 on one second-region path. |
| Unused word | No explicit read or forwarding is visible. |
| Initial seed word | Stored in a function-local 32-bit state cell before the entry dependency executes. On admission, the cell is overwritten with zero before acquisition begins. |

The source token is opaque. It is not established to be an address, handle, identifier, or owned resource.

The initial seed does not supply the initial state argument to the dispatched operation: the admitted path explicitly replaces it with zero. No additional dependency argument channel may be inferred merely from incidental machine state.

No explicit argument validation is visible.

### Result

The normal result is a 32-bit word:

- **0:** The entry dependency’s low result word was nonzero.
- **1:** That low word was zero and processing reached a visible normal exit.

The function does not expose a count of appended values or a separate dependency-error result. A return of 0 is specifically the entry-gate outcome, not an established allocation-failure or append-failure indication.

### Destination fields

All offsets below are byte offsets from the destination address. End and boundary fields are accessed as 32-bit words.

| Offset | Established role |
|---|---|
| +4 | Descriptor address passed to the first-region helper. This function does not explicitly read the word at this offset. |
| +8 | First-region end address. |
| +12 | First-region boundary address. |
| +16 | Descriptor address passed to the second-region helper. This function does not explicitly read the word at this offset. |
| +20 | Second-region end address. |
| +24 | Second-region boundary address. |

Offset 0 is not explicitly accessed. The evidence does not establish the complete destination layout or that the descriptor words at +4 and +16 are begin pointers.

Both inline paths advance an end address by four bytes. The path selection is equality against the corresponding boundary, not an ordered comparison. An unequal pair selects the inline path even when the end is numerically beyond the boundary.

### Execution preconditions

Every executed load, store, and indirect call must be valid for the target environment. In particular:

- Destination fields must be accessible when their paths execute.
- A nonzero old first-region end must identify writable storage for a 32-bit store on the inline path.
- The object, dispatch table, and selected callable used during acquisition must be valid.
- Address calculations must have defined behavior under the target’s arithmetic and addressing rules.

These are execution requirements, not checks performed by the function.

## Behavior

### Entry decision and initialization

The function stores the initial seed in its local state cell and calls F-128132bf95c0. No explicit arguments are shown for that dependency.

The observed result is used as two 32-bit words. Its low word controls admission; its high word supplies the first auxiliary acquisition argument.

If the low word is nonzero, the function returns 0 without an explicit destination-field access or acquisition attempt. This restriction applies to the function’s own visible actions. It does not constrain memory or global-state effects inside F-128132bf95c0.

If the low word is zero, the function writes zero to the state cell and begins acquisition. This overwrite occurs after the entry dependency returns.

### Acquisition and dispatch

Each acquisition attempt calls F-6e90825b1a9f with the source token and a 32-bit auxiliary value. The first auxiliary value is the entry dependency’s high result word. Subsequent auxiliary values are selected by the preceding second-region path.

The acquisition dependency’s result is treated as an object address. The function reads a 32-bit dispatch-table address at the start of that object, then reads a callable address from byte offset +12 in that table. The selected operation receives the object address and the current state-cell value.

The dispatched operation’s 32-bit result replaces the state cell. A zero result causes immediate normal termination with result 1. Neither region’s append path executes for that terminal result.

There is no visible null check for the object, table, or callable address.

For the first dispatched call, the state argument is zero. On later attempts, it is the state value present when that argument is obtained after F-6e90825b1a9f returns. Preservation of a previously escaped state-cell address is not assumed: a dependency capable of accessing that address could affect the value before dispatch.

### First-region operation

For a nonzero acquisition result, the first region is processed before the second.

The function reads the end field at +8 and then the boundary field at +12. Equality selects F-e4b46f999bb4. That dependency receives:

- The descriptor address at destination +4.
- The end address just read.
- The address of the local state cell.

The state argument is therefore supplied by address. The dependency’s effects on that cell, destination fields, allocation, and ownership are not established.

When end and boundary differ, the function writes the old end advanced by four bytes to the end field at +8. Only after that update, if the old end was nonzero, it writes the current 32-bit state value to the old end address.

A zero old end suppresses the element store but not the end-field update. For a zero old end and nonzero boundary, the visible end value becomes 4.

The equality decision provides no capacity-ordering validation. No explicit first-region element store occurs on the helper path; any such action would belong to F-e4b46f999bb4.

### Second-region operation

After the first-region path completes normally, the function reads the second-region end at +20 and then its boundary at +24. These are fresh reads after the first-region operation. Effects of F-e4b46f999bb4 on those fields can therefore affect second-region path selection.

If end and boundary are equal, F-148222599059 receives:

- The descriptor address at destination +16.
- The end address just read.
- The source token.

The next auxiliary acquisition value is taken from an observed secondary machine result associated with this call. Its semantic meaning is not established.

If end and boundary differ, the function computes the old end advanced by four bytes and writes that value to the end field at +20.

For a nonzero old end, it then calls F-67a6a52fe86f. No explicit arguments are shown. Following that call, the next auxiliary value comes from its observed secondary machine result.

For a zero old end, the call is skipped. The advanced end value, 4, supplies the auxiliary value if another acquisition occurs.

There is no explicit second-region element store on this inline path. Neither an implicit constructor argument nor a specific element representation can be established from the supplied evidence.

### Final state test and continuation

The function tests the state cell after the second-region operation. Zero produces normal return 1; nonzero causes another acquisition attempt.

A first-region helper’s write of zero does not by itself guarantee termination. There is no state test between the first- and second-region operations. Moreover, if the helper retains or publishes the state-cell address, a subsequent dependency could modify the cell before the final test. Continuation depends on the value actually observed at that final test.

Consequently:

- If the first helper writes zero and the cell remains zero through the second-region operation, the function returns 1 without another acquisition.
- If a later effect makes the cell nonzero before the final test, another acquisition occurs.
- If the state remains nonzero after every append sequence and acquisition never returns zero, no iteration bound or other normal exit is visible.

## Side effects

### Explicit memory and call effects

The function explicitly performs some or all of the following, depending on the selected paths:

- Initializes and updates its local 32-bit state cell.
- Calls the entry and acquisition dependencies.
- Reads an object’s dispatch-table address and a callable address.
- Invokes the dynamically selected operation.
- Reads destination end and boundary fields.
- Advances destination end fields by four bytes.
- Stores a 32-bit state value at a nonzero first-region old end.
- Passes the state-cell address to F-e4b46f999bb4.
- Invokes either second-region dependency.

The established order matters: first-region end advancement precedes its inline element store; the first-region operation precedes second-region field reads; second-region inline end advancement precedes F-67a6a52fe86f; and the final state test follows the entire second-region operation.

These orderings do not establish atomicity, synchronization, rollback, or transactionality. Earlier writes may remain visible if a later access or call fails.

### Preservation and ownership

The function contains no explicit clearing pass, release, destruction, or deallocation. This does not establish that existing entries or resources remain unchanged: the entry, acquisition, dispatch, and append dependencies have unconstrained effects in the supplied evidence.

Ownership of the acquired object and the meanings of nonzero state values are unknown. The evidence does not establish whether helpers copy, retain, transfer, allocate, or release resources.

The state cell has function-local lifetime. Its address is explicitly exposed to F-e4b46f999bb4. Whether that address is retained or published, and whether another dependency accesses it while the invocation is active, are unresolved. Continued validity after the function returns is not established.

### Errors and partial progress

No explicit exception handling, allocation-failure handling, descriptor validation, rollback, or dependency-error translation is visible.

A failing second-region action may occur after first-region changes. A failure during an inline first-region store may occur after the corresponding end field has advanced. The supplied evidence does not define the resulting failure mechanism or guarantee recovery.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-128132bf95c0 | Called once at entry with no explicit arguments shown. Its low observed result word gates processing; its high word initializes the auxiliary acquisition value. |
| F-6e90825b1a9f | Called for each acquisition attempt with the source token and auxiliary value. Its result is used as an object address for dynamic dispatch. |
| F-e4b46f999bb4 | Called when first-region end equals boundary. Receives the descriptor address, old end, and state-cell address. |
| F-148222599059 | Called when second-region end equals boundary. Receives the descriptor address, old end, and source token. An observed secondary result supplies the next auxiliary value. |
| F-67a6a52fe86f | Called after second-region inline end advancement when the old end is nonzero. Explicit arguments are not shown. An observed secondary result supplies the next auxiliary value. |

The dynamically selected operation has no supplied opaque function ID. Its selection through dispatch-table offset +12, its two visible arguments, and its 32-bit result assignment are established. Its other effects are unknown.

None of these dependencies is assumed to preserve globals, destination contents, reachable state, or ownership relationships.

## Acceptance examples

These are conditional static-contract examples, not executed tests. Unless a condition says otherwise, executed accesses are valid and dependencies return normally.

1. **Entry denied.**  
   F-128132bf95c0 supplies low word 7. The function returns 0 without explicitly accessing destination fields or attempting acquisition. The entry dependency’s own side effects remain unconstrained.

2. **Immediate exhaustion.**  
   The entry low word is zero. The first dispatched operation receives state zero and returns zero. The function returns 1 without executing either append path. Its first auxiliary acquisition argument is the entry dependency’s high result word.

3. **One nonzero value through both inline paths.**  
   Acquisition produces a nonzero value. Both end/boundary pairs differ, and both old ends are nonzero. The first end advances four bytes before the state-value store at its old end. The second end then advances four bytes before F-67a6a52fe86f executes. If the final state test observes nonzero, the next acquisition uses that dependency’s observed secondary result.

4. **First helper writes zero, which remains zero.**  
   Acquisition returns nonzero and first-region end equals boundary. F-e4b46f999bb4 writes zero through the supplied state-cell address. The second-region operation still executes. If the state cell remains zero through that operation, the function returns 1 without another acquisition.

5. **Zero is replaced before the final test.**  
   Under the preceding example’s initial conditions, F-e4b46f999bb4 makes the state-cell address accessible to a later dependency. If a second-region dependency writes a nonzero value through it before the final test, the function continues acquisition. Such address sharing is not established to occur; the example demonstrates why the earlier zero write alone cannot guarantee termination.

6. **Zero old end on an inline path.**  
   A region’s end is zero and its boundary is nonzero. The inline path writes 4 to its end field. For the first region, the element store is skipped. For the second, F-67a6a52fe86f is skipped and 4 is available as the next auxiliary value. This describes the visible branch, not a valid collection-layout guarantee.

7. **No explicit clearing by this function.**  
   The destination initially contains entries, and the first dispatched operation returns zero. The function performs no explicit clearing or append operation. Existing entries are preserved only if the entry, acquisition, and dispatch dependencies leave them unchanged.

8. **Fresh second-region fields.**  
   F-e4b46f999bb4 changes the second-region end and boundary so they are equal. After that helper returns, the function observes those fields and selects F-148222599059, regardless of their values before the first-region operation.

9. **Unbounded continuation.**  
   Every dispatched result is nonzero, every final state test observes nonzero, and all accesses and dependencies continue successfully. No bounded-count termination is established.

## Uncertainties

The following remain **documented_only**:

- The semantic purpose of the entry gate and its result words.
- Whether the entry result is a declared 64-bit return or a representation of multiple machine outputs.
- The ABI meaning and stability of secondary results used after second-region dependencies.
- Concrete destination types, descriptor contents, and second-region element layout.
- Dependency allocation, ownership, memory-preservation, global-state, and failure behavior.
- The complete calling convention and effects of F-67a6a52fe86f.
- Whether the escaped state-cell address is retained, published, or accessed by later dependencies.
- Behavior at arithmetic overflow or address-space wrap boundaries.

The representation establishes 32-bit words and four-byte end advancement. It does not establish checked arithmetic, intentional modular wrapping, or portable signed-overflow behavior. Defined execution therefore requires valid address calculations rather than an assumed wrap policy.

Coordinated append behavior is inferred intent. Successful insertion into both regions, equal collection lengths, and transactional consistency are not established.

## Evidence

This contract uses only static inspection of the supplied function representation and opaque dependency mapping. Established claims concern visible widths, offsets, comparisons, argument flow, access ordering, branches, state tests, and normal return values.

Collection and acquisition terminology describes observable structure rather than recovered types. Unknown dependency effects are not replaced with preservation, ownership, or lifetime assumptions.

No runtime tests, dependency implementations, ABI confirmation, or independent memory-layout validation were available or performed.
