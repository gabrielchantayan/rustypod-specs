<!-- Trusted provenance: F-883ff1acbf29; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-883ff1acbf29

## Purpose

F-883ff1acbf29 obtains a 32-bit payload through an external descriptor, passes that payload to F-b5bbbba23d63 together with an address inside a receiver, and then selects optional indirect calls using an 8-bit selector read from the receiver.

The normal return value identifies a control-flow outcome, not dependency success:

- A zero descriptor returns zero without accessing the receiver.
- A nonzero descriptor causes payload acquisition and a call to F-b5bbbba23d63 before selector inspection.
- Selector zero returns zero after that work.
- Every nonzero selector returns one if execution completes normally, including selectors that cause no indirect call.

A receiver-associated update followed by optional actions is plausible intent. The meanings of the payload, selector values, globals, and indirect actions are not established.

## Interface

### Arguments and representation

The function accepts:

| Argument | Established role |
|---|---|
| Receiver | Address of storage containing a dispatch-table reference, a selector-bearing word, and an embedded region passed to F-b5bbbba23d63. |
| Descriptor | A 32-bit address-like argument. Zero selects an immediate return; nonzero values are used to traverse a descriptor chain. |

The supplied representation uses four-byte integer words and 32-bit address-oriented operations. This establishes the offsets and scalar widths described below, but not a complete platform ABI.

All offsets in this contract are byte offsets.

### Receiver layout

| Offset | Direct use |
|---|---|
| `+0x00` | Read to obtain the dispatch-table address when an indirect call is required. |
| `+0x18` | Read as a four-byte word and narrowed to an 8-bit character for selector dispatch. |
| `+0x8E8` | Address formed and passed to F-b5bbbba23d63. |

The embedded-region offset is `0x23A × 4 = 0x8E8` bytes. Forming this address does not itself read the region. Its size, alignment requirements, contents, and lifetime contract are unknown.

Dispatch depends on the narrowed selector, not on the complete word at `+0x18`. Under the supplied representation, the low eight bits determine the recognized selector values. Character signedness does not change recognition of zero, one, two, or three; all other nonzero character values follow the same fallback path.

### Descriptor layout

For a nonzero descriptor, payload acquisition consists of three dependent reads:

1. Read a 32-bit address-like value at descriptor offset `+0x08`.
2. Read an address-like value at the resulting address plus `0x0C`, interpreted as a pointer to a four-byte payload.
3. Read that 32-bit payload.

The address-oriented representation is 32-bit; a broader pointer ABI is not independently established. Neither intermediate address receives a visible null or validity check.

The offsets are byte displacements. No bounds check or overflow check is visible for these address computations. The contract does not establish useful normal behavior for invalid, overflowing, or otherwise unusable address calculations.

### Dispatch table

When dispatch is required, the table address comes from receiver offset `+0x00`.

| Table offset | Use |
|---|---|
| `+0x74` | One indirect call for selector one. |
| `+0x58` | One indirect call for selector two; three sequential indirect calls for selector three. |

The entries are treated as callable addresses. Target identities, declared parameter types, return types, and calling conventions are unavailable.

### Global inputs and width limits

Four distinct global locations appear in the supplied representation. This contract labels them by use:

| Label | Use and established width information |
|---|---|
| Primary global | Read after the helper and assigned to a retained 32-bit value. Also referenced freshly in selector-one dispatch and the first selector-three call. Its declaration, storage width, and read width are not supplied. |
| Selector-one global | Passed directly to the selector-one indirect call. Its declaration, storage width, read width, and effective argument width are unknown. |
| First selector-three global | Passed directly to the first selector-three call. Its declaration, storage width, read width, and effective argument width are unknown. |
| Second selector-three global | Passed directly to the second selector-three call. Its declaration, storage width, read width, and effective argument width are unknown. |

The **saved primary snapshot** has an established 32-bit representation because its destination is explicitly four bytes. That fact does not establish the primary global’s own width or the width of a fresh direct use as an indirect-call argument.

Without the global declarations and target signatures, this contract does not specify any conversion beyond what is established by the 32-bit snapshot destination. In particular, a fresh primary-global argument must not automatically be equated with the saved snapshot in either width or representation.

The labels do not imply immutability, distinct values, resource ownership, or any particular semantic meaning.

### Result

On normal completion, the function returns a four-byte value equal to zero or one. No dependency result is used to choose that value.

## Behavior

### Zero descriptor

A zero descriptor causes an immediate return of zero.

This path performs no receiver read, descriptor-chain read, identified-global read, helper call, or indirect call. The function’s own operations therefore do not require a valid receiver on this path.

### Payload acquisition and helper call

For a nonzero descriptor, the function traverses the descriptor chain and retains the resulting 32-bit payload.

It then calls F-b5bbbba23d63 with these positional arguments:

1. Receiver-relative address `+0x8E8`.
2. The retained 32-bit payload.

The caller does not consume the helper’s result.

These operations occur before the selector is read. An eventual selector of zero or an unsupported value does not bypass descriptor traversal or the helper.

The payload is not reloaded from the descriptor chain later in this invocation. Changes to descriptor storage during a dependency call do not, by themselves, alter the retained value.

### Snapshot and selector acquisition

After F-b5bbbba23d63 returns, the function:

1. Reads the primary global and assigns the result to the saved 32-bit snapshot.
2. Reads the receiver word at `+0x18` and narrows it to the selector character.

The helper may affect both observations. No preservation guarantee is supplied for receiver fields or globals.

The snapshot is acquired even when the selected branch never uses it. The selector is read once and is not refreshed during later indirect calls. Consequently, an indirect dependency’s subsequent selector change does not redirect this invocation.

### Selector zero

The function makes no indirect call and returns zero.

Descriptor traversal, the helper call, the primary-global snapshot, and selector acquisition have already occurred. Zero therefore does not mean that the invocation had no effects.

### Selector one

The function resolves the target through receiver offset `+0x00` and table offset `+0x74`, then invokes it with four explicit arguments:

1. Receiver address.
2. A fresh reference to the primary global.
3. The selector-one global.
4. The retained 32-bit payload.

The second argument is not the saved 32-bit snapshot. Its effective width and conversion cannot be established from the available declarations and signatures.

After the indirect call returns, the function returns one. The call result is ignored.

### Selector two

The function resolves the target through receiver offset `+0x00` and table offset `+0x58`, then invokes it with two explicit arguments:

1. Receiver address.
2. The saved 32-bit primary snapshot.

There is no branch-specific fresh primary-global read for this call.

After the call returns, the function returns one, ignoring its result.

### Selector three

The function performs three indirect calls in order. Each resolves its target using table offset `+0x58`.

| Call | Explicit positional arguments |
|---|---|
| First | Receiver; fresh primary-global reference; first selector-three global. |
| Second | Receiver; saved 32-bit primary snapshot; second selector-three global. |
| Third | Receiver; saved 32-bit primary snapshot. |

The second call is prepared after the first returns. The third call is prepared after the second returns.

The fresh primary-global reference used for the first call does not replace the saved snapshot. A dependency’s change to the primary global therefore does not cause the second or third call to receive a refreshed primary value.

Target resolution is repeated for each call. The receiver’s dispatch-table reference and the selected entry are not retained across all three calls. If an earlier call changes either location, a later call can invoke a different target, provided subsequent accesses remain valid.

The second selector-three global is used when preparing the second call, after the first call has returned. It is not established as an earlier snapshot.

The final call has only two explicit arguments. The supplied evidence does not establish an additional trailing argument or a default value for one.

After the third call returns, the function returns one. All three call results are ignored.

### Other nonzero selectors

Any narrowed selector other than zero, one, two, or three causes a return of one without an indirect call.

The descriptor reads, helper call, primary snapshot, and selector read still occur. There is no distinct unsupported-selector error result.

### Ordering boundaries

The established sequence is:

- Complete descriptor traversal before calling the helper.
- Complete the helper before acquiring the primary snapshot.
- Acquire the snapshot before reading the selector.
- For selector three, complete each indirect call before preparing the next.

The representation does not establish a total order among all target-lookup and global-argument reads within one indirect-call expression. Consumers must not depend on an unproven ordering between those reads.

## Side effects

### Direct operations

The visible body performs memory reads, address formation, dependency calls, and a scalar return. It contains no explicit memory store, allocation, deallocation, reference-count operation, or ownership-transfer operation.

This does not imply that the invocation is side-effect-free: dependencies may mutate reachable state.

### Dependency-mediated changes and ownership

F-b5bbbba23d63 receives an address inside the receiver and the payload. Its effects are unknown.

The indirect dependencies receive the receiver and the branch-specific values described above. They must not be assumed to preserve:

- Receiver contents or lifetime.
- Dispatch-table references or entries.
- Selector storage.
- Any identified global.
- Descriptor-related storage or resources represented by the payload.

Passing an address or scalar does not establish ownership transfer. Nor does the absence of an explicit release in this caller establish that dependencies retain or release nothing.

The payload’s meaning is unknown. If a dependency treats it as an address, handle, or owned resource, the associated validity and lifetime rules remain outside this established contract.

### Preconditions and failure behavior

For a nonzero descriptor, normal completion requires the exercised descriptor reads and helper call to be valid. Subsequent receiver reads must remain valid after the helper.

Branches that perform indirect calls additionally require valid dispatch-table reads, callable targets, compatible invocation behavior, and sufficient receiver lifetime for all later accesses.

No visible validation establishes:

- Descriptor-chain validity.
- Receiver bounds or alignment.
- Dispatch-table or target validity.
- Payload validity.
- Dependency success.

No local recovery converts a fault, exception, nonreturning call, or invalidated receiver into a specified zero or one result. The return rules apply only to normal completion.

Selector zero and unsupported selectors do not require this caller to read a dispatch table. The helper may independently require additional receiver state; that requirement is unknown.

## Dependencies

| Dependency | Invocation condition | Established caller contract |
|---|---|---|
| F-b5bbbba23d63 | Every nonzero descriptor | Receives receiver-relative address `+0x8E8` and a 32-bit payload. Result is not consumed. |
| Unresolved indirect dependency selected at `+0x74` | Selector one | Called once with four explicit arguments. |
| Unresolved indirect dependency selected at `+0x58` | Selector two or three | Called once for selector two, or three times for selector three, with the differing argument lists specified above. |

F-b5bbbba23d63 is the only dependency with an available opaque function ID. Table offsets do not identify the indirect functions, and repeated resolution of one offset need not produce the same target.

No dependency is established to preserve globals, own or release memory, or communicate success through a result used by this caller.

## Acceptance examples

These are conditional requirements derived from static evidence, not runtime test results. Unless a case states otherwise, all required accesses are valid and all called dependencies return normally.

### Zero descriptor with unusable receiver

Given a zero descriptor and a receiver that cannot safely be dereferenced:

- Return zero.
- Do not access the receiver or descriptor chain.
- Do not read the identified globals or invoke dependencies.

### Selector zero after helper effects

Given a valid nonzero descriptor yielding payload `0xA1B2C3D4`, with the post-helper selector equal to zero:

- Call F-b5bbbba23d63 with receiver-relative address `+0x8E8` and that payload.
- Acquire the primary snapshot, then the selector.
- Make no indirect call.
- Return zero.

### Helper changes dispatch selection

If the selector is initially zero but F-b5bbbba23d63 changes it to three, perform the three-call selector-three sequence. The entry-time selector does not control dispatch.

### Selector one preserves the acquired payload

If the descriptor yields `0x11223344`, and the helper later changes the original payload storage, the selector-one call still receives `0x11223344` as its fourth explicit argument.

Its second argument comes from a fresh primary-global reference, not the saved snapshot. This requirement does not prescribe that argument’s unknown width.

### Selector two ignores upper selector-word bits

Given a receiver selector word of `0x12345602`:

- Select branch two.
- Call the target at table offset `+0x58` with the receiver and saved 32-bit snapshot.
- Return one.

The full word need not equal two.

### Selector three retains the snapshot but refreshes targets

Suppose snapshot acquisition produces the 32-bit value `10`. The first indirect call then changes the primary global and replaces the receiver’s dispatch-table reference.

- The second call still receives snapshot value `10`.
- Its target is resolved using receiver state after the first call.
- Its third argument comes from the second selector-three global as used after the first call.
- The third call again receives snapshot value `10` and resolves its target after the second call.

This example does not assert the width or representation of the fresh primary-global argument passed to the first call.

### Unsupported nonzero selector

Given a post-helper selector of four:

- Perform descriptor traversal, helper invocation, snapshot acquisition, and selector acquisition.
- Make no indirect call.
- Return one.

### Invalid descriptor-chain address

Given a nonzero descriptor whose chain reaches unusable storage, there is no established normal zero-return fallback. Platform failure behavior is outside the normal-completion contract.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the selector, payload, globals, and indirect actions.
- Complete receiver and descriptor layouts, allocation bounds, and alignment requirements.
- Exact ABI, byte order, and indirect-target signatures.
- Declarations, storage widths, and read widths of all four globals.
- Effective widths and conversions of globals used directly as indirect-call arguments.
- The precise conversion from the primary global into the established 32-bit snapshot.
- Dependency effects, ownership rules, preservation guarantees, and lifetime requirements.
- Concurrency, volatility, memory-mapped behavior, and synchronization requirements.
- Exact relative ordering of reads preparing one indirect call.
- Why table offset `+0x58` is invoked with differing explicit argument counts.
- Fault, unwind, and termination behavior outside normal completion.

Retained values explain which later arguments avoid rereading memory. They do not establish atomicity, thread safety, or globally consistent observations.

## Evidence

Established static evidence supports the descriptor-zero guard; dependent descriptor reads; four-byte payload; receiver offsets; helper-before-selector ordering; explicitly 32-bit primary snapshot; selector narrowing; branch-specific argument lists; repeated target resolution; and normal returns of zero or one.

The global references lack supplied declarations. Only the retained primary snapshot—not the primary global’s storage or read, and not the other globals’ arguments—has an established four-byte representation.

The characterization as an update followed by optional actions is inferred intent rather than a semantic guarantee.

No runtime tests have been performed. No dependency implementations, indirect-target identities, global declarations, or external ownership contracts were supplied.
