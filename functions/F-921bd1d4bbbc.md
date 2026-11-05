<!-- Trusted provenance: F-921bd1d4bbbc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-921bd1d4bbbc

## Purpose

F-921bd1d4bbbc selects between two operations on a mutable input record:

- If the record’s auxiliary-chain reference is nonzero and an owner-associated flag is clear, traverse the auxiliary chain, invoking F-f9fbf4d34e67 once per visited element, then return.
- Otherwise, invoke preparatory dependencies, decrement an owner-associated count, splice the record out of a linked structure, invoke further dependencies, and make one or two calls to F-8b9249678402.

The reciprocal link updates and endpoint replacements support interpreting the second operation as removal from a doubly linked structure. The application-level purposes of the traversal and dependency calls remain unknown. In particular, the body does not establish destruction, deallocation, resource release, or ownership transfer.

This contract distinguishes visible behavior from inferred structural intent and unresolved dependency semantics. No runtime tests have been performed.

## Interface

### Input and result

The function accepts one address identifying a mutable record. It returns without an explicit value. There is no explicit status result or error-reporting interface.

The supplied representation uses four-byte integer fields for both numerical values and addresses. The following offsets are byte offsets. These observations do not establish a portable pointer representation, endianness, alignment requirement, or calling convention.

### Record layout

| Offset | Width | Observed use |
|---|---:|---|
| +0 | 4 bytes | Owner-associated address |
| +4 | 4 bytes | Forward link, as inferred from splice operations |
| +8 | 4 bytes | Backward link, as inferred from splice operations |
| +16 | 4 bytes | Address through which a flags byte is read late in the removal path |
| +36 | 4 bytes | Auxiliary-chain head |

### Owner-associated layout

| Offset | Width | Observed use |
|---|---:|---|
| +33 | 1 byte | Flags; mask `0x02` participates in path selection |
| +36 | 4 bytes | Count-like field decremented on the removal path |
| +40 | 4 bytes | First linked-structure endpoint |
| +44 | 4 bytes | Last linked-structure endpoint |
| +68 | 4 bytes | Argument value for the final dependency call |
| +72 | 4 bytes | Argument value for the conditional preceding call |
| +84 | Address only | Interior address passed to F-ebd0ee53699f |

Each auxiliary-chain element exposes a four-byte next-reference field at offset +8.

The labels “owner,” “count,” “forward,” “backward,” “first,” and “last” describe structural interpretations, not established application-level types.

### Additional global input

On the removal path, F-ebd0ee53699f receives an argument read from an opaque global data object. That read occurs after F-724e1e872fe7 returns and before F-ebd0ee53699f is invoked.

The supplied body does not establish the global’s declaration, width, meaning, mutability, or ownership significance. In particular, it is not established as a constant. F-724e1e872fe7 has unknown effects and must not be assumed to preserve it.

### Access and lifetime preconditions

The first record access reads offset +36, before the input address is tested for zero. Null input is therefore not a supported safe no-op.

Further requirements depend on the selected path:

- A nonzero auxiliary head requires the retained owner address to support a byte read at +33.
- Each visited auxiliary element must permit a four-byte read at +8.
- Removal requires the retained owner’s count and relevant endpoint fields to be accessible.
- Every nonzero neighbor used by the splice must provide writable reciprocal-link storage.
- The opaque global must remain readable when the argument to F-ebd0ee53699f is evaluated.
- The record must remain accessible for reads following dependency calls.
- The address read from record offset +16 must permit a byte read.
- The owner freshly obtained for the final call must permit a four-byte read at +68.

These are requirements for defined execution, not checks performed by the function. No general pointer, membership, lifetime, or consistency validation is visible.

## Behavior

### Entry and path selection

The function first reads the auxiliary-chain head from record offset +36.

It then tests whether the input address is zero. For a nonzero input, it reads the owner-associated address at record offset +0 and retains that address. The zero-input alternative retains zero, but does not protect the preceding access.

Path selection follows these rules:

- A zero auxiliary head selects removal without reading the owner flags byte.
- A nonzero auxiliary head causes a byte read at retained-owner offset +33.
- If mask `0x02` is clear, the function selects auxiliary traversal.
- If mask `0x02` is set, it selects removal.

Other bits in the owner flags byte do not affect this decision.

### Auxiliary traversal

Traversal starts with the auxiliary head captured at entry. For each visited element, the function reads the next reference at offset +8 before invoking F-f9fbf4d34e67. No explicit argument is supplied to that dependency.

After the call returns, traversal continues using the saved next reference:

- Zero ends traversal and returns from F-921bd1d4bbbc.
- Nonzero identifies the next element to visit.

A terminal element still causes one dependency call: its zero next reference is read before the call, and termination is checked afterward.

The current element’s next reference is not reread after the call. If the dependency changes that field, the change does not redirect the immediately following step. Changes to an element not yet visited can affect its later next-reference read.

This path contains no explicit count update, link write, or removal-path dependency call. It does not explicitly clear the auxiliary head.

No cycle detection or iteration bound is visible. A cycle whose relevant links remain unchanged does not reach the zero-reference termination condition. Whether dependency effects alter such traversal is unresolved.

### Removal: preparatory calls and global argument

The first removal-path call is F-724e1e872fe7, with the input record address.

After that call returns, the function invokes F-ebd0ee53699f with five explicit arguments in this positional order:

1. The retained owner address plus 84 bytes.
2. The value read from the opaque global data object.
3. The input record address.
4. Zero.
5. Zero.

The global-sourced argument is evaluated after F-724e1e872fe7 returns and before F-ebd0ee53699f is invoked. Consequently, a change to that global made by F-724e1e872fe7 can affect the value supplied to F-ebd0ee53699f. The body does not support substituting an entry-time snapshot or an immutable value.

The supplied representation does not establish an evaluation order among the five argument expressions beyond their completion before the invocation. Their positional order is established; their mutual evaluation order is not.

The owner address used for the interior-address argument was retained before both calls. A dependency’s change to record offset +0 does not replace that retained address for the subsequent count and endpoint operations.

Neither dependency has an established preservation guarantee. Later field reads observe the accessible values present when those reads occur, not necessarily the entry values.

### Removal: count update

After both preparatory calls return, the function reads the four-byte count-like field at retained-owner offset +36, subtracts one, and writes the result to that location.

There is no visible positivity, membership, or consistency check. Zero is not treated specially: subtracting one from an ordinary signed value of zero produces negative one.

The supplied representation expresses this operation using signed integer arithmetic. For signed 32-bit values greater than the minimum representable value, the result is the prior value minus one. The behavior of decrementing the minimum signed value is not established here; unconditional modulo-\(2^{32}\) wrapping must not be assumed.

### Removal: ordered splice

The splice has two stages, after the count update.

**Preceding-side stage:**

- Read the record’s backward link at +8 to select the destination.
- If it is zero, write the record’s forward link to retained-owner offset +40.
- Otherwise, write the record’s forward link to offset +4 of the backward neighbor.

**Following-side stage:**

- Read the record’s forward link at +4 to select the destination.
- If it is zero, write the record’s backward link to retained-owner offset +44.
- Otherwise, write the record’s backward link to offset +8 of the forward neighbor.

These operations are consistent with unlinking the record from a doubly linked structure. The function does not verify that a neighbor points back to the record, and does not explicitly clear either link in the record.

The links are read during the splice, rather than all being captured at entry. An unusual overlap between the first-stage destination and a subsequently read field can affect the second stage. The ordinary unlinking interpretation therefore assumes a non-overlapping structural layout and stable access.

The supplied representation establishes the order of the two stages, but not an additional evaluation order among independent reads within an assignment. No synchronization with concurrent mutation is visible.

### Removal: later dependencies and indirect flag

After both splice stages, the function calls:

1. F-49e78cd0d276 with the retained owner address.
2. F-53c70d0f97c5 with the input record address.

After F-53c70d0f97c5 returns, the function reads the address at record offset +16, then reads a byte through that address.

If mask `0x01` is set in that byte, it reads the four-byte value at retained-owner offset +72 and calls F-8b9249678402 with that value as its sole explicit argument. If the mask is clear, that call is skipped. Other bits do not affect this decision.

The flag is evaluated at this late point. It is not an entry-time snapshot, and preceding dependencies can affect either the address stored at +16 or the byte reached through it.

### Removal: final call

After the optional call, or immediately after deciding to skip it, the function freshly reads the owner address from record offset +0. It reads the four-byte value at offset +68 of that owner and invokes F-8b9249678402 with:

1. The value obtained from that owner’s +68 field.
2. The input record address.

The function then returns.

This fresh owner lookup is distinct from the retained owner used for the count, endpoints, and optional +72 argument. A preceding dependency—including the optional invocation—can change which owner supplies the final argument, provided required storage remains accessible.

## Side effects

The traversal path contains no explicit memory writes. Its effects include ordered chain reads and any effects of F-f9fbf4d34e67.

The removal path explicitly writes:

- The retained owner’s count-like field.
- Either the retained owner’s first endpoint or the backward neighbor’s forward link.
- Either the retained owner’s last endpoint or the forward neighbor’s backward link.

These writes precede F-49e78cd0d276 and F-53c70d0f97c5.

Removal also reads the opaque global between the first two dependency invocations. The body does not explicitly write that global, but dependencies may modify it. No preservation of globals or structural fields may be assumed across calls.

No rollback is visible. If execution fails or ceases to return after a count or link write, the body supplies no recovery operation.

No allocation, deallocation, resource release, or ownership transfer is directly established. In particular, F-53c70d0f97c5 must not be classified as unconditional destruction merely from its position: subsequent operations still read the record. Required lifetime guarantees must come from the surrounding contracts.

## Dependencies

| Opaque ID | Invocation and ordering | Unresolved behavior |
|---|---|---|
| F-f9fbf4d34e67 | Once per visited auxiliary element, after reading that element’s next reference; no explicit arguments | Ambient inputs, effects, ownership, and termination |
| F-724e1e872fe7 | First removal-path call; receives the record | Effects on the record, owner, opaque global, other state, and lifetimes |
| F-ebd0ee53699f | Receives retained-owner +84, the opaque global-sourced value evaluated after F-724e1e872fe7 returns, the record, zero, zero | Global argument’s type and meaning; callee effects and ownership |
| F-49e78cd0d276 | After count and splice writes; receives retained owner | Effects and preservation guarantees |
| F-53c70d0f97c5 | After F-49e78cd0d276; receives record | Effects and record lifetime guarantees |
| F-8b9249678402 | Conditional one-argument invocation followed by a final two-argument invocation on the removal path | Exact prototype, argument interpretation, and effects |

No dependency return value is consumed by the visible body. No visible branch checks a dependency success or failure result. Different explicit argument counts for F-8b9249678402 do not establish its true interface.

## Acceptance examples

These are conditional consequences of static evidence, not runtime results. Unless stated otherwise, they assume accessible storage, returning dependencies, ordinary non-overlapping layouts, and no relevant dependency or concurrent mutation.

### Finite auxiliary chain

Given a nonzero auxiliary head, owner mask `0x02` clear, and two elements ending in zero:

- Each element’s next reference is read before its corresponding F-f9fbf4d34e67 call.
- Exactly two such calls occur.
- The function returns after the second call.
- No explicit count or splice write occurs.

If the first call changes the first element’s already-read next reference, traversal still proceeds using the saved reference.

### Removal selection

With a zero auxiliary head, removal proceeds without a branch-selection read at owner offset +33. This does not make an invalid owner safe.

With a nonzero auxiliary head and owner mask `0x02` set, removal proceeds without traversing the auxiliary chain.

### Global changed by the first dependency

Suppose the opaque global initially contains one value, and F-724e1e872fe7 changes it to another value before returning.

If no subsequent change intervenes, F-ebd0ee53699f receives the latter value as its second argument. Supplying the earlier value merely because it existed at entry would not match the visible ordering.

This example does not imply a particular global width, value type, or interpretation.

### Interior record

Given nonzero backward and forward links and a count of seven:

- The count becomes six.
- The backward neighbor’s forward link receives the record’s forward link.
- The forward neighbor’s backward link receives the record’s backward link.
- Neither owner endpoint is explicitly written by the splice.
- The record’s links are not explicitly cleared.

### Sole record and count boundary

Given both links zero and a count of one, the count becomes zero and both owner endpoints are written as zero. The later calls and flag processing still occur.

Given an ordinary signed count of zero, the decrement stores negative one; there is no zero-count rejection. No result is specified here for decrementing the minimum signed 32-bit value.

### Late flag selection

If F-53c70d0f97c5 leaves the indirectly addressed flags byte with mask `0x01` set, the optional F-8b9249678402 call uses retained-owner offset +72 and precedes the final invocation.

If that mask is clear at the actual read, the optional call is skipped, irrespective of an earlier flag value.

### Owner replacement

Suppose a dependency changes record offset +0 to another valid owner without invalidating required storage:

- Count and endpoint operations use the retained entry owner.
- The optional argument comes from the retained owner’s +72 field.
- The final argument comes from +68 of the owner present when the final fresh lookup occurs.

### Null input

Null input does not promise a successful no-op, error result, or controlled fault. The initial record access precedes the null test.

## Uncertainties

The following remain **documented_only**:

- Exact dependency prototypes and calling convention.
- The opaque global’s declaration, width, representation, meaning, and mutation rules.
- Whether dependencies modify that global or any other relevant state.
- Evaluation order among independent argument expressions and within assignments where not established by sequencing.
- Application-level meanings of the count, flags, chain, and final-call values.
- Ownership and lifetime obligations across every dependency call.
- Signed arithmetic behavior at the decrement overflow boundary.
- Endianness, alignment, and address-arithmetic boundary behavior.
- Fault, exception, and nonreturning-call behavior.
- Behavior under malformed pointers, inconsistent links, overlapping objects, or concurrent mutation.
- Whether external invariants exclude auxiliary cycles.

Inferred intent is limited to traversal and doubly linked removal. It does not establish notification, destruction, finalization, or resource-release semantics.

## Evidence

The supplied static body establishes the field accesses, masks, explicit writes, positional call arguments, call sequencing, and lack of an explicit return value.

The decisive ordering observations are:

- The auxiliary-head read precedes the null test.
- Each auxiliary next-reference read precedes its corresponding dependency call.
- F-724e1e872fe7 returns before the opaque global is read for F-ebd0ee53699f.
- That global read occurs before F-ebd0ee53699f is invoked; it is not established as a fixed or entry-captured value.
- Both preparatory calls precede the count decrement.
- The count write precedes the preceding-side splice, which precedes the following-side splice.
- Both splice stages precede F-49e78cd0d276 and F-53c70d0f97c5.
- The indirect flags read follows F-53c70d0f97c5.
- The final owner lookup occurs after the optional call, if taken, and is fresh rather than retained from entry.

No global declaration, dependency bodies, runtime traces, or test results were supplied. All acceptance examples are conditional static specifications, not observed executions.
