<!-- Trusted provenance: F-bd4a91a0cf4e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bd4a91a0cf4e

## Purpose

F-bd4a91a0cf4e conditionally obtains and initializes a 12-byte primary record, optionally obtains and attaches an 8-byte auxiliary record, and publishes the primary address through a caller-supplied output slot.

A nonzero creation value enables the primary-record path. A nonzero auxiliary selector enables the auxiliary-record path only when creation is already enabled.

On normal completion, the function returns the original output-slot address, not the address of either record.

These are structural observations. The application meaning of the records, the meaning of their fields, and the ownership consequences of publication are not established. The dependencies’ effects must not be limited to allocation or assumed to preserve previously initialized memory.

## Interface

### Inputs

| Input | Representation established by the supplied evidence | Use |
|---|---|---|
| Output slot | Address of a writable four-byte integer slot | Written with zero on entry; written with the encoded primary address on a normally completed creation path. |
| Creation value | 32-bit integer | Compared with zero; when nonzero, copied into the primary field at offset 0. |
| Auxiliary selector | 32-bit integer | Compared with zero only on the enabled creation path; not stored in either record. |

Both positive and negative nonzero integer values select the corresponding enabled path. The function shows no sign-based distinction, range restriction, or normalization.

The output slot’s previous contents are not read by this function. A valid output slot is required even when the creation value is zero.

### Return value

Every normal return yields the original output-slot address.

There is no separate success indicator or error code. On a normally completed creation path, the caller obtains the encoded primary address from the output slot.

### Primary-record layout

The first storage request has size 12 bytes. The function writes three consecutive four-byte fields:

| Byte offset | Width | Explicit initialization |
|---|---:|---|
| 0 | 4 bytes | Creation value |
| 4 | 4 bytes | Integer one |
| 8 | 4 bytes | Zero |

If auxiliary creation is selected, offset 8 is subsequently overwritten with the encoded auxiliary address.

The constant one has no established application-level meaning. It must not be specified as a reference count, version, flag, or ownership marker without additional evidence.

### Auxiliary-record layout

The conditional second storage request has size 8 bytes:

| Byte offset | Width | Explicit initialization |
|---|---:|---|
| 0 | 4 bytes | Zero |
| 4 | 4 bytes | Zero |

No additional fields, padding requirements, or terminators are established.

### Widths and address representation

The supplied representation uses four-byte integer fields and stores record addresses through those fields. This supports a 32-bit address representation for the operations shown, but does not independently establish the complete platform ABI.

Lossless address publication requires returned addresses to be representable by the four-byte address encoding used here. The evidence does not define conversion behavior for arbitrary wider addresses. Byte order is also unspecified.

### Preconditions

Defined execution of the shown memory operations requires:

- An output slot writable and appropriately aligned for a four-byte store.
- Storage returned by each successful dependency call that supports the subsequent writes at the stated offsets.
- Valid storage for as long as this function accesses it.
- Addresses compatible with the observed address-to-integer storage.
- Satisfaction of any additional dependency requirements, which are unavailable.

There is no visible null-output guard or allocation-result validation. Freshness and nonoverlap of returned storage are not established by the dependency interface alone.

## Behavior

### Entry and disabled creation

The first explicit memory operation is a four-byte zero store to the output slot. This precedes the creation-value test and all dependency calls.

If the creation value is zero:

- No storage request is made.
- F-b00260d1087a is not called.
- The auxiliary selector does not affect execution.
- The function returns the output-slot address.

In ordinary execution without external interference, this path leaves the output slot containing zero. Its previous contents are irrelevant.

### Primary initialization

If the creation value is nonzero, the function calls F-8157a180d901 with a size argument of 12 bytes.

After that call returns normally, the function uses its result as the primary address and performs these four-byte stores in order:

1. The creation value at primary offset 0.
2. Integer one at primary offset 4.
3. Zero at primary offset 8.

The function does not explicitly read the returned storage before these stores.

If the auxiliary selector is zero, the function then writes the encoded primary address to the output slot and returns the output-slot address. No further dependency call occurs on this path.

With valid, nonoverlapping storage and no external interference, the primary-only path therefore ends with primary fields containing the creation value, one, and zero.

### Auxiliary initialization and attachment

If both integer inputs are nonzero, the function calls F-8157a180d901 a second time, requesting 8 bytes. This call occurs after all three initial primary-field stores.

The second call is an interference boundary. Its implementation is unavailable; it may modify accessible memory, including the already initialized primary fields. Distinct returned allocations do not prove that this dependency preserves the primary record.

After the second call returns normally, the function:

1. Stores zero at auxiliary offset 0.
2. Stores zero at auxiliary offset 4.
3. Stores the encoded auxiliary address at primary offset 8.
4. Calls F-b00260d1087a with no explicit arguments.
5. After that call returns normally, stores the encoded primary address in the output slot.
6. Returns the output-slot address.

The auxiliary address is thus attached before F-b00260d1087a runs. The primary address is explicitly published through the output slot only after that call returns.

No explicit argument to F-b00260d1087a conveys either record address. Whether it can discover or affect the records through implicit state is unknown.

### Initialization writes versus final contents

The explicit stores and their order are established. Final record contents require additional preservation assumptions.

On the auxiliary path:

- Primary offsets 0 and 4 are written before the second F-8157a180d901 call and are not explicitly rewritten afterward. Either that call or F-b00260d1087a could change them.
- Primary offset 8 is initially zeroed before the second storage request, then explicitly overwritten with the auxiliary address after that request returns. F-b00260d1087a runs after this overwrite and could change it.
- Both auxiliary fields are explicitly zeroed after the second storage request returns. F-b00260d1087a could subsequently change them.

Consequently, assuming only that F-b00260d1087a leaves both records unchanged is insufficient to guarantee the complete initialized primary state. The second F-8157a180d901 call must also preserve primary offsets 0 and 4 for those values to remain established.

The output slot has a separate ordering guarantee: the function performs its final primary-address store after all dependency calls on a normally completed creation path. Its initial zero store does not imply that the slot remains zero during dependency execution.

### Reads, arithmetic, and conversions

The function tests the creation value before deciding whether to request primary storage. It tests the auxiliary selector only on the enabled creation path.

It performs no explicit reads of previous output-slot or record contents. Dependency-internal reads are unknown.

The storage sizes are fixed constants, 12 and 8 bytes. Neither input participates in size arithmetic, multiplication, count accumulation, or offset selection. The creation value is copied into a four-byte field; the auxiliary selector contributes only its zero-versus-nonzero condition.

The material width issue is address encoding, not input-dependent size overflow.

## Side effects

### Explicit writes and calls

Depending on the inputs, the function performs:

| Condition | Storage requests | Other dependency calls | Explicit output-slot stores |
|---|---|---|---|
| Creation value zero | None | None | Zero |
| Creation value nonzero, auxiliary selector zero | 12 bytes | None | Zero, then primary address |
| Both nonzero | 12 bytes, then 8 bytes | F-b00260d1087a once | Zero, then primary address |

On enabled paths it initializes all three primary fields. On the auxiliary path it also initializes both auxiliary fields and overwrites the primary link field.

These describe this function’s operations, not the complete effects of its dependencies. No global-state preservation guarantee is established.

### Ownership and lifetime

There is no explicit release, destruction, rollback, or cleanup operation in the supplied function.

Publication makes an encoded address available to the caller. It does not establish:

- Ownership transfer.
- A caller obligation to release either record.
- A release API.
- The lifetime of either storage region.
- Whether either dependency retains or releases storage.

Construction of a primary object with an optional auxiliary object is a plausible intent, not an established ownership contract.

### Errors and interrupted execution

There is no explicit error return or branch handling an unusable storage result. If a dependency returns an address unsuitable for the next write, the function proceeds to use it. The resulting fault or other behavior is environment-dependent and unspecified here.

If a dependency fails to return normally, subsequent operations are not reached:

- Interruption during the first storage request precedes primary initialization and final publication.
- Interruption during the second request follows the three initial primary stores but precedes auxiliary initialization and attachment.
- Interruption during F-b00260d1087a follows auxiliary initialization and attachment but precedes final publication.

No explicit cleanup is shown on these interrupted paths. Neither an unchanged output slot nor preservation of earlier record writes can be inferred across an unknown dependency.

## Dependencies

### F-8157a180d901

**Established use:** Called with one size argument; its returned address is used as writable storage.

**Call pattern:** A 12-byte request when creation is enabled, followed by an 8-byte request only when auxiliary creation is also selected.

**Unknown contract:** Freshness, alignment, zero-initialization, failure convention, lifetime, ownership, global-state effects, and behavior on abnormal completion.

The second invocation occurs after primary initialization. Its possible effects on the existing primary record must remain part of the contract uncertainty. Returning distinct storage does not imply preservation of other accessible memory.

The caller explicitly writes every byte in the two described layouts, assuming ordinary four-byte stores. This does not require pre-zeroed storage.

### F-b00260d1087a

**Established use:** Called once with no explicit arguments when both integer inputs are nonzero. No returned value is consumed.

**Ordering:** After auxiliary initialization and attachment; before final publication through the output slot.

**Unknown contract:** Purpose, implicit inputs, memory and global-state effects, ownership effects, and failure or non-return behavior.

Its position in the sequence does not establish that it validates, registers, retains, releases, or otherwise processes either record.

## Acceptance examples

These are conditional static examples, not executed tests. Unless an example addresses invalid storage, assume normal dependency returns, representable addresses, valid and distinct storage regions that do not overlap the output slot, and no concurrent external writes. These assumptions do not by themselves restrict dependency side effects.

### Creation disabled

Given any previous output-slot value, a creation value of zero, and any auxiliary selector:

- The function writes zero to the output slot.
- Neither dependency is called.
- The returned address is the output-slot address.
- The output slot contains zero on normal completion.

### Primary record only

Given creation value 23 and auxiliary selector zero, with the 12-byte request returning address A:

- The output slot is first written with zero.
- One 12-byte request occurs.
- The function writes 23, one, and zero at primary offsets 0, 4, and 8, respectively.
- F-b00260d1087a is not called.
- The output slot is then written with the encoding of A.
- The return value is the output-slot address, not A.

Because no dependency runs after primary initialization on this path, the stated nonoverlap and no-external-interference conditions suffice for those primary values to remain at normal return.

### Auxiliary path: unconditional explicit operations

Given both inputs nonzero, with the requests returning primary address A and auxiliary address B:

- The request sizes occur in order: 12 bytes, then 8 bytes.
- All three initial primary stores precede the second request.
- After the second request returns, both auxiliary zero stores precede attachment of B at primary offset 8.
- F-b00260d1087a runs after attachment.
- The final explicit output-slot store encodes A and occurs after that call returns.

This example guarantees writes and ordering, not final record values.

### Auxiliary path: conditional final values

Additionally assume:

- The second F-8157a180d901 call preserves the primary fields at offsets 0 and 4.
- F-b00260d1087a preserves both records and their validity.

Then, on normal completion, the primary fields contain the creation value, one, and the encoded address B; both auxiliary fields contain zero.

Preservation of the primary field at offset 8 across the second storage request is unnecessary for this final-state example because the function overwrites that field with B afterward.

### Dependency changes a primary field

Suppose the second F-8157a180d901 call changes primary offset 4 from one to another value, and F-b00260d1087a subsequently leaves both records unchanged.

The function does not restore primary offset 4. Its final value is therefore not guaranteed to be one. This remains true even with valid, distinct, nonoverlapping allocations.

### Negative values and invalid storage

A creation value of −7 and any negative nonzero auxiliary selector enable both paths. The function writes −7 into the first primary field in the supplied 32-bit representation; it does not store the selector.

If the first storage request instead returns an unusable address, there is no visible graceful-failure branch. Requiring an error code or a guaranteed zero output on that event would exceed the evidence.

## Uncertainties

The following remain **documented_only**:

- Application meaning of either record or any field.
- Complete ABI, byte order, and pointer-conversion rules.
- Dependency guarantees concerning allocation, alignment, freshness, lifetime, and failure.
- Ownership, release obligations, and cleanup after partial execution.
- Dependency access to output storage, records, or global state.
- Preservation of primary fields across the second storage request.
- The purpose and effects of F-b00260d1087a.
- Supported aliasing arrangements.
- Concurrency, reentrancy, synchronization, and safe-publication requirements.
- Runtime behavior after invalid memory access or dependency failure.

## Evidence

Established static evidence consists of the supplied function body and opaque dependency mapping: the input zero tests, fixed request sizes, four-byte field layout, explicit store order, dependency-call order, final address publication, and return of the original output-slot address.

The distinction between initialization and final state follows from the placement of unknown dependency calls between writes and return. In particular, primary offsets 0 and 4 are initialized before the second storage request and never explicitly restored afterward.

Object-construction terminology describes inferred intent only. No dependency implementations, caller contracts, allocator guarantees, or runtime observations were supplied. No runtime tests have been performed.
