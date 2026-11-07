<!-- Trusted provenance: F-186eb8c5ea0f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-186eb8c5ea0f

## Purpose

F-186eb8c5ea0f performs a query-dependent sequence of calls involving a supplied object, writes a one-byte marker in that object, and then performs additional dispatches on both the supplied object and an object returned by a dependency.

The initial query controls only the conditional portion. On either query outcome, normal continuation reaches the marker write, two dispatches on the supplied object, an object-producing dependency, and three dispatches on its result.

This is a structural behavioral contract. Initialization, activation, registration, notification, and cleanup are not established purposes. The meanings of the data operands and the marker remain unknown.

## Interface

### Input and representation

The function receives one pointer to a mutable **primary object**. The supplied representation uses 32-bit words and pointer-bearing fields. Relevant offsets are byte offsets:

| Location | Observed use |
|---|---|
| Primary object + 0 | Dispatch-table pointer |
| Primary object + 44 | Destination of a one-byte store |
| Primary object + 136 | Pointer used to reach a secondary object |
| Primary object + 140 | Address passed to F-438680ffebfe |
| Secondary object + 0 | Dispatch-table pointer |
| Returned object + 0 | Dispatch-table pointer |

The dispatch-table offsets are:

| Byte offset | Observed operation |
|---|---|
| 32 | Initial query reached through the secondary object |
| 36 | Conditional operation reached through the secondary-object field |
| 88 | Repeated operations on the primary object and returned object |

The primary-object offsets follow from four-byte indexing: 34 words correspond to 136 bytes, 35 words to 140 bytes, and 11 words to 44 bytes. Dispatch-table offsets are already byte displacements; they are not word indices.

These observations do not establish complete object sizes, class identities, field ownership, alignment guarantees, or a portable calling convention.

### Output

No explicit return value is represented. Observable results consist of the ordered dependency calls, their operands, the direct marker write, and any dependency-mediated effects.

### Preconditions for normal completion

Each memory access and indirect call actually reached must be valid:

- The primary object must remain accessible for its reached field reads, dispatch lookups, and byte write.
- The initial secondary-object field must support the lookup at dispatch-table offset 32.
- On the nonzero path, the secondary-object field read after F-29eca01f3b88 must support the operation at offset 36.
- Every primary-object dispatch requires a usable dispatch-table pointer and entry at offset 88 at that time.
- The object returned by F-ac356e5603c7 must support each of the three subsequent dispatches at offset 88.
- The address primary object + 140 and the separate external address passed with it must satisfy F-438680ffebfe’s unknown requirements.
- All reached dependencies must return normally for the entire represented sequence to complete.

There are no visible null checks, bounds checks, overflow checks, indirect-target validation, or local recovery paths. The displacements describe required accesses, not checked arithmetic or authorization to access an otherwise invalid address.

## Behavior

### External operands and read identity

The contract uses the following labels instead of unavailable semantic names:

| Label | Meaning |
|---|---|
| A | A 32-bit value captured from an external location immediately after the initial query |
| B location | An external 32-bit location read twice during the conditional sequence |
| Captured B | The first of those two B-location reads |
| C location | An external 32-bit location read twice after F-ac356e5603c7 returns |
| Captured C | The first of those two C-location reads |
| P1–P3 | External 32-bit operands for the three conditional primary-object dispatches, in order |
| P4–P5 | External 32-bit operands for the two unconditional primary-object dispatches, in order |
| Q1–Q3 | External 32-bit operands for the returned-object dispatches, in order |
| R | The address of a separate external data location |

A captured value and a later read of the same location are distinct observations. Neither location stability nor equality of successive reads is assumed.

R is passed as an address. Its use does not establish that the function itself reads the contents at that address.

### Initial query and A capture

The function reads the secondary-object pointer from primary object + 136, follows that object’s dispatch-table pointer, and invokes the entry at byte offset 32.

No explicit arguments are represented for this query. Selecting a target through an object does not, by itself, establish that the object is passed as an explicit receiver. Any implicit receiver or machine-state convention is unresolved.

The query result is tested as a 32-bit integer. Zero omits the conditional sequence; every nonzero result enters it. Negative results are not excluded.

After the query returns, the function reads and captures A. This read occurs on both paths, before any conditional dependency call. Subsequent uses of A refer to this capture, not fresh reads of its external location.

### Conditional operations

When the query result is nonzero, F-29eca01f3b88 is called with the primary-object pointer.

After that call returns, the secondary-object field at primary object + 136 is read again for the operation at dispatch-table offset 36. The represented arguments are the secondary-object pointer and zero. The function does not deliberately retain the initial query’s secondary-object pointer for this operation. A replacement installed by F-29eca01f3b88 can therefore determine the later operation’s object.

F-438680ffebfe is then called with the address primary object + 140 and R. The first argument is an address within the primary object, not a value loaded from that address.

Three primary-object dispatches at offset 88 follow:

| Conditional dispatch | Represented explicit operands |
|---|---|
| First | Primary-object pointer, captured A, P1 |
| Second | Primary-object pointer, a fresh read of the B location, P2 |
| Third | Primary-object pointer, captured B, P3 |

The B capture occurs **after the first conditional primary-object dispatch completes and before the second begins**. The second dispatch nevertheless reads the B location again as part of its own call expression. It does not explicitly use captured B. Only the third conditional dispatch explicitly uses that capture.

Thus, the established read pattern is two reads of the B location: an earlier capture retained for the third dispatch, and a separate read supplying the second dispatch. Those values may coincide, but equality requires a stability assumption absent from the supplied evidence.

After all three conditional primary-object dispatches, the function calls F-6ac70711bbb5 and then F-93bd7d506529. Neither call has explicit arguments in the supplied representation.

Each primary-object dispatch resolves its target again through the primary object’s dispatch table. An earlier dependency may therefore affect a later dispatch by changing that table pointer or its entries.

### Marker write and unconditional primary-object dispatches

After the conditional sequence finishes—or immediately after it is skipped—the function stores the byte value 1 at primary object + 44.

This is an eight-bit store, not a 32-bit field assignment. The store itself does not overwrite bytes 45 through 47.

Two primary-object dispatches at offset 88 follow, in this order:

| Unconditional dispatch | Represented explicit operands |
|---|---|
| First | Primary-object pointer, captured A, P4 |
| Second | Primary-object pointer, captured A, P5 |

Both use the A value captured immediately after the initial query. Conditional calls and changes to A’s external location do not cause A to be reloaded.

The marker write precedes both dispatches. It does not establish that the marker remains 1 at function completion: later dependencies may modify the primary object, including that byte.

### Returned-object dispatches

After the two unconditional primary-object dispatches, F-ac356e5603c7 is called with no explicit arguments. Its result is treated as an object pointer.

The function then reads the C location and retains captured C. Three dispatches through the returned object’s table entry at offset 88 follow:

| Returned-object dispatch | Ordinary leading operands |
|---|---|
| First | Returned-object pointer, a fresh read of the C location, Q1 |
| Second | Returned-object pointer, captured C, Q2 |
| Third | Returned-object pointer, captured C, Q3 |

The first dispatch does **not** explicitly receive captured C. Its call expression separately reads the C location after the capture. The second and third dispatches use the earlier capture.

The first dispatch’s represented argument list additionally includes an apparent function-pointer value obtained from the returned object’s offset-88 entry and four 32-bit machine-state-derived values. Their effective ABI roles are unresolved. They must not be silently discarded to assert a three-argument signature, nor assigned invented semantic meanings.

The function retains the returned object pointer across all three dispatches, but resolves the dispatch-table target again for each call. The first call expression also contains the additional function-pointer lookup just described. No stable-table assumption establishes that its value must equal the invoked target under mutation.

The represented sequence ends after the third dispatch. The supplied static representation explicitly has a control-flow recovery limitation at that final operation, so the precise ABI-level interpretation of the ending remains unresolved.

### Ordering boundaries

Completed call statements, captures, and the marker write have the sequence described above. Separate operand reads within one call expression do not have an established relative evaluation order.

In particular:

- The fresh B read and P2 read are part of the second conditional primary-object call expression; their order relative to that expression’s target lookup is unspecified.
- The fresh C read, Q1 read, invoked-target lookup, and additional function-pointer lookup belong to the first returned-object call expression; their relative read order is unspecified.
- The B and C captures precede their respective later call expressions, but that does not turn the fresh reads into uses of those captures.
- P and Q operands are read for their respective calls, not established as one initial snapshot.

## Side effects

### Direct state change

The only explicit object-memory write is the one-byte store of 1 at primary object + 44.

The representation does not show an explicit allocation, deallocation, retain, release, or write to an external data location by F-186eb8c5ea0f itself.

### Dependency-mediated changes

Every direct dependency and indirect dispatch may have additional effects. Their contracts are unavailable. In particular, they are not assumed to preserve:

- External operand locations.
- The primary object or its dispatch table.
- The secondary-object field.
- The returned object or its dispatch table.
- The marker byte.
- Object lifetimes or shared state.

Later explicit reads are therefore significant. Conversely, later changes to an external location do not retroactively alter an already captured value.

### Ownership and lifetime

Passing the primary-object pointer, an interior address, or R does not establish ownership transfer, copying, retention, or eventual release.

F-ac356e5603c7’s returned pointer has no established ownership convention. No explicit release of that pointer is represented here. This does not prove either a leak or the absence of dependency-managed lifetime effects.

Valid continued access is a precondition for normal execution, not a guarantee that dependencies preserve lifetime.

### Errors and interrupted execution

The query’s result controls a branch; zero is not established as an error.

Other dependency return values are not tested for status. No local rollback, retry, exception handling, or error result is shown. If a dependency does not return normally, subsequent operations are not established to occur, and no visible mechanism reverses earlier effects.

Invalid pointers, unavailable memory, and invalid call targets have no specified recovery behavior in this contract.

## Dependencies

| Dependency | Established invocation relationship |
|---|---|
| F-29eca01f3b88 | Conditional; receives the primary-object pointer before the secondary-object reload |
| F-438680ffebfe | Conditional; receives primary object + 140 and R |
| F-6ac70711bbb5 | Conditional; follows the three conditional primary-object dispatches; no explicit arguments |
| F-93bd7d506529 | Conditional; follows F-6ac70711bbb5; no explicit arguments |
| F-ac356e5603c7 | Unconditional after the two post-marker primary-object dispatches; result used as an object pointer |

Indirect targets have no supplied opaque function IDs. They are identified only by their observed dispatch locations: secondary-object offsets 32 and 36, primary-object offset 88, and returned-object offset 88. These offsets identify lookup sites, not stable function identities.

There are **four mapped branch-only dependency calls**. The conditional secondary-object operation is an additional indirect call, not a fifth mapped dependency.

No dependency’s semantic behavior, preservation guarantees, or ownership rules are established beyond these invocation relationships.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests.

### Zero query result

Given valid reached accesses and normally returning dependencies, a zero query result produces the following behavior:

- F-29eca01f3b88, F-438680ffebfe, F-6ac70711bbb5, and F-93bd7d506529 are not called.
- The secondary-object operation at offset 36 is omitted.
- All three conditional primary-object dispatches are omitted, including both reads of the B location.
- A is still captured.
- The marker byte is written to 1 before the P4 and P5 dispatches.
- Both primary-object dispatches use captured A.
- F-ac356e5603c7 is called, followed by the Q1, Q2, and Q3 dispatches.

This path contains five offset-88 dispatches: two on the primary object and three on the returned object.

### Nonzero query result

Under the same normal-completion conditions, query results of 1 and −1 both enter the conditional sequence.

All conditional operations precede the marker store. There are eight offset-88 dispatches: five on the primary object and three on the returned object.

### Secondary-object replacement

If F-29eca01f3b88 replaces the secondary-object field and it remains valid and stable during the next call expression, the offset-36 operation uses that replacement object. Retaining the original query object instead would contradict the subsequent field read.

### A capture survives external changes

If a conditional dependency changes A’s external location after A was captured, the conditional dispatch using A and both post-marker primary-object dispatches still receive the earlier captured value.

### B capture and fresh read remain distinct

Suppose the B capture obtains value X and the separate B read supplying the second conditional primary-object dispatch obtains value Y. That second dispatch receives Y; the third receives X.

If the second dispatch changes the B location to Z, the third still receives X. The evidence does not establish a deterministic mechanism causing X and Y to differ, but it does establish the two distinct reads and which dispatch consumes each result.

### C capture and fresh read remain distinct

Suppose captured C is X and the separate read in the first returned-object call expression yields Y. The first returned-object dispatch receives Y as its second ordinary operand; the second and third receive X.

A change to the C location by the first dispatch does not change those later captured-C operands. No particular ordering is required among the first expression’s fresh C read and its other operand or target reads.

### Byte-store boundary

If bytes 45–47 hold unrelated state immediately before the marker write, the explicit store alone leaves those bytes unchanged. This criterion does not constrain effects of later dependencies.

### Interrupted conditional execution

If F-6ac70711bbb5 does not return normally, the represented sequence does not reach F-93bd7d506529, the marker write, or the unconditional dispatches. No rollback of earlier effects is established.

## Uncertainties

The following remain **documented_only**:

- Semantic purpose and operand meanings.
- Concrete external values, location mutability, and semantic types.
- Complete layouts, object identities, aliasing relationships, and lifetime rules.
- Implicit inputs and effective calling convention of the initial query.
- ABI significance of the first returned-object dispatch’s additional apparent operands.
- Precise low-level control flow at the final indirect operation.
- Dependency allocation, release, retention, callback, reentrancy, and shared-state behavior.
- Behavior after invalid accesses or abnormal dependency completion.
- Concurrency, atomicity, synchronization, and guarantees about stable reads.
- Any durable completion invariant associated with the marker.

“Marker” names the observed byte write only. Treating it as a Boolean field, activation flag, or guaranteed successful-completion indicator would be inferred intent.

## Evidence

This contract rests on the supplied static function representation and dependency-to-opaque-ID mapping. Prior prose, comments, and review feedback are not independent behavioral authority.

Established static evidence includes the 32-bit operand widths, offset arithmetic, zero/nonzero branch, ordered calls, one-byte store, repeated dispatch lookups, and distinct capture-versus-fresh-read patterns for B and C.

The representation establishes an earlier B capture followed by a separate B read for the second conditional primary-object dispatch, with the third using the capture. It likewise establishes an earlier C capture followed by a separate C read for the first returned-object dispatch, with the second and third using the capture.

Semantic intent and unprovided dependency guarantees are not established. The unusual final-call representation leaves genuine interface and control-flow uncertainty.

No runtime tests have been performed. No dynamic traces, concrete operand values, dependency effects, or successful executions have been observed.
