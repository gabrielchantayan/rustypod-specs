<!-- Trusted provenance: F-0cea7023bcfc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0cea7023bcfc

## Purpose

F-0cea7023bcfc returns a 32-bit Boolean-like result after conditionally obtaining and inspecting a byte sequence associated with an input object.

The established recognition rule is narrower and less regular than a conventional case-insensitive extension check:

- A preliminary object method must return a nonzero value.
- A subsequently obtained candidate address must be nonzero.
- A separately obtained unsigned 32-bit quantity must be greater than four.
- The final two bytes selected by that quantity must be `b` or `B`, preceded by `4`.
- The third byte from the end must either be lowercase `m`, with no fourth-byte requirement, or uppercase `M`, with a preceding `.`.

Thus, lowercase and uppercase `m` do not have identical acceptance conditions. The apparent intent may be to identify an `.m4b`-like suffix, but that intent does not override the established predicate.

The function returns exactly `1` when the predicate succeeds and exactly `0` on its ordinary nonmatching paths. Memory faults, dependency failures, and abnormal control transfers have no locally established translation into a return value.

## Interface

### Inputs

The interface has four machine-word arguments:

1. **Object address:** a pointer to an object whose first 32-bit word supplies a dispatch-table address.
2. **Initial descriptor word:** an uninterpreted 32-bit value.
3. **Initial temporary word A:** an uninterpreted 32-bit value.
4. **Initial temporary word B:** an uninterpreted 32-bit value.

The available representation uses 32-bit pointer and integer operations. It does not establish a portable interface for wider-pointer environments.

The second through fourth arguments are not independently dereferenced by this function before being placed in local storage. Their meanings depend on the methods and dependencies subsequently called.

### Local storage exposed to callees

Before invoking any dependency, the function establishes two adjacent-word storage regions:

| Region | Offset | Initial contents |
|---|---:|---|
| Primary region | 0 bytes | Object address |
| Primary region | 4 bytes | Initial descriptor word |
| Temporary region | 0 bytes | Initial temporary word A |
| Temporary region | 4 bytes | Initial temporary word B |

The primary region precedes the temporary region in the displayed local layout. These are descriptions of observed storage and addresses passed to callees, not proof of two formally defined eight-byte object types.

Dependencies receive addresses into these local regions. Consequently, later operations may observe values changed through those addresses. In particular, the descriptor word used for the quantity query is not necessarily the original second argument.

The storage remains local to this invocation. Retention of its addresses beyond the invocation is not established as valid.

### Output

The ordinary return value is a 32-bit word:

- `1`: the preliminary gate and byte predicate both succeed.
- `0`: the preliminary gate is zero, or a later predicate fails.

There is no separate local error code, exception mapping, or diagnostic output.

### Preconditions

For defined, normally returning execution, the caller and dependencies must collectively provide:

- A readable object and dispatch table sufficient for the indirect calls described below.
- Callable methods at byte offsets 8 and, when the gate succeeds, 16 in that table.
- Valid behavior for all dependency operations on the supplied local storage.
- A readable candidate range at the exact byte addresses selected by the later arithmetic.

There is no explicit null-object check. A null object address does not constitute an ordinary false result: dispatch is attempted first.

## Behavior

### 1. Establish initial local state

The function stores all four incoming words into the local regions described above before calling the preliminary method.

These stores matter because later callees receive addresses into that storage. The available evidence does not establish that the regions remain unchanged after such calls.

### 2. Invoke the preliminary gate

The function reads the dispatch-table address from the object’s first word, obtains a callable entry at table byte offset 8, and invokes it with the object address.

Its result is interpreted as a 32-bit integer:

- If zero, the function immediately returns `0`.
- If nonzero, processing continues.

On the zero path, neither the second dispatch method nor any of the four named dependencies is called. No local cleanup dependency is invoked for either region.

The gate call itself may have effects. Nothing in this function establishes that it preserves object fields, global state, or externally owned memory.

### 3. Populate or transform temporary storage

After a nonzero gate result, the function resolves the object’s dispatch-table address again and invokes the entry at byte offset 16.

The arguments, in observed order, are:

1. The address of the temporary region.
2. The original object address.

No result from this method is directly tested. Its relevant observable contribution may be writes through the temporary-region address or other side effects.

Resolving the table again is significant: the function does not establish that the dispatch-table pointer remains the same across the gate call.

### 4. Process both local regions

The following calls occur in this order:

1. **F-b9acea4b06ad**, receiving the primary-region address and the temporary-region address.
2. **F-289ec4311c64**, receiving the temporary-region address.
3. **F-44362605ab79**, receiving the primary-region address.
4. **F-d8cbf9aa79a9**, receiving the current 32-bit descriptor word at primary-region offset 4.

The 32-bit result of F-44362605ab79 is subsequently treated as a candidate byte address. The 32-bit result of F-d8cbf9aa79a9 is treated as an unsigned quantity.

The quantity query occurs even if the candidate address is zero. A null candidate therefore suppresses later byte reads, but does not suppress F-d8cbf9aa79a9.

The descriptor word is read for the quantity query after the earlier dependencies have returned. Assuming it equals the original second argument would require an additional dependency guarantee not present here.

Although the roles suggest a candidate-buffer operation and a length query, their association is not independently established. This function does not check that the returned quantity describes the buffer at the candidate address.

### 5. Apply the recognition predicate

The candidate is inspected only when both conditions hold:

- The candidate address is nonzero.
- The unsigned quantity is greater than `4`.

The threshold is strict. A quantity of exactly `4` is rejected without reading suffix bytes.

For an eligible candidate, the function forms an end address by adding the quantity to the candidate address using the displayed 32-bit arithmetic. It then performs short-circuited byte comparisons in this order:

1. Read one byte before the end address. Continue only for `b` or `B`.
2. Read two bytes before the end address. Continue only for `4`.
3. Read three bytes before the end address:
   - Lowercase `m` completes a successful predicate.
   - Uppercase `M` requires a further read.
   - Any other byte fails.
4. Only for uppercase `M`, read four bytes before the end address and require `.`.

The alternatives for `b`/`B` and `m`/`M` are expressed as separate short-circuited comparisons. Static evidence therefore permits a second access to the same byte when the first alternative fails. It does not establish a single cached read for each position or a hardware-level load count.

There is no full-string scan, terminator check, encoding conversion, or general case folding in this function. Only the specified trailing byte positions participate in recognition.

### 6. Release or finalize primary storage and return

After the predicate is decided, F-289ec4311c64 is called with the primary-region address.

- A successful predicate is followed by this call and return value `1`.
- A failed predicate is followed by this call and return value `0`.

This cleanup call does not alter the already-selected predicate result if it returns normally. Its own return value is not tested.

Across a normally completed nonzero-gate path, F-289ec4311c64 is therefore called twice: first for temporary storage, then for primary storage.

## Side effects

### Established local writes and exposed addresses

The function initializes four local 32-bit words and passes addresses of local storage to:

- The dispatch method at table offset 16.
- F-b9acea4b06ad.
- F-44362605ab79.
- F-289ec4311c64.

Those operations may mutate the referenced storage. The exact write sets are not available.

There are no explicit direct writes to the candidate byte sequence in the visible function body. This does not establish that callees leave that sequence unchanged.

### Observable read ordering

The principal ordering is:

1. Initial local stores.
2. Object dispatch-table read and gate-method call.
3. If permitted, a fresh dispatch-table read and second method call.
4. F-b9acea4b06ad.
5. Temporary-region F-289ec4311c64.
6. F-44362605ab79.
7. Descriptor-word read and F-d8cbf9aa79a9.
8. Conditional suffix-byte reads.
9. Primary-region F-289ec4311c64.
10. Return.

Byte reads are short-circuited. A failed final-byte comparison prevents all earlier-position reads; a failed `4` comparison prevents the `m`/`M` reads; lowercase `m` prevents the dot read.

### Ownership and lifetime

Calling F-289ec4311c64 on each region resembles finalization, but its ownership semantics are unknown. The evidence does not establish that it frees memory, decrements a reference count, invalidates aliases, or is safe on arbitrary word pairs.

In particular, the temporary-region call occurs before candidate acquisition and inspection. Successful execution requires whatever lifetime relationships the dependencies implement to remain sufficient for the subsequent accesses. The function itself supplies no independent lifetime protection.

No allocation, ownership transfer, global-state preservation, or release behavior may be assumed for an unknown dependency solely from its placement in this sequence.

### Failure behavior

No local mechanism handles:

- Invalid object or table pointers.
- Invalid callable entries.
- Unreadable suffix addresses.
- Address-arithmetic overflow.
- A callee that fails to return normally.
- Dependency violations of storage or lifetime requirements.

The visible control flow does not establish primary-region cleanup if execution is interrupted before the normal cleanup point.

## Dependencies

| Dependency | Observed interface and use | Not established |
|---|---|---|
| Object method at table offset 8 | Receives the object address; its 32-bit result gates further work | Semantic meaning of the gate, state mutations, preservation guarantees |
| Object method at table offset 16 | Receives temporary-region address, then object address | Exact output type, extent of writes, ownership effects |
| F-b9acea4b06ad | Receives primary-region address, then temporary-region address | Copy, move, conversion, retention, and aliasing semantics |
| F-289ec4311c64 | Receives one local-region address; called first for temporary storage and later for primary storage | Destruction, release, reference counting, and exceptional behavior |
| F-44362605ab79 | Receives primary-region address; result is used as a 32-bit candidate address | Buffer extent, encoding, termination, ownership, stability across later calls |
| F-d8cbf9aa79a9 | Receives the current descriptor word; result is compared as an unsigned 32-bit quantity | Whether it measures the candidate, units, side effects, validity guarantees |

The two indirect methods have no supplied opaque function IDs. They are identified only by their dispatch-table positions.

## Acceptance examples

The following are conditional contract examples, not runtime observations. Except where stated otherwise, assume all calls return normally, the gate is nonzero, the candidate address is nonzero, the quantity is the displayed byte count, arithmetic yields the intended addresses, and all required bytes remain readable during inspection.

| Candidate bytes | Quantity | Result | Reason |
|---|---:|---:|---|
| `x.m4b` | 5 | `1` | Lowercase `m`, then `4`, then lowercase `b` |
| `x.M4B` | 5 | `1` | Uppercase `M` has the required preceding dot |
| `xxm4b` | 5 | `1` | Lowercase `m` does not require a preceding dot |
| `xxM4b` | 5 | `0` | Uppercase `M` requires a dot immediately before it |
| `.m4b` | 4 | `0` | Quantity must be greater than four |
| `.M4B` | 4 | `0` | Same strict threshold |
| `x.m4c` | 5 | `0` | Final byte is neither `b` nor `B` |
| `x.m5b` | 5 | `0` | Second-to-last byte is not `4` |
| `x.n4b` | 5 | `0` | Third-to-last byte is neither `m` nor `M` |

Additional ordering and boundary examples:

- **Zero gate:** returns `0` without invoking the offset-16 method or named dependencies, regardless of the other input words.
- **Null candidate after a nonzero gate:** F-d8cbf9aa79a9 is still called; suffix bytes are not read; primary cleanup occurs; the function returns `0`.
- **Quantity from zero through four:** no suffix bytes are read; primary cleanup occurs; the function returns `0`.
- **Nonmatching final byte:** the second-, third-, and fourth-from-end positions are not inspected.
- **Lowercase `m` with an unreadable fourth-from-end position:** the predicate does not require that position to be read, provided every earlier operation and required read is valid.
- **Uppercase `M` in the same situation:** the function attempts the fourth-from-end read. A false return is not guaranteed if that access faults.
- **Unsigned quantity with its high bit set:** it passes the greater-than-four comparison. Safety and result then depend on the actual address arithmetic and selected memory, not on treating the value as a negative length.

## Uncertainties

### Established static evidence

The supplied body establishes the branch structure, call ordering, local word initialization, unsigned threshold, case-sensitive asymmetry, conditional byte accesses, and ordinary return values described above.

### Inferred intent

The trailing pattern suggests recognition of an `.m4b`-style name or descriptor. The dependency sequence suggests temporary construction followed by finalization. Both interpretations remain inferred intent, not additional acceptance requirements.

### documented_only unknowns

- The actual types and invariants of the two local storage regions.
- The semantic meaning of all four input words beyond their observed uses.
- Whether the candidate address and quantity refer to the same underlying object.
- The quantity’s units outside its use as a byte-address displacement here.
- Ownership, aliasing, allocation, release, and global-state effects of dependencies.
- Whether any dependency can invalidate the candidate before inspection.
- Exact calling-convention details behind the offset-16 method’s output-address argument.
- Target-level consequences of 32-bit address addition overflow. No overflow check is visible; a portable, defined wraparound guarantee is not established by the supplied representation.
- Behavior under concurrent mutation, volatile memory, faults, exceptions, or nonlocal exits.

These unknowns must not be replaced with assumptions that make the function safer or more conventional than the evidence supports.

## Evidence

This contract is based solely on static inspection of the supplied function representation and the supplied opaque dependency mapping.

The evidence directly supports two indirect object calls, four named dependencies, the exposed local-storage layout, the conditional suffix predicate, and the ordinary cleanup and return paths. No dependency implementations or external type definitions were supplied.

No runtime tests have been performed. Acceptance examples are deductions under explicit preconditions, not observed executions.
