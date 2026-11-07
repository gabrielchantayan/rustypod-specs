<!-- Trusted provenance: F-4edb61bcc32f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4edb61bcc32f

## Purpose

F-4edb61bcc32f overwrites two words in caller-associated storage, runs a dependency-controlled traversal, resolves further state, conditionally drains a linked structure, and invokes a final dependency.

The two overwritten words receive values **read from global storage**, not established constants. A third global is read separately for two dependency calls. Dependencies execute between these reads, and their global-state effects are unknown; later reads must not be treated as snapshots taken at entry.

The sequence resembles teardown or state replacement. That is **inferred intent**, not an established destruction, ownership-transfer, or memory-release contract.

## Interface

### Input and result

The function accepts one address identifying writable storage, called the **input object** below. It returns no explicit value.

The supplied representation uses four-byte words and four-byte integer results. Address-valued words and resolver results are used as addresses. Offsets in this contract are byte offsets.

There is no visible input validation. In particular, a null input is not handled before the first write.

### Storage terminology

| Term | Meaning |
|---|---|
| Global A | Global word read to obtain the first value written to the input object. |
| Global B | Global word read after the first resolver returns, to obtain the second stored value. |
| Global C | Global word independently read for F-f35532238f81 and F-7fa3e55b7211. |
| First resolved address | Result of F-dae8ca024616. |
| Second resolved address | Result of F-1b9e6fff9bc9. |
| Container address | Second resolved address minus `28` bytes. |
| Conditional field | Four-byte field at second resolved address minus `12` bytes, or container address plus `16` bytes. |
| Head field | Four-byte word at the container address. |
| Token | Four-byte temporary passed by address to F-f35532238f81. |
| Traversal record | Three adjacent four-byte temporary fields supplied to the traversal dependencies. |

These terms do not establish allocation boundaries, dynamic types, non-aliasing, or ownership.

### Directly accessed storage

| Region | Offset | Visible use |
|---|---:|---|
| Global A | `0` | Read a four-byte value before the first object write. |
| Global B | `0` | Read a four-byte value after F-dae8ca024616 returns. |
| Global C | `0` | Read separately for each of two dependency calls. |
| Input object | `0` | Write the value read from Global A. |
| Input object | `176` | Form the argument to F-dae8ca024616. |
| First resolved address | `−176` | Write the value read from Global B. |
| First resolved address | `−28` | Form the argument to F-1b9e6fff9bc9. |
| Second resolved address | `−12` | Read the conditional field at several points. |
| Second resolved address | `−28` | Read and update the head field. |
| Second resolved address | `−148` | Form the final dependency argument. |
| Address obtained from the conditional field | `8` | Read one four-byte word. |
| Current node | `0` | Read its successor value. |
| Current node | `4`, `8` | Read arguments for the first per-node call. |

These are access requirements, not complete object layouts.

### Preconditions and arithmetic

Normal execution requires every reached memory access to be valid. This includes the global reads, the initial object write, negative offsets from resolver results, conditional-state dereferences, and node accesses.

The representation does not establish alignment guarantees, object extents, valid pointer encodings, or checked address arithmetic. Although the visible words and integer results are four bytes wide, no portable overflow or wraparound rule for the address calculations is established. A contract for invalid addresses, arithmetic overflow, or inaccessible memory is **documented_only**.

Dependencies may impose additional preconditions not available here.

## Behavior

### Initial global read and object write

The function reads Global A and writes the resulting four-byte value to offset `0` of the input object.

This occurs before any dependency call. There is no comparison with the old object word, and the old word is not explicitly saved.

The assigned value is the value obtained by this read. The evidence does not establish that Global A is immutable or that its value has a particular semantic meaning.

### First resolution and second global-derived write

F-dae8ca024616 receives input object plus `176` bytes. Its result becomes the first resolved address.

The function places first resolved address minus `176` bytes into the first element of a temporary two-element address array. It then reads Global B and writes that four-byte value through the stored address.

The Global B read occurs **after** F-dae8ca024616 returns. Consequently, if the resolver changes Global B, the subsequent store uses the value obtained after that change, subject to the ordinary validity of the execution. An entry-time Global B value is not retained by this function.

Only the first address-array element is explicitly initialized. The second element has no established initial value.

### Token setup using the first Global C read

The function reads Global C for the call to F-f35532238f81. That call receives:

1. The address of the four-byte token.
2. The value obtained from this Global C read.
3. The address of the temporary address array.

The token has no explicit initialization before the call. Whether F-f35532238f81 supplies a usable token is unknown. Whether it examines or initializes the second array element is also unknown.

No global-preservation guarantee is established for this dependency.

### Traversal initialization and exact termination condition

After F-f35532238f81 returns, the traversal record is initialized in this visible write order:

1. Its field at relative offset `8` is set to zero.
2. Its field at relative offset `4` is set to zero.
3. Its field at relative offset `0` receives the current token value.

The resulting record contains the token followed by two zero words.

F-a1ecaae6738a is then called repeatedly with the traversal-record address and the address of a four-byte result slot.

A return value of exactly `1` ends the loop. Every other return value causes a call to F-dcbd5e1e578f with the traversal-record address, followed by another check.

Thus:

- The first check precedes any advancement call.
- Each non-`1` result has one following advancement call.
- The terminating check has no following advancement call.
- A return value of `2`, another nonzero value, or a negative value does not itself terminate the loop.
- There is no visible iteration bound or alternative exit.

The result slot is not explicitly initialized before its address is passed to the checker. Its later use depends on a usable value having been established; the dependency contract needed to guarantee this is unavailable.

### Completed traversal and the second Global C read

After the checker returns `1`, the function copies the current result-slot value into a separate four-byte temporary. It then copies the current token value into another four-byte temporary.

The token copy comes from the separate token storage, not directly from the first traversal-record field. Those locations must not be assumed to remain equal after dependency calls.

The function reads Global C again for F-7fa3e55b7211. That call receives:

1. The address of a four-byte temporary work area.
2. The value obtained from this second Global C read.
3. The address of the token copy.
4. The address of the result copy.

The work area has no explicit initialization before this call. The call’s return value is not consumed.

This is a new Global C read, not reuse of the value supplied to F-f35532238f81. The initializer, traversal checks, and any advancement calls execute between the two reads. Their effects on Global C are unknown, so the two argument values may differ.

### Second resolution

F-1b9e6fff9bc9 receives first resolved address minus `28` bytes. Its result becomes the second resolved address.

The container address, conditional field, and final dependency argument are derived from this second result. They are not derived from the earlier resolver result.

### Conditional preparation

The function reads the conditional field. If it is zero, the entire preparation and node-drain branch is skipped.

If the initial read is nonzero, the function proceeds in this order:

1. Reads the conditional field again and uses that value as an address.
2. Reads the four-byte word at that address plus `8` bytes and stores it in a temporary.
3. Reads the conditional field again and copies its current value into another temporary.
4. Calls F-d0acc006efef with the work-area address, container address, and addresses of those two copied values, in that order.
5. Reads the conditional field again.
6. Calls F-88d9235286f8 with the container address, that freshly read field value, and zero.

The copied values reuse the temporary slots previously used for the token and traversal-result copies. The same work area is supplied to both preparation-related calls that take it.

The repeated conditional-field reads are distinct. In particular, F-d0acc006efef may change the field before the read used for F-88d9235286f8. No preservation guarantee permits replacement of that fresh read with the earlier copy.

A nonzero initial condition does not establish that later field values remain nonzero or valid addresses.

### Head-driven node drain

After F-88d9235286f8 returns, the function repeatedly reads the head field.

A zero head ends the drain. Otherwise, the read value identifies the current node. For that node:

1. Its offset-`0` word is read as the successor.
2. The successor is written to the head field.
3. F-ff1bdc376dcd receives the node’s offset-`8` word, its offset-`4` word, and zero.
4. F-ff1bdc376dcd receives the node address, integer `1`, and zero.
5. The head field is read again for the next iteration.

The head update precedes both per-node calls. Both node-field arguments for the first call are obtained before that call begins; their relative read order is not established.

The current node address is retained for the second call. By contrast, the next iteration uses a fresh head-field read rather than a privately retained successor. Dependency changes to the head can therefore change the subsequent drain path.

There is no visible cycle detection, node validation, iteration bound, or explicit error branch.

### Final call

After the conditional branch is skipped or its drain terminates, F-57d76f0801f8 receives second resolved address minus `148` bytes.

If this dependency returns normally, the function returns normally. The final call is not guaranteed to occur after earlier nontermination, invalid memory access, or a dependency’s nonlocal exit.

## Side effects

### Direct effects and observations

The function directly:

- Reads Global A once before the initial object write.
- Reads Global B once after the first resolver returns.
- Reads Global C separately for two calls.
- Overwrites input-object offset `0` with the value read from Global A.
- Overwrites first resolved address minus `176` with the value read from Global B.
- Initializes and updates temporary storage.
- Writes each selected node’s successor into the head field before processing that node.

It does not explicitly restore the two overwritten words. It contains no explicit assignment to the three named globals.

These facts do not prove the globals remain unchanged: dependency effects and possible storage aliasing are not constrained by the supplied evidence.

### Ownership and lifetime

Dependencies receive addresses into caller-associated storage and temporary storage. The evidence establishes neither ownership transfer nor allocation or deallocation semantics.

In particular, the two calls per node must not be described as a guaranteed payload release followed by a guaranteed node free. Those are possible interpretations, not established behavior.

Temporary addresses are supplied during the invocation. Whether dependencies retain them, and whether any such retention would be valid, is unknown.

### Errors and partial progress

There is no explicit error result, rollback path, or visible recovery mechanism.

If execution stops early, preceding writes and dependency effects may already have occurred. Neither atomicity nor guaranteed final cleanup is established.

## Dependencies

| Opaque ID | Observed interface role |
|---|---|
| F-dae8ca024616 | Receives input object plus `176`; returns the first resolved address. Runs before the Global B read. |
| F-f35532238f81 | Receives token storage, the first observed Global C value, and the address array. |
| F-a1ecaae6738a | Receives traversal state and result storage; only return value `1` terminates traversal. |
| F-dcbd5e1e578f | Receives traversal state after every non-`1` check. |
| F-7fa3e55b7211 | Receives work storage, the independently reread Global C value, and copied token/result storage. |
| F-1b9e6fff9bc9 | Receives first resolved address minus `28`; returns the second resolved address. |
| F-d0acc006efef | Receives work storage, the container, and two copied conditional-state values. |
| F-88d9235286f8 | Receives the container, a fresh conditional-field value, and zero. |
| F-ff1bdc376dcd | Receives the two per-node argument sets described above. |
| F-57d76f0801f8 | Receives second resolved address minus `148` after prior processing completes normally. |

Other than resolver results and the traversal check result, dependency return values are not consumed. Dependency bodies and full contracts are unavailable.

## Acceptance examples

These are conditional consequences of the static evidence, not executed tests. Each assumes valid reached storage and normally returning dependencies unless otherwise stated.

### Resolver changes Global B

Suppose Global B initially contains `17`, and F-dae8ca024616 changes it to `29` before returning.

If no intervening effect changes it again, the write at first resolved address minus `176` stores `29`, not the entry-time value `17`.

### Global C differs between calls

Suppose the Global C read for F-f35532238f81 yields `41`. During initialization or traversal, a dependency changes Global C so that the read for F-7fa3e55b7211 yields `53`.

The two calls receive `41` and `53`, respectively. Equality between these arguments is not an acceptance requirement.

### Immediate traversal completion

If the first checker call returns `1`, there is no F-dcbd5e1e578f call. The result and token copies are then made, Global C is read again, and F-7fa3e55b7211 is invoked.

### Noncanonical nonzero checker result

For checker results `2`, `0`, then `1`, there are three checks and two advancement calls. The result `2` does not end traversal.

### Zero conditional field

If the first conditional-field read yields zero, neither preparation dependency is called and the head is not directly inspected by this function. A nonzero head elsewhere in that storage does not independently trigger draining. The final call still follows normal completion.

### One stable node

Suppose the branch is entered, preparation leaves one valid node at the head, its successor is zero, and the per-node calls leave the head unchanged.

The head becomes zero before either per-node call. The first receives the node’s offset-`8` and offset-`4` values with zero; the second receives the node address, `1`, and zero. The next head read terminates the drain.

### Mutation during preparation or draining

If F-d0acc006efef changes the conditional field, F-88d9235286f8 receives the subsequently read value, not automatically the saved copy.

If a per-node dependency changes the head, the next iteration observes that head value rather than automatically following the earlier successor.

## Uncertainties

The following remain **documented_only**:

- Values, semantic roles, mutability rules, and synchronization requirements of Globals A, B, and C.
- Whether dependencies change those globals or other reachable state.
- Complete layouts, allocation extents, alignment, and aliasing relationships.
- Whether dependencies initialize every subsequently consumed temporary.
- Whether the second address-array element is inspected.
- Ownership, release, allocation, and retained-address behavior.
- Arithmetic overflow and invalid-address behavior.
- Concurrency, reentrancy, and atomicity guarantees.
- Failure signaling, exceptions, and nonlocal exits.
- Termination guarantees for either loop.

Teardown and resource release remain inferred intent. They are not substitutes for missing dependency contracts.

## Evidence

This contract is based solely on static inspection of the supplied function representation.

Established evidence includes the three global read locations, separate Global C reads, their placement relative to dependency calls, four-byte accesses, byte displacements, temporary initialization order, exact comparison with `1`, repeated conditional-field reads, and head writes before per-node calls.

Dependency implementations, complete type definitions, allocation metadata, and runtime observations were not supplied. No runtime tests have been performed. The acceptance examples describe conditional behavior, not observed execution.
