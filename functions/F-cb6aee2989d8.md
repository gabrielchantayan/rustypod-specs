<!-- Trusted provenance: F-cb6aee2989d8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cb6aee2989d8

## Purpose

F-cb6aee2989d8 coordinates an operation over a context, a mutable cursor, and a boundary token. It publishes 32-bit words through an output destination and returns a packed 64-bit value.

Two paths are established by the supplied body:

- A conditionally selected bulk path calls a dependency, resets fields reached through the context’s current anchor, clears a context field, and publishes the resulting anchor address.
- A repeated path captures a cursor word, calls an advancement dependency, supplies retained result words by reference to a per-item dependency, and publishes the then-current high result word. A comparison result of exactly one terminates this path.

The layout suggests an anchored container and a range operation. That is inferred intent, not an established requirement. Element types, container invariants, destruction, allocation, and ownership transfer are not established.

The output word and packed return are distinct results. In particular, the packed return’s low component is not generally the word published through the output destination.

## Interface

### Arguments

The function receives four arguments:

| Argument | Visible representation and use |
|---|---|
| Output destination | Address supporting a 32-bit write. Its address also initializes the retained low return component. |
| Context | 32-bit address used to access fields at byte offsets 16 and 20. Its address initializes the retained high return component. |
| First cursor | Address passed to comparison and advancement dependencies. Each repeated-path iteration reads a 32-bit word through this address. |
| Boundary token | 32-bit value supplied directly to comparison calls. The body does not dereference it directly. |

The cursor and boundary need not have identical representations merely because both are comparison arguments. Their compatibility requirements belong to F-dac28b48a0fc’s unavailable contract.

### Directly accessed storage

“Anchor” denotes the address read from context offset 16. This name does not establish a node type, sentinel role, or ownership relationship.

| Base | Byte offset | Width | Direct use |
|---|---:|---:|---|
| Context | 16 | 32 bits | Read as an anchor address |
| Context | 20 | 32 bits | Tested for nonzero; cleared on the bulk path |
| Anchor | 4 | 32 bits | Read for the bulk dependency; subsequently cleared |
| Anchor | 8 | 32 bits | Read for initial comparison; assigned an anchor address during reset |
| Anchor | 12 | 32 bits | Assigned an anchor address during reset |

All offsets are byte offsets. The directly accessed values and address-bearing words are 32 bits wide.

### Return value

The return is a 64-bit concatenation:

- Bits 63–32: the high component.
- Bits 31–0: the low component.

These are numeric bit positions, not a statement about memory byte order.

On the bulk path, the high component is the final anchor address read before the final output write. The low component is the retained output-destination address.

On the repeated path, both components come from retained 32-bit storage:

- The high component is initialized to the context address.
- The low component is initialized to the output-destination address and overwritten with the captured cursor word on each iteration, after the advancement call.

F-b194cadb2a11 receives addresses of both retained components. Their contents after exposure cannot be treated as exclusively controlled by this function.

### Preconditions

The body performs no explicit null, alignment, bounds, or lifetime validation. Defined execution requires every actual memory access and dependency invocation to be valid when performed.

The output destination must support each required 32-bit write. Context and anchor storage must support the accesses reached on the selected path. The cursor must support a 32-bit read whenever an iteration begins. Dependency-specific requirements remain additional preconditions.

No checked address arithmetic is visible. The evidence does not establish valid wrapping behavior, overflow recovery, or support for addresses wider than the represented 32-bit values.

## Behavior

### Entry reads and initial output

Before any dependency call, the function performs these observable memory operations in order:

1. Reads the anchor word from context offset 16.
2. Writes that word through the output destination.
3. Reads context offset 16 again.
4. Reads the word at offset 8 of the anchor obtained by that second lookup.

The word from anchor offset 8 is retained in temporary storage for the first comparison. Before that comparison, the function also initializes the retained return components from the output-destination and context addresses.

The second context lookup is a fresh read. It must not be replaced contractually with the first value: the initial output write may alias context storage and change what the second lookup obtains.

### Bulk-path eligibility

The first invocation of F-dac28b48a0fc receives:

- The first cursor address.
- The address of the temporary word copied from anchor offset 8.

A result of zero skips the remaining bulk eligibility checks and proceeds to the repeated path.

For any nonzero first result, the function freshly reads context offset 16 into another temporary word. A second invocation of F-dac28b48a0fc receives:

- The boundary token.
- The address of that temporary anchor word.

Only if this second result is nonzero does the function read context offset 20.

The bulk path is selected precisely when:

1. The first comparison result is nonzero.
2. The second comparison result is nonzero.
3. The subsequently read context field at offset 20 is nonzero.

Neither comparison must return exactly one for bulk eligibility. Negative results and positive results other than one qualify. Likewise, the context field is tested only for zero versus nonzero; no positive-count requirement is established.

The temporary words are exposed by address. A comparison may modify them or retain their addresses while their storage remains alive. The body does not use their modified contents as the source of the later anchor reset or return components. This does not imply that comparison side effects are otherwise irrelevant.

### Bulk call and reset

On the bulk path, the function freshly reads context offset 16, then reads offset 4 of that anchor. It calls F-e7bd4289b2a9 with:

- The original context address.
- That 32-bit anchor-associated word.

The dependency’s return value has no visible direct use.

After normal return, the function performs these writes and reads in statement order:

1. Writes an anchor address to offset 8 of an anchor.
2. Writes zero to offset 4 of an anchor.
3. Writes an anchor address to offset 12 of an anchor.
4. Writes zero to context offset 20.
5. Reads context offset 16 as the final anchor result.
6. Writes that result through the output destination.
7. Returns the final anchor result as the high component and the retained output-destination address as the low component.

The anchor accesses during reset are derived from fresh reads of context offset 16. The function does not preserve a pre-call anchor snapshot for this purpose. If F-e7bd4289b2a9 replaces the context’s anchor, the subsequent accesses use the current context field.

Each assignment of an anchor address to anchor storage entails an anchor lookup for the destination and a context-field read for the value. The supplied representation establishes the order between statements, but not a total order between those reads within an individual assignment.

With stable, nonaliased storage after the bulk call, the reset leaves:

- Anchor offset 4 equal to zero.
- Anchor offsets 8 and 12 equal to the anchor address.
- Context offset 20 equal to zero.

These simplified postconditions require the stated stability and aliasing assumptions. The final output write can itself affect reachable storage if the output destination aliases it.

The bulk path returns without entering the repeated-path comparison loop.

### Repeated-path comparison

If bulk eligibility fails at any stage, the function makes a fresh call to F-dac28b48a0fc with:

- The first cursor address.
- The boundary token.

This call is independent of the eligibility comparisons. A prior result does not eliminate or determine this comparison.

The repeated path stops only when the result is exactly one. Every other result enters an iteration, including zero, two, and negative one.

### Repeated-path iteration

Each iteration has the following order:

1. Read a 32-bit word through the first cursor address and retain it as the captured item.
2. Call F-047d5ca4034d with the first cursor address.
3. Replace the retained low return component with the captured item.
4. Call F-b194cadb2a11 with:
   - The address of the retained high component.
   - The original context address.
   - The address of the retained low component.
5. Read the then-current high component and write it through the output destination.
6. Perform the next repeated-path comparison.

The captured item comes from before advancement. The per-item call occurs after advancement. No additional cursor-word read is used to initialize that iteration’s low component.

F-b194cadb2a11 can modify both referenced components. Its return value has no visible direct use.

### Retained references and intervening calls

The body does not reinitialize the high component between iterations. Each per-item call receives its address and therefore has access to its then-current contents.

This is not a guarantee that a later per-item call sees the value left by the preceding per-item call. F-b194cadb2a11 may retain the supplied address while the local storage remains alive. An intervening comparison or advancement dependency may then modify the high component through that escaped reference. Whether such references are retained or used is unknown.

The low component is also exposed by address. Later dependencies may modify it through an escaped reference, but the body explicitly overwrites it with the newly captured item after each advancement call and before the next per-item call. A modification to the low component during advancement is therefore superseded by this assignment, assuming normal valid execution.

After the terminating comparison returns one, the function returns the then-current retained components without another output write. Once their addresses have escaped, the terminating comparison may modify either component. Consequently, the returned high component need not equal the last word published through the output destination.

If no iteration occurs, no per-item call exposes these retained components. Their explicit initial values produce a return with the context address high and the output-destination address low, absent otherwise unestablished access to their storage.

### Termination and failure

There is no visible iteration bound, explicit error result, recovery branch, or exception handler.

Repeated-path termination requires F-dac28b48a0fc eventually to return exactly one. The advancement call alone does not establish progress or eventual termination.

If a dependency does not return normally, no rollback is established. Earlier output writes and state mutations may remain observable.

## Side effects

The function directly performs:

- One unconditional entry output write.
- One additional output write on the bulk path.
- One output write after each normally completed per-item call.
- Bulk-path writes to anchor offsets 8, 4, and 12, followed by context offset 20.

Dependencies can introduce additional effects. Their contracts are unavailable; they must not be assumed to preserve context fields, cursor contents, globals, or storage accessible through retained references.

In particular, passing addresses of retained result words creates a possible means for later calls to alter those words during the current invocation. No validity of those addresses after this function returns is established.

No direct allocation or release semantics are established. The bulk dependency’s anchor-associated argument does not prove deallocation. The captured item’s passage by reference does not prove ownership transfer. Advancement does not establish whether the previously captured item remains valid.

The body provides no synchronization and establishes neither thread safety nor an atomic multi-field reset.

## Dependencies

| Opaque function ID | Established use | Unresolved contract |
|---|---|---|
| F-dac28b48a0fc | Eligibility comparisons tested for nonzero; repeated-path comparisons tested for exact equality to one | Comparison relation, argument representation, mutation, retained references, global effects, failure behavior |
| F-047d5ca4034d | Called with the cursor after each item capture and before low-component replacement | Advancement meaning, invalidation, progress, ownership effects, access through previously escaped references |
| F-b194cadb2a11 | Receives addresses of both retained result words and the original context address | Result mutations, reference retention, item semantics, ownership, other state changes |
| F-e7bd4289b2a9 | Receives the context and a freshly obtained anchor offset-4 word on the bulk path | Traversal, destruction, clearing, release, context mutation, failure behavior |

Only comparison return values visibly control branching. Ignoring another dependency’s return value does not make that dependency’s side effects irrelevant.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless an example states otherwise, assume valid, stable, nonaliased storage and no dependency side effects beyond those specified.

### Noncanonical results select the bulk path

The first eligibility result is two, the second is negative one, and context offset 20 is nonzero.

Both comparison predicates pass. The bulk dependency runs once, the reset follows, and the final anchor is published. No repeated-path comparison occurs.

### Zero state field prevents bulk execution

Both eligibility comparisons return one, but context offset 20 is zero.

The bulk dependency is skipped. A fresh repeated-path comparison occurs. If it returns one immediately, there is no advancement or per-item call.

### No repeated-path iteration

The first eligibility comparison returns zero, and the first repeated-path comparison returns one.

Only the entry output write occurs. The packed return contains the original context address high and output-destination address low.

### Per-item call changes both result words

An iteration captures an item, advances the cursor, and invokes F-b194cadb2a11. That dependency replaces the high and low components with two specified words.

The output receives the replacement high word. If the next comparison returns one without modifying either component, the packed return contains those replacement words.

### Escaped high component changes between iterations

A first per-item call writes a high word and retains its address. The next comparison returns zero. The advancement dependency then uses that retained reference to write a different high word.

The body does not restore the earlier high word before the second per-item call. That call receives the address of storage containing the newer word. This behavior is compatible with the visible body; preservation across intervening dependencies is not required.

### Low component is overwritten after advancement

A prior per-item call retains the low component’s address. During a later iteration, advancement modifies that storage through the retained address.

After advancement returns, the body replaces the low component with the item captured before advancement. The next per-item call therefore receives that captured item in the low component, unless some additional stated effect changes it afterward.

### Terminating comparison changes the return

A per-item call publishes a high word and retains addresses of both result components. The next comparison modifies those components through the retained references, then returns one.

No further output write occurs. The packed return uses the modified components; the output destination still contains the previously published word under the example’s nonaliasing assumptions.

### Only exact one terminates repetition

Successive repeated-path comparison results are two, negative one, and one.

Two iterations occur. Neither nonzero result before the final one terminates repetition.

### Bulk dependency replaces the anchor

The bulk dependency changes context offset 16 to a different valid anchor.

Under otherwise stable, nonaliased storage, the replacement anchor is reset and published. The original anchor is not automatically reused.

## Uncertainties

The following remain **documented_only**:

- Abstract container type, element type, and intended range semantics.
- Meaning of the anchor fields and context offset 20.
- Comparison relation and supported cursor/boundary representations.
- Supported aliasing relationships.
- Whether dependencies retain references to temporary or result storage.
- Whether intervening dependencies modify result storage through escaped references.
- Ownership, allocation, destruction, release, and invalidation rules.
- Full ABI interpretation of the packed return.
- Address-overflow, fault, exception, concurrency, and reentrancy guarantees.
- Additional memory accessed by dependencies.
- Any invariant proving eventual exact-one termination.

The body’s lack of high-component reinitialization is established. Preservation of that component across dependency calls is not.

## Evidence

Established static evidence comes from the supplied function body: 32-bit memory accesses, fixed byte offsets, dependency arguments, statement ordering, branch predicates, by-reference exposure of retained words, and 64-bit concatenated returns.

The nonzero bulk predicates and exact-one repeated-path predicate are distinct. Fresh context reads remain significant across calls and potentially aliasing writes.

Escaped-reference mutation is a permitted possibility, not an observed dependency behavior. No dependency implementations or independent contracts were supplied. The anchored-container interpretation remains inferred intent.

No runtime tests have been performed.
