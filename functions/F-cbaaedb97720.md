<!-- Trusted provenance: F-cbaaedb97720; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cbaaedb97720

## Purpose

F-cbaaedb97720 updates a two-slot record and a three-slot record, optionally modifying another record reached through the first.

The established behavior is a conditional sequence of four-byte reads and writes, preceded by an unconditional call to F-58820f51cfa6. A value read from the first record’s second slot selects between two update paths.

The apparent intent is to insert a node at one end of a doubly linked structure:

- The first record appears to contain two endpoint references.
- The supplied node receives a zero link, a link to the previous endpoint when applicable, and a reference to the first record.
- In the nonempty case, the previous endpoint receives a reciprocal link to the supplied node.

This interpretation is inferred, not a separately established data-structure contract. The function does not validate that the records form a list, that the endpoints agree, or that the supplied node is detached.

## Interface

**Inputs**

Two address-valued arguments are supplied:

| Input | Role in this contract | Directly accessed layout |
|---|---|---|
| Container address | Address of the two-slot record | Four-byte slots at byte offsets 0 and 4 |
| Node address | Address of the record being attached | Four-byte slots at byte offsets 0, 4, and 8 |

The neutral field names used below are:

| Record | Offset | Name |
|---|---:|---|
| Container | 0 | Endpoint A |
| Container | 4 | Endpoint B |
| Node | 0 | Link A |
| Node | 4 | Link B |
| Node | 8 | Container reference |

These names describe locations, not verified public type definitions.

**Widths and representation**

Every explicit record read or write is four bytes wide. The branch decision tests the entire four-byte Endpoint B value for equality with zero. No signed ordering, link count, capacity, or length calculation is involved.

Several four-byte slots hold address representations. In the nonzero branch, Endpoint A is interpreted as an address and used as the destination of a four-byte write. The supplied container and node addresses are also stored in four-byte slots.

The supplied evidence therefore supports a four-byte address-slot model. It does not establish behavior for addresses that cannot be represented in those slots, or any wider-address adaptation. Byte order and alignment requirements are not established.

**Outputs**

The function has no defined result value. Its results are memory mutations and any effects of F-58820f51cfa6.

**Preconditions for an ordinary successful call**

- Both supplied addresses must permit the accesses made on the selected path.
- The container must support a read at offset 4 and writes at the offsets described below.
- The node must support four-byte writes at offsets 0, 4, and 8.
- If the selected branch uses Endpoint A as an address, the value obtained by that particular read must identify a location permitting a four-byte write.
- Any requirements imposed by F-58820f51cfa6 must also hold.

The direct behavior contains no validation of these conditions. Stronger conditions—such as distinct records, consistent endpoints, or exclusive access—may be necessary for the inferred list operation, but are not enforced here.

## Behavior

### Dependency call and branch selection

F-58820f51cfa6 is called first, with no explicit arguments. No direct container or node memory access is shown before this call.

Only after that call returns does F-cbaaedb97720 read the container’s Endpoint B slot. The four-byte value obtained from this read determines the entire branch selection:

- Zero selects the zero-endpoint path.
- Any nonzero value selects the nonzero-endpoint path.

The branch decision is retained. Endpoint B is not reread to reconsider the branch after subsequent writes.

Consequently, the relevant starting state is the state after the dependency returns, not necessarily the state at entry. The dependency’s effects must not be assumed absent merely because it receives no explicit arguments.

### Zero-endpoint path

When the captured Endpoint B value is zero, the direct accesses occur in this order:

1. Store the node address in the container’s Endpoint B slot.
2. Store the node address in the container’s Endpoint A slot.
3. Store a four-byte zero in the node’s Link B slot.
4. Store a four-byte zero in the node’s Link A slot.
5. Store the container address in the node’s Container reference slot.

The previous Endpoint A value is not read. In particular, a zero Endpoint B selects this path even when Endpoint A contains a nonzero or inconsistent value.

For distinct, nonoverlapping container and node records, successful completion leaves both endpoints referring to the supplied node, both node links zero, and the node’s Container reference referring to the container.

That final-state description is conditional on nonoverlap. The ordered writes remain the applicable description when records overlap.

### Nonzero-endpoint path

When the captured Endpoint B value is nonzero, the direct accesses occur in this order:

1. Store a four-byte zero in the node’s Link A slot.
2. Read the container’s Endpoint A slot.
3. Store the value just read in the node’s Link B slot.
4. Read the container’s Endpoint A slot again.
5. Interpret this second value as an address and store the node address in the four-byte slot at that address.
6. Store the node address in the container’s Endpoint A slot.
7. Store the container address in the node’s Container reference slot.

The two Endpoint A reads are distinct. The function does not retain the first value for both uses. An intervening write to the node’s Link B slot can therefore matter if that slot overlaps Endpoint A.

For stable, distinct, nonoverlapping records, with Endpoint A initially identifying an existing node, successful completion has these effects:

- The supplied node’s Link A becomes zero.
- Its Link B refers to the previous Endpoint A node.
- The four-byte slot at offset 0 of that previous node becomes the supplied node address.
- Endpoint A becomes the supplied node address.
- Endpoint B is not directly written on this path.
- The supplied node’s Container reference becomes the container address.

The captured nonzero Endpoint B value is used only to select the branch. Its address, if it represents one, is not dereferenced by this function.

### Ordering, overlap, and interruption

The sequence above describes the direct accesses established by the supplied source. It is not an assertion of processor-level synchronization, atomicity, or ordering guarantees for concurrent observers.

No overlap checks are performed. If an input record, a node field, or a reached destination overlaps another accessed location, an earlier write can change a later read or overwrite an earlier result. The ordinary list-shaped postconditions must not be applied unconditionally to such cases.

There is no explicit rollback. If execution is interrupted or an access fails after some writes have occurred, the function provides no error-return path or restoration sequence.

## Side effects

**Direct memory effects**

On the zero-endpoint path, there are five explicit four-byte writes: two endpoint writes and three node-field writes.

On the nonzero-endpoint path, there are five explicit four-byte writes: three node-field writes, one write through Endpoint A, and one Endpoint A write. Endpoint B is read but is not directly assigned on this path.

These counts describe write operations, not necessarily distinct destinations. Aliasing can make multiple writes affect the same storage.

**Ownership and lifetime**

The function directly stores references to caller-supplied storage. It contains no explicit allocation, deallocation, reference-count adjustment, ownership check, or detachment operation.

That observation does not establish the effects of F-58820f51cfa6. Its ownership and memory-management behavior is unknown.

For the inferred linked structure to remain usable after return, the records referenced by the resulting links would need suitable lifetimes. The function neither verifies nor extends those lifetimes in its visible direct operations.

**Existing membership**

The supplied node’s three fields are overwritten on both paths. The function does not inspect their old values and does not remove the node from any prior structure.

If the node was already linked elsewhere, the visible operations do not establish that the former structure remains valid. Reinserting a node already present in the same structure is likewise not checked or given special treatment.

**Errors and partial mutation**

There is no explicit status return, rejection branch, or diagnostic. Zero input addresses and inaccessible storage are not checked.

The exact consequence of an invalid access is outside the supplied evidence. It must not be specified as a particular exception, signal, recoverable error, or successful no-op. Earlier writes may already have taken place.

## Dependencies

### F-58820f51cfa6

Established properties:

- Called once before the direct record accesses.
- Receives no explicit arguments.
- Its result, if any, is not consumed.
- Must return normally before the described record updates can occur.

Documented-only unknowns:

- Global, thread-local, or externally reachable state it reads or changes.
- Whether it can affect either supplied record through another reference.
- Allocation, release, ownership, locking, or synchronization behavior.
- Failure, nonreturning behavior, callbacks, or reentrancy.

The function must not be described as protected by a lock, preceded by validation, or free of allocation solely on the basis of this call. No dependency semantics are supplied to support those claims.

## Acceptance examples

These examples are conditional behavioral expectations derived from static evidence. They are not runtime test results. Unless stated otherwise, they assume the dependency returns, storage is valid, addresses fit the four-byte slots, records do not overlap, and no other actor changes the relevant memory during the direct accesses.

### 1. Zero Endpoint B

After the dependency returns, Endpoint B is zero. Endpoint A may contain any four-byte value.

Expected outcome:

- Both container endpoints refer to the supplied node.
- Both node links are zero.
- The node’s Container reference refers to the container.
- The previous Endpoint A value is neither read nor dereferenced.

This example also covers an inconsistent container whose Endpoint A is nonzero while Endpoint B is zero. The function overwrites Endpoint A rather than reporting inconsistency.

### 2. Nonzero Endpoint B and valid Endpoint A

After the dependency returns, Endpoint B is nonzero and Endpoint A identifies an existing writable record.

Expected outcome:

- The supplied node has zero Link A and Link B equal to the previous Endpoint A value.
- The previous Endpoint A record’s offset-0 slot receives the supplied node address.
- Endpoint A receives the supplied node address.
- Endpoint B retains its value.
- The node’s Container reference receives the container address.

No traversal toward Endpoint B is performed.

### 3. Nonzero Endpoint B but zero Endpoint A

The nonzero-endpoint path is selected.

The node’s Link A is first cleared. The zero Endpoint A value is then copied into Link B. The function subsequently attempts a four-byte write through the address obtained from the second Endpoint A read.

Acceptance is not “reject inconsistent container” or “treat as empty.” The established behavior is the attempted access sequence; its platform-specific failure consequence is undocumented.

### 4. Dependency changes the branch-selecting field

Endpoint B is zero on entry, but F-58820f51cfa6 changes it to nonzero before returning.

The nonzero-endpoint path is selected. Conversely, a dependency-induced change from nonzero to zero selects the zero-endpoint path.

This example does not claim that the dependency actually performs either change. It specifies which state controls the branch if it does.

### 5. Supplied node is already Endpoint A

Assume a valid nonzero Endpoint B and that Endpoint A already equals the supplied node address, without overlap with the container.

The node’s Link A is cleared, Link B receives its own address, and the later write through Endpoint A replaces Link A with its own address. Endpoint A remains that same address.

The function does not provide an idempotent “already attached” operation.

### 6. Node fields overlap container fields

No overlap rejection is expected. Writes must be understood in the established sequence, including the second Endpoint A read.

A simplified acceptance rule that always uses the first Endpoint A value as the indirect-write destination would be incorrect for overlap arrangements that change Endpoint A between those reads.

## Uncertainties

The following remain documented_only:

- The semantic names and complete layouts of the container and node types.
- Whether Endpoint A is conventionally called a head or tail.
- The full valid-input contract, including alignment and permitted aliasing.
- Whether callers guarantee consistent endpoints and detached nodes.
- The address representation beyond the observed four-byte slots.
- Thread-safety, atomicity, and synchronization guarantees.
- All unspecified effects of F-58820f51cfa6.
- Platform behavior for invalid addresses or interrupted execution.

The linked-structure interpretation is useful inferred intent. It does not supersede the established access sequence or justify assuming validation, ownership transfer, or invariant repair.

## Evidence

The supplied static body establishes the unconditional dependency call, four-byte field accesses, branch selection from Endpoint B, ordered writes on each path, repeated Endpoint A read on the nonzero path, and absence of an explicit returned status.

No dependency implementation, caller contract, authoritative type declaration, or runtime observation was supplied. No runtime tests have been performed. The acceptance examples are conditional deductions from the static behavior, not claims of observed execution.
