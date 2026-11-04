<!-- Trusted provenance: F-f9f9ca8db028; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f9f9ca8db028

## Purpose

F-f9f9ca8db028 performs two successive, independent traversals of a pointer-linked structure and writes two 32-bit results to caller-supplied storage. Both traversals use F-bcefa5957e76 to choose between links at node offsets +0x08 and +0x0C, but they reverse the order of two comparison arguments and use different rules for updating their saved results.

The established behavior is a pair of searches with saved pointer-like values. A tree-search or range-search purpose is plausible, but neither ordering semantics nor conventional lower-bound or upper-bound behavior is established. In particular, a saved result is sometimes replaced by the **destination of a followed link**, not by the node at which the comparison occurred. A zero link can therefore become a final result.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

The function receives four arguments:

| Argument role | Representation | Contract |
|---|---|---|
| Output location | Address of writable 32-bit storage | Receives two consecutive 32-bit values, at offsets +0x00 and +0x04. Existing contents are not read by this function. |
| Context base | 32-bit address-like value | Identifies a structure containing a header reference and an embedded comparison-state address. |
| Search operand | Opaque 32-bit value | Passed unchanged to the dependency, in different argument positions during the two traversals. |
| Auxiliary operand | Opaque 32-bit value | Passed unchanged as the dependency’s fourth argument on every invocation. |

There is no explicit return value. The observable result is the pair of output writes, together with any effects of dependency calls.

The supplied representation uses 32-bit integers and four-byte fields for addresses, links, operands, and output entries. A replacement implementation must preserve the distinction between 32-bit field values and any wider native host pointers. No 64-bit structure layout is evidenced.

### Referenced layout

The function accesses the following locations:

| Base | Offset | Meaning established by use |
|---|---:|---|
| Context | +0x10 | A 32-bit reference to a header-like object |
| Context | +0x19 | Address supplied as the dependency’s first argument |
| Header | +0x04 | A 32-bit starting-node reference |
| Node | +0x08 | A 32-bit outgoing link |
| Node | +0x0C | A second 32-bit outgoing link |
| Node | +0x10 | Address supplied as a comparison operand |
| Output | +0x00 | First 32-bit result |
| Output | +0x04 | Second 32-bit result |

Offsets are byte offsets. The context address at +0x19 is passed directly; the function does not load a pointer from that location. Likewise, the node address at +0x10 is passed directly rather than dereferenced by this function.

The size and interpretation of the objects beginning at those two addresses are unknown. Their required accessibility depends on F-bcefa5957e76.

### Preconditions

For a normally completing call:

- The context field at +0x10 must be readable whenever accessed.
- Each header used for a starting-node load must permit a four-byte read at +0x04.
- Every nonzero visited node must support the selected four-byte link read after the corresponding dependency call.
- The output location must support two consecutive four-byte writes.
- All dependency invocations must satisfy that dependency’s own preconditions.
- Both traversals must eventually reach a zero node reference.

There are no explicit null, alignment, bounds, cycle, or output-capacity checks. Zero is recognized as an end marker for a **current node**, not as a generally valid header or context address.

## Behavior

### First traversal initialization

The function reads the context’s header reference and then reads the starting-node reference at that header’s +0x04 field.

It subsequently reads the context’s header reference again and saves this second value as the first result’s initial candidate. These are separate source-level reads. The contract does not require the two header-reference reads to return the same value if memory changes between them.

Thus, when the initial node is zero, the first result remains the separately acquired header reference.

### First traversal

For each nonzero current node, the function invokes F-bcefa5957e76 with these five arguments, in order:

1. Context address plus 0x19.
2. Current-node address plus 0x10.
3. Search operand.
4. Auxiliary operand.
5. Zero.

The return value is tested only for equality with zero. Positive and negative nonzero values follow the same branch.

| Dependency result | Selected node link | First candidate update |
|---|---|---|
| Zero | Link at +0x08 | Replace the candidate with the loaded link value |
| Nonzero | Link at +0x0C | Leave the candidate unchanged |

In either case, the selected link value becomes the next current node. A zero next-node value ends the traversal.

The link is read **after** the dependency returns. If the dependency changes the selected link, the subsequent read observes the value then present, subject to the underlying memory execution model. The branch does not use a link value cached before the call.

The candidate update on a zero return uses the newly loaded +0x08 link value. It does not save the current node. Consequently, taking a zero-return branch whose +0x08 link is zero sets the first candidate to zero immediately before termination.

### Second traversal initialization

After the first traversal finishes, the function acquires a new starting point:

1. It reads the context’s header reference and then that header’s starting-node field at +0x04.
2. It reads the context’s header reference again and saves this value as the second result’s initial candidate.

The second traversal does not reuse the first traversal’s root or initial header value. Changes made during the first traversal can therefore affect the second traversal’s initial state.

The first result remains saved while the second traversal runs. It is not written to the output yet.

### Second traversal

For each nonzero current node, the function invokes F-bcefa5957e76 with these arguments:

1. Context address plus 0x19.
2. Search operand.
3. Current-node address plus 0x10.
4. Auxiliary operand.
5. Zero.

The search operand and node-derived address exchange positions relative to the first traversal. No symmetry or antisymmetry of the dependency is assumed.

| Dependency result | Selected node link | Second candidate update |
|---|---|---|
| Zero | Link at +0x0C | Leave the candidate unchanged |
| Nonzero | Link at +0x08 | Replace the candidate with the loaded link value |

Again, the selected link is read after the call, and its value becomes the next current node. Zero terminates the traversal.

A nonzero dependency return followed by a zero +0x08 link sets the second candidate to zero. A zero return followed by a zero +0x0C link terminates without changing the candidate.

### Output publication

Only after both traversals terminate does the function write its results:

1. The first candidate is stored at output offset +0x00.
2. The second candidate is stored at output offset +0x04.

There is no dependency invocation between these stores and no further traversal read after output publication begins. The stores are separate operations, not an evidenced atomic pair.

Output storage overlapping the traversed data is not explicitly rejected. The function’s own output writes occur too late to influence either completed traversal. This does not establish that such overlap is safe for the dependency or for other observers.

### Arithmetic and termination

All stated offsets are byte displacements. Candidate assignment preserves the selected 32-bit value, including zero; there is no normalization, range check, or conversion to a node payload.

No iteration limit, visited-node set, or cycle detection is present. A cyclic path may prevent completion. Whether a particular cycle is followed depends on dependency results and any intervening mutation.

Address-wrap behavior and invalid-address consequences are not established. Valid-address preconditions must cover the required additions and memory accesses rather than treating overflow as supported behavior.

## Side effects

The only direct memory writes visible in this function are the two final output stores.

F-bcefa5957e76 may have additional effects. Its name and call shape do not establish purity, read-only behavior, global-state preservation, allocation behavior, or ownership rules. In particular:

- It may modify comparison state reachable through context +0x19.
- It may modify headers, nodes, or unrelated state.
- It may change the link subsequently read by the caller.
- Its behavior may depend on call order or previous invocations.
- Its effects during the first traversal may change the second traversal.

These are possibilities, not established dependency behavior.

No direct allocation, deallocation, reference-count adjustment, or ownership transfer is visible. Output values are copied references or link values; the function does not establish that they identify live nodes, owned objects, or objects safe to dereference.

There is no explicit error result or recovery path. If a dependency call fails to return, traversal does not proceed past that call. If execution fails before output publication, this function has not yet performed either output store. If the first output store succeeds but the second fails, partial publication is possible; no rollback mechanism is visible.

## Dependencies

### F-bcefa5957e76

This is the only directly invoked dependency.

Its required calling interface consists of five arguments: the context-derived address, two operands whose order depends on traversal phase, the auxiliary operand, and zero. The caller consumes a 32-bit result as a zero/nonzero discriminator.

The following are not established:

- Whether the result represents a Boolean, ordering, equality, or another predicate.
- Whether swapping operands reverses or otherwise relates results.
- Whether the search operand is an address, scalar, handle, or encoded value.
- Which bytes the dependency reads or writes.
- Whether it retains arguments, allocates memory, or releases objects.
- Whether it can report failures through mechanisms other than its return value.

A behavioral substitute must preserve argument order and call sequencing without replacing the dependency with an assumed standard comparison.

## Acceptance examples

These are conditional static traces, not observed executions. Unless stated otherwise, assume stable memory, valid addresses, a normally returning dependency, and a header reference **H** with starting node **N**.

### Empty structure

If the starting-node field is zero in both initializations and all header-reference reads yield H:

- Neither traversal invokes the dependency.
- Both candidates remain H.
- Output receives H followed by H.

An empty structure therefore does not necessarily produce two zero values.

### One node, zero results in both phases

Suppose both links of N are zero and the dependency returns zero in each phase.

- The first traversal reads N’s +0x08 link and replaces its candidate with zero.
- The second traversal reads N’s +0x0C link and retains H.
- Output is zero followed by H.

### One node, nonzero results in both phases

With the same zero links, suppose both calls return nonzero.

- The first traversal follows +0x0C without replacing H.
- The second traversal follows +0x08 and replaces its candidate with zero.
- Output is H followed by zero.

Any nonzero result, including a negative 32-bit result, has this branch behavior.

### Saving a destination rather than the compared node

Suppose N’s +0x08 link is another node **M**.

In the first traversal, a zero return at N saves M. If the next call at M returns nonzero and M’s +0x0C link is zero, the traversal ends with M saved.

This outcome depends on saving the followed link. Replacing the saved result with N would violate the established behavior.

### Dependency mutation before link selection

Suppose the first-phase call at N returns zero and changes N’s +0x08 link from M to zero before returning.

The function reads that link afterward, saves zero, and terminates the first traversal. It does not continue to M based on the former link value.

### Fresh initialization between phases

Suppose first-phase dependency activity changes the context’s header reference from H to another valid header **J**, whose starting-node field is zero. Assume the first traversal otherwise completes.

The second traversal uses fresh header reads. If both yield J, it makes no dependency calls and saves J, regardless of the first traversal’s initial header.

## Uncertainties

The following are **documented_only**:

- **Semantic intent:** The layout resembles a searchable linked structure, but conventional tree invariants and bound semantics are unproven.
- **Layout extent:** Only the listed offsets are evidenced. Full object sizes, payload formats, packing, byte order, and alignment guarantees are unknown.
- **Dependency semantics:** No purity, ownership, comparison law, error convention, or memory-lifetime guarantee is supplied.
- **Concurrency:** Source-level access order is described, but atomicity, synchronization, compiler treatment of concurrent mutation, and hardware visibility are unspecified.
- **Invalid inputs:** Null contexts, invalid headers, dangling nodes, arithmetic wraparound, and insufficient output storage have no defined recovery behavior in the supplied evidence.
- **Result lifetime:** A saved nonzero value need not still identify a valid object when written, especially if dependency calls mutate or release underlying storage.
- **Termination:** No bound exists without additional structural and dependency assumptions.

These unknowns must not be replaced with inferred library conventions.

## Evidence

Established static evidence consists of two sequential traversal loops, four source-level acquisitions of the context’s header reference, phase-specific dependency argument ordering, zero/nonzero branch conditions, post-call link reads, destination-based candidate updates, and two ordered final 32-bit output writes.

The search/range interpretation is inferred intent only. All acceptance examples are conditional consequences of the supplied control flow and stated assumptions. No runtime execution, dependency implementation, external layout definition, or independent memory-model evidence was available.
