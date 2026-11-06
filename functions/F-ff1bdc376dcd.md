<!-- Trusted provenance: F-ff1bdc376dcd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ff1bdc376dcd

## Purpose

F-ff1bdc376dcd accepts a nullable address, validates metadata immediately preceding a non-null address, reduces several counters in a selected state object, and passes the metadata address to a dependency before a final state-related call.

The established behavior is metadata validation and ordered accounting updates. The arrangement is consistent with releasing an accounted allocation, but allocation ownership, actual reclamation, synchronization, and the meanings of the accounting categories are not established by this function alone.

A zero input is a complete no-op. A nonzero input whose initial metadata word is zero or has its least significant bit set takes a non-returning error path. Other inputs proceed without visible range, ownership, or counter-underflow checks.

## Interface

### Input

There is one argument: a 32-bit integer interpreted as an address, designated **P**.

- **P = 0:** return normally without calling dependencies or accessing metadata or state.
- **P ≠ 0:** interpret the preceding eight bytes as metadata and use a state-object address obtained through shared storage.

The source-facing representation uses 32-bit integer and address quantities and a 16-bit metadata quantity. Signedness affects some expressions, but the accounting values are handled as 32-bit stored bit patterns. This contract does not establish portable behavior for translating those expressions to a different language or machine model.

### Metadata layout

For nonzero P, the following locations are accessed:

| Location relative to P | Width | Established use |
|---|---:|---|
| −8 bytes | 32 bits | Initial validity checks; masked accounting amount |
| −4 bytes | 16 bits, unsigned | First state-counter selector |
| −2 bytes | 16 bits, unsigned | Second state-counter selector |

The metadata address passed to a dependency is **P − 8**, not P.

The first metadata word is masked with **0x3FFFFFFC** when extracting an accounting amount. Consequently:

- Bits 0 and 1 do not contribute to that amount.
- Bits 30 and 31 do not contribute.
- The amount is a multiple of four.
- Its representable range is 0 through **0x3FFFFFFC**, inclusive.
- A zero extracted amount is permitted by the visible validation.

Only bit 0 is tested as a disqualifying flag. No meanings are established for the other excluded bits.

### State layout

Let **S** be the state-object address read from shared storage after F-2fd8826557a4 returns.

The function uses these 32-bit state fields:

| Offset from S | Access or role |
|---|---|
| 0x008 | Subtract a late-read masked metadata amount |
| 0x0E0 + 4 × second selector | Subtract the cached accounting amount |
| 0x220 | Subtract the cached accounting amount |
| 0x224 + 4 × first selector | Subtract the cached accounting amount |
| 0x30C | Subtract the cached accounting amount |
| 0x310 + 4 × dependency result | Decrement by one |
| 0x390 | Decrement by one |

These are address formulas, not proof of disjoint arrays or valid selector ranges. Some formulas can identify the same location.

### Output and termination

There is no explicit return value.

Normal completion means the visible updates and final dependency calls have completed. It does not independently establish that storage has been released.

The metadata-error dependency is marked non-returning. Its exact failure mechanism—termination, exception, trap, or another control transfer—is not established here.

### Preconditions for defined successful execution

For a nonzero input:

- The metadata reads must address accessible storage with suitable access characteristics.
- The shared-state lookup must produce a usable state address.
- Every computed counter location must support its required 32-bit read and write.
- Selector values and the dependency-provided index must produce valid addresses.
- Metadata and state needed after each dependency call must remain usable.

The function does not visibly enforce these conditions. The contracts of unknown dependencies cannot be assumed to supply them.

## Behavior

### 1. Null handling

The argument is tested before any dependency call or metadata access.

If it is zero, the function returns immediately. In particular, it does not perform initialization, retrieve the state address, validate metadata, update accounting, or invoke the final dependencies.

### 2. Preparation and state capture

For a nonzero argument:

1. F-2fd8826557a4 is called with no explicit arguments.
2. After that call returns, the state-object address S is read from shared storage.
3. The metadata word at P − 8 is examined.

The ordering matters: F-2fd8826557a4 can affect the state address subsequently selected. This function does not read that address before the call.

Once captured, S remains the address used by this function. Later changes to the shared state-address storage do not cause it to select a different state object. This does not imply that later dependencies preserve the contents or lifetime of the captured object.

### 3. Metadata validation

The initial metadata check rejects either of these conditions:

- The 32-bit metadata word is zero.
- Its least significant bit is set.

The second condition is reached only when the zero comparison is false. The supplied representation contains a metadata-word read for the zero comparison and another for the bit test. A single immutable snapshot across those reads is not established.

On rejection, F-a817d8d58c10 is called without explicit arguments and does not return. The subsequent state-processing call and accounting updates are not reached.

Validation is narrower than a general allocation-validity check. It does not establish:

- That P identifies an allocation owned by S.
- That the metadata describes a currently live object.
- That the extracted amount is nonzero.
- That either selector is within an intended range.
- That any counter is large enough for subtraction.
- That the metadata remains unchanged after validation.

### 4. State-processing dependency and amount capture

After validation succeeds, F-c0fc6d033caa is called with S.

Only after that call returns does the function read the first 16-bit selector and obtain the masked accounting amount from the metadata word at P − 8.

Designate this amount **A**. It is computed by retaining only bits selected by 0x3FFFFFFC.

Because A is captured after F-c0fc6d033caa, it need not come from the same metadata value that passed validation. This function does not revalidate the word after that dependency returns.

### 5. Cached-amount accounting updates

The following 32-bit read-modify-write updates occur in order:

1. Subtract A from the location at S + 0x224 + four times the first unsigned 16-bit selector.
2. Subtract A from the location at S + 0x30C.
3. Read the second unsigned 16-bit selector at P − 2, then subtract A from the location at S + 0x0E0 + four times that selector.
4. Subtract A from the location at S + 0x220.

The first selector is read before A is captured. The second selector is read after the first two counter updates. This difference is observable if metadata overlaps a written counter or changes externally.

Each selector is zero-extended before scaling by four. Scaling a 16-bit unsigned selector by four produces a value from zero through **0x3FFFC**; that scaling alone does not overflow 32 bits. Adding the result to S and the field offset still requires a valid address.

The function does not require the selected counters to be distinct from fixed counters or from each other. If two formulas resolve to the same location, the later operation uses the value left by the earlier operation.

### 6. Count updates

After the four cached-amount subtractions:

1. F-09dcb55e8e0a is called without explicit arguments.
2. Its integer result is used as an index in the address S + 0x310 + four times that result.
3. The 32-bit value at that address is decremented by one.
4. The 32-bit value at S + 0x390 is decremented by one.

No index-range check is visible. In particular, this function does not establish that the dependency result is nonnegative or bounded by any table size.

If the indexed location is S + 0x390, the same counter is decremented twice in sequence.

F-09dcb55e8e0a executes after the cached-amount updates and before these decrements. Its effects on metadata, state, or other shared storage are unknown.

### 7. Late amount read and final calls

After both count decrements, the metadata word at P − 8 is read again and masked with 0x3FFFFFFC. Designate this second extracted amount **B**.

The function subtracts B from the 32-bit field at S + 0x008.

There is no requirement in the visible behavior that B equal A. Intervening dependency activity, aliasing, or external modification can make them differ. There is also no repeat of the zero-word or bit-0 validation.

Finally:

1. F-cc6c9b4a0f68 is called with S and the metadata address P − 8.
2. After that call returns, F-e586bff64fb7 is called with S.
3. After that call returns, F-ff1bdc376dcd returns normally.

No explicit metadata read or write occurs in this function after F-cc6c9b4a0f68 is called. This does not establish whether that dependency consumes, retains, releases, or modifies the supplied storage.

### Arithmetic and access-order limits

The updates are ordinary 32-bit counter operations. No saturation, checked subtraction, underflow error, or rollback is visible. At the represented 32-bit word level, subtracting from zero can produce a wrapped bit pattern.

The supplied evidence establishes the sequencing of the updates described above. It does not establish atomic read-modify-write operations or a complete machine-level ordering of operand reads within an individual expression.

## Side effects

### Direct effects

For a successful nonzero invocation, the function directly performs seven counter updates:

- Four subtractions using A.
- Two decrements by one.
- One subtraction using B.

Those seven operations need not affect seven distinct locations.

It directly reads:

- The shared state-address storage.
- The metadata word multiple times.
- Two unsigned 16-bit selectors.
- Each counter value before replacing it.

It has no explicit direct write to the metadata. A computed state-counter address could nevertheless alias metadata; non-aliasing is not established here.

### Dependency-mediated effects

The dependencies may have additional effects not visible in this body. It is not justified to describe the function as side-effect-free except for counters, nor to assume that calls preserve globals, metadata, state contents, or memory lifetime.

Possible synchronization roles are inferred intent only. The call placement does not establish a lock protocol.

### Failure and partial progress

The visible metadata-error path precedes direct accounting updates, but follows F-2fd8826557a4 and the state-address read. Effects of that first dependency may therefore already have occurred.

There is no visible rollback. If a later dependency does not return, or an invalid access prevents progress, earlier updates are not explicitly undone.

### Ownership

The function accepts a raw address without an explicit ownership token or allocator argument. Passing P − 8 to F-cc6c9b4a0f68 suggests a metadata-management operation, but does not prove a transfer of ownership or physical reclamation.

Whether the caller may reuse P after normal completion is **documented_only** pending dependency evidence.

## Dependencies

| Opaque ID | Invocation | Established role |
|---|---|---|
| F-2fd8826557a4 | No explicit arguments; first call for nonzero input | Runs before state capture and metadata validation |
| F-a817d8d58c10 | No explicit arguments; non-returning | Handles initial zero-word or bit-0 rejection |
| F-c0fc6d033caa | S | Runs after validation, before selector and amount accounting |
| F-09dcb55e8e0a | No explicit arguments | Supplies the index for a count decrement |
| F-cc6c9b4a0f68 | S, P − 8 | Receives state and metadata addresses after all direct updates |
| F-e586bff64fb7 | S | Final dependency call |

For a normally completing nonzero input, the call order is:

F-2fd8826557a4 → F-c0fc6d033caa → F-09dcb55e8e0a → F-cc6c9b4a0f68 → F-e586bff64fb7.

None of these roles establishes an internal implementation or an ownership guarantee.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

### Null input

Given P = 0, the function returns normally with no dependency calls, shared-state lookup, metadata reads, or counter updates.

### Zero metadata rejection

Given a nonzero readable P and a zero word at P − 8 when checked, the function calls F-2fd8826557a4, captures S, and reaches F-a817d8d58c10. It does not reach the subsequent accounting path.

### Bit-0 rejection

Given stable metadata with bit 0 set, the function reaches the same non-returning rejection path, regardless of the other bits.

### Ordinary accounting with stable, disjoint storage

Suppose:

- The metadata word remains 0x00000042, giving A = B = 64.
- The first selector is 2.
- The second selector is 3.
- F-09dcb55e8e0a returns 1.
- All relevant locations are valid and distinct.
- Dependencies do not independently modify these values.

Then the function subtracts 64, in order, at offsets 0x22C, 0x30C, 0x0EC, and 0x220; decrements offsets 0x314 and 0x390; and subtracts 64 at offset 0x008 before the final two calls.

### Accepted zero accounting amount

Given a stable metadata word of 0x00000002, initial validation succeeds and both extracted amounts are zero. The count decrements and final calls still occur. Zero-amount subtraction is not a reason to skip the path.

### Distinct cached and late amounts

Suppose A is captured as 64, and F-09dcb55e8e0a changes the metadata word so that B becomes 128. Assuming valid, otherwise unaffected storage, the four earlier amount counters lose 64 each, while the field at offset 0x008 loses 128.

### Overlapping count locations

If F-09dcb55e8e0a returns 32, its indexed location is offset 0x390. Assuming no intervening external modification, that field is decremented twice: once as the indexed count and once as the fixed count.

### No counter-underflow rejection

If a valid selected 32-bit counter contains zero and A is four, the function does not visibly reject the subtraction. Under 32-bit word arithmetic, the resulting bit pattern is 0xFFFFFFFC.

## Uncertainties

The following remain **documented_only**:

- Allocation type, payload extent, and permitted caller ownership.
- Meaning of metadata flags other than the observed bit-0 rejection.
- Intended ranges and category meanings of the selectors.
- Valid range and meaning of the dependency-provided index.
- Whether the state regions are intentionally overlapping.
- Synchronization, reentrancy, atomicity, and concurrency guarantees.
- Whether dependencies mutate shared-state selection, metadata, or counters.
- Whether any dependency releases or invalidates memory.
- Exact non-returning failure behavior.
- Alignment requirements, byte order, and fault behavior.
- Full machine-level access ordering within individual expressions.

The apparent release/accounting purpose is inferred intent. It must not be elevated into an ownership, locking, or memory-reclamation guarantee without additional evidence.

## Evidence

This contract is based solely on the supplied static function body and the supplied opaque dependency mapping.

Established observations include the null guard, call ordering, state capture, metadata widths and offsets, validation conditions, mask value, counter address formulas, ordered updates, late metadata reread, and final dependency arguments.

No dependency implementations or runtime observations were supplied. No runtime tests have been performed. Acceptance examples state consequences under explicit assumptions rather than reporting observed executions.
