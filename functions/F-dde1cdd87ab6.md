<!-- Trusted provenance: F-dde1cdd87ab6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dde1cdd87ab6

## Purpose

F-dde1cdd87ab6 constructs a temporary object from three opaque inputs and offers it to recipients in two stages. An optional collection supplies first-stage recipients. If none returns a nonzero result, the function traverses a fallback chain, optionally filtering its nodes.

The function returns the exact 32-bit result of the first recipient callback that returns nonzero. It returns zero if dispatch finishes without such a result. On normal completion, it invokes a finalization method on the constructed object if that object is nonnull.

“Dispatch object,” “recipient,” “collection,” “filtering,” and “finalization” describe roles inferred from the call structure. They do not establish application semantics. In particular, a nonzero callback result stops dispatch; it is not independently established as success, failure, or consumption.

## Interface

### Inputs and output

| Input | Representation | Established use |
|---|---|---|
| First construction input | Opaque 32-bit value | Passed unchanged to F-a22e780f366f. |
| Second construction input | Opaque 32-bit value | Passed unchanged to F-a22e780f366f. |
| Third construction input | Opaque 32-bit value | Passed unchanged to F-a22e780f366f. |
| Initial recipient | 32-bit pointer-shaped value | Copied into a local recipient slot later supplied by address to F-6e4651b532dc. |

The function does not locally interpret or validate the three construction inputs. Their representations do not establish whether they contain integers, pointers, handles, or other data.

The initial recipient is not directly dereferenced before enumeration. Whether enumeration uses its incoming value, replaces it, or conditionally leaves it unchanged is **documented_only**.

The output is a 32-bit integer. Zero means that no invoked recipient returned nonzero before normal dispatch termination. Any nonzero result is returned without Boolean normalization. This includes bit patterns interpreted as negative signed integers.

### Accessed layouts

The supplied representation uses 32-bit pointers and 32-bit object words. All offsets below are byte offsets.

| Receiver or storage | Offset | Established access |
|---|---:|---|
| Context from F-3fdd14ecd250 | `0x2c` | First-stage collection reference. |
| Context from F-3fdd14ecd250 | `0x28` | Fallback head pointer. |
| Dispatch object | `0x00` | Method-table pointer. |
| Dispatch-object method table | `0x04` | Finalization method. |
| Dispatch-object method table | `0x14` | Query method used before fallback traversal. |
| First-stage recipient | `0x00` | Method-table pointer. |
| First-stage recipient method table | `0x34` | Dispatch method. |
| Fallback node | `0x00` | Method-table pointer. |
| Fallback-node method table | `0x30` | Dispatch method. |
| Fallback-node method table | `0x1c` | Method supplying the next node. |
| Fallback node | `0x0c` | Pointer used for metadata access and as a continuation condition. |
| Object referenced by that pointer | `0x0c` | 32-bit eligibility field. |

These are partial access requirements, not complete type or allocation-size declarations. The field at node offset `0x0c` is not itself established as the next-node pointer; the next node comes from the traversal method.

The function reserves a 20-byte local workspace and passes its address to enumeration dependencies. No direct initialization or other direct write to those 20 bytes is visible. Any initialization or mutation of that storage occurs inside the called dependencies. Additional alignment requirements are **documented_only**.

## Behavior

### Construction and initial local state

The function first copies the initial-recipient argument into its local recipient slot.

It then calls F-f2ddcdbfb0c6 without visible arguments. Its result is passed to F-c5a232e4455b together with the 32-bit constant `0x18`, numerically 24. The meaning and unit of this constant are not established.

After that call, the function assigns the initial-recipient argument to the local recipient slot again. It then calls F-a22e780f366f with the result from F-c5a232e4455b followed by the three construction inputs.

The result of F-a22e780f366f is retained as the dispatch-object pointer. The accumulated callback result is initialized to zero.

There are no local null checks between the construction calls. Intermediate null values, if returned, are forwarded to the next dependency. The function does not establish whether those dependencies accept such values.

### First-stage collection

The function calls F-3fdd14ecd250 and reads the returned context’s collection field at `0x2c`.

If that field is zero, first-stage enumeration is skipped. Neither enumeration initialization nor enumeration cleanup is called on this path.

If the field is nonzero, the function reads that field again and calls F-538044b67754 with:

- The address of the 20-byte local workspace.
- The collection value from the second read.
- The 32-bit bit pattern `0xfffffffe`.

The last argument is numerically −2 under signed 32-bit interpretation. Its meaning to the dependency is unknown.

Enumeration then proceeds through F-6e4651b532dc. Each call receives the workspace address and the address of the local recipient slot.

A zero enumeration result ends the first stage. A nonzero enumeration result causes the function to use the recipient currently in the slot: it reads that recipient’s method table and invokes the method at `0x34`, passing the recipient and the dispatch-object pointer.

A zero callback result requests another enumeration step. A nonzero callback result is retained and ends dispatch.

After either enumeration exhaustion or a nonzero first-stage callback result, the function calls F-944b595e25e9 with the workspace address. This call occurs before fallback initialization on exhaustion and before dispatch-object finalization on first-stage acceptance.

A nonzero first-stage callback result bypasses fallback completely.

There is no null check on a recipient following a nonzero enumeration result. Defined execution requires that the slot then contain a receiver valid for the reached method-table access.

### Fallback initialization

Fallback is entered only if the collection was absent or enumeration ended without a nonzero callback result.

The function calls F-3fdd14ecd250 again and reads the fallback head at `0x28` from this newly returned context. It does not reuse the earlier context result. The fallback head is captured before the dispatch-object query.

Filtering is selected as follows:

- With a null dispatch-object pointer, filtering is inactive and no query method is called.
- With a nonnull dispatch object, its method at table offset `0x14` is invoked with that object.
- The query’s represented result is 64 bits. Its low 32 bits alone determine filtering: zero disables filtering; any nonzero value enables it.

The query occurs even if the captured fallback head is null.

The function does not reread the context’s fallback head after the query. A query that changes that context field therefore does not change the already captured starting pointer.

The representation carries the head in the low word of a 64-bit intermediate. Its upper word comes from the query result, or from `0x00000a00` on the null-object path. This upper word does not control recipient selection: when filtering is active, the tested value is replaced by a metadata read; when filtering is inactive, it does not gate dispatch.

### Fallback node processing

Traversal proceeds while the current node is nonnull and the accumulated callback result is zero.

With filtering inactive, the function invokes the current node’s dispatch method at table offset `0x30` without first reading its eligibility metadata.

With filtering active, it first reads the pointer at node offset `0x0c`, then reads the 32-bit eligibility field at offset `0x0c` through that pointer:

- A zero eligibility field permits dispatch without an additional gate call.
- A nonzero eligibility field causes a call to F-35b8bd5fa9e0 with the dispatch-object pointer.
- A nonzero gate result permits dispatch.
- A zero gate result skips the node’s dispatch callback and leaves the accumulated result zero.

Each invoked fallback dispatch method receives the current node and the dispatch-object pointer. Its exact 32-bit return replaces the accumulated callback result.

A nonzero callback result ends traversal immediately. There is no subsequent continuation-field read or traversal-method call for that node.

After a skipped node or a callback returning zero, the function reads the current node’s field at `0x0c`:

- If zero, traversal ends.
- If nonzero, the function calls the current node’s method at table offset `0x1c`, passing the node.
- The low 32 bits of that method’s represented 64-bit result become the next node pointer.

The upper 32 bits of the traversal result do not determine continuation or callback eligibility.

The continuation-field read occurs after any gate call and dispatch callback. When filtering is active, this is a distinct read from the earlier metadata-pointer read. Intervening calls may alter the field; the later read is not replaced by the earlier value.

There is no visible cycle detection or iteration limit. Traversal may continue indefinitely if methods keep supplying nonnull nodes and no callback returns nonzero.

### Finalization and return

After dispatch ends normally, a nonnull dispatch object receives its method at table offset `0x04`. A null object receives no such call.

The method’s result is ignored. The function then returns the accumulated 32-bit callback result.

The finalization call is made once on normal completion for a nonnull dispatch object. This statement does not extend to faults, nonreturning calls, exceptions, or nonlocal exits.

## Side effects

### Direct and indirect memory activity

The visible function explicitly assigns the local recipient slot and local control state. Its direct nonlocal memory accesses are reads of context fields, receiver method tables, node fields, and conditional eligibility metadata.

It reserves the 20-byte enumeration workspace but does not directly write its contents. Passing its address to F-538044b67754, F-6e4651b532dc, and F-944b595e25e9 permits those dependencies to access or mutate the workspace. Their actual writes and initialization rules are not visible.

Similarly, F-6e4651b532dc receives the recipient slot by address and may change it. Such changes are dependency effects, distinct from the function’s explicit assignments of the initial recipient.

No explicit store to a caller-provided object, context field, fallback node, or metadata object is visible. Dependencies and indirect methods may nevertheless mutate those objects or other external state.

### Observable ordering

On normally returning paths:

- Construction precedes context lookup and recipient dispatch.
- Enumeration cleanup precedes either fallback initialization or finalization after first-stage acceptance.
- The second context lookup and fallback-head read precede the dispatch-object query.
- An eligibility-field read precedes its conditional gate call.
- A required gate call precedes that node’s dispatch callback.
- A zero-result callback or rejected gate precedes the continuation-field read.
- The continuation-field read precedes the traversal method.
- Finalization follows dispatch.

These orderings do not imply that unknown callees preserve globals, fields, method tables, or object lifetimes.

### Ownership and lifetime

The function retains the constructed pointer for dispatch and normal-exit finalization. The finalization call does not prove deallocation, reference-count adjustment, or exclusive ownership.

Recipients receive the same pointer, including null if construction returns null. Permission to retain, mutate, or release it is **documented_only**. Defined execution requires that any object or method table used later remain valid.

The workspace has local storage duration. Whether enumeration creates additional resources, and whether F-944b595e25e9 releases them, is unknown. No explicit release of contexts, recipients, nodes, or metadata is visible.

### Errors and preconditions

There is no separate explicit error channel and no local recovery for invalid pointers or failed dependencies. A nonzero callback result must not be classified as an error without an external contract.

Every reached dereference requires valid readable storage, and every reached indirect call requires a valid callable method.

Filtering creates an important boundary: the node’s `0x0c` pointer is dereferenced for metadata before the later continuation null check. With filtering active, a null field violates that dereference precondition. With filtering inactive, a null field can terminate traversal normally after a zero-result dispatch callback.

## Dependencies

| Opaque ID | Established contract at this call site |
|---|---|
| F-f2ddcdbfb0c6 | Called without visible arguments; its result feeds F-c5a232e4455b. |
| F-c5a232e4455b | Receives that result and `0x18`; its result feeds F-a22e780f366f. |
| F-a22e780f366f | Receives the preceding result and three opaque 32-bit inputs; supplies the dispatch-object pointer. |
| F-3fdd14ecd250 | Supplies a context for collection access and, on fallback entry, a separately obtained context for head access. |
| F-538044b67754 | Receives workspace address, collection reference, and `0xfffffffe`. |
| F-6e4651b532dc | Receives workspace and recipient-slot addresses; zero ends enumeration, nonzero permits recipient dispatch. |
| F-944b595e25e9 | Receives the workspace after initialized enumeration ends normally. |
| F-35b8bd5fa9e0 | Receives the dispatch object; conditionally gates a filtered node using zero versus nonzero. |

Indirect dependencies are identified by receiver role and method-table offset. Their identities, internal effects, and broader contracts are unknown.

## Acceptance examples

These are conditional consequences of the static representation, not executed tests. All reached accesses and calls are assumed valid and normally returning unless stated otherwise.

1. **First-stage acceptance.** Recipient callbacks return zero and then `7`. Enumeration cleanup occurs, fallback is bypassed, the nonnull object is finalized, and the result is `7`.

2. **Exact result preservation.** A callback returns `0xfffffffd`, representing −3 as a signed 32-bit integer. Dispatch stops and the function returns that bit pattern, not one.

3. **Empty collection enumeration.** The collection field is nonzero, but the first enumeration step returns zero. Cleanup occurs before the second context lookup. No first-stage recipient method is called.

4. **Workspace handling.** Enumeration initialization writes state into the supplied workspace. Those writes are effects of F-538044b67754, not direct stores performed by F-dde1cdd87ab6. The visible function provides no prior workspace-content initialization guarantee.

5. **Null constructed object.** Construction returns null and first-stage dispatch does not accept. Fallback filtering is inactive; query and finalization are omitted. Invoked recipient methods receive null as the dispatch-object argument.

6. **Empty fallback with an object.** The captured head is null and the object is nonnull. The query still occurs, followed by finalization. With no prior acceptance, the return is zero.

7. **Head changed by query.** The query changes the context’s fallback head. Traversal nevertheless begins from the pointer captured before that query.

8. **Filtered rejection.** Filtering is active, eligibility is nonzero, and F-35b8bd5fa9e0 returns zero. The node’s dispatch callback is skipped. A subsequent continuation read determines whether traversal proceeds.

9. **Zero eligibility.** Filtering is active and eligibility is zero. The gate dependency is omitted and the node’s dispatch callback is invoked.

10. **Continuation changed by callback.** A fallback callback returns zero and clears the node’s `0x0c` field. The later read sees zero; no traversal method is invoked.

11. **Acceptance before continuation.** A fallback callback returns nonzero. Its continuation field is not read afterward. With filtering inactive, that field need not have been read earlier either.

12. **Null metadata under filtering.** Filtering is active and the node’s `0x0c` pointer is null. Defined-execution preconditions are violated; neither a clean zero return nor finalization is guaranteed.

## Uncertainties

The following remain **documented_only**:

- Application meanings and ownership of the construction inputs.
- Allocation behavior and the significance of `0x18`.
- Enumeration interpretation of `0xfffffffe` and the incoming recipient value.
- Workspace layout, initialization requirements, and dependency resource management.
- Semantic meanings of the query, eligibility field, gate, and callback results.
- Broader significance of the represented 64-bit query and traversal results.
- Whether finalization releases memory or changes references.
- Dependency mutations, reentrancy, concurrency, and memory-ordering guarantees.
- Behavior on allocation failure, invalid accesses, exceptions, or nonlocal exits.

Two-stage dispatch and temporary-object finalization are inferred intent. They do not establish an application-specific event model or ownership policy.

## Evidence

Established behavior is grounded in the supplied static representation: argument forwarding, explicit local assignments, workspace-address passing, byte-offset reads, indirect calls, zero/nonzero branches, represented return widths, and normal-exit call ordering.

The workspace distinction follows directly from the visible accesses: storage is reserved and its address is passed onward, but no direct write into its contents is shown.

No runtime tests have been performed. Acceptance examples are conditional static predictions. Dependency implementations, indirect-method semantics, and omitted material are not established by this evidence.
