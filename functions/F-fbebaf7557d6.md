<!-- Trusted provenance: F-fbebaf7557d6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fbebaf7557d6

## Purpose

F-fbebaf7557d6 conditionally initializes a field in a primary object, performs a fixed sequence of operations involving a secondary object, and selects a notification according to two dependency results.

The established contract concerns accesses, calls, and branching. The domain meanings of the objects, the field described here as a **resource**, and the retained **intermediate value** are unknown. “Initialization” describes the zero-check-and-store mechanism, not a proven allocation or ownership model.

Every normal return supplies the four-byte value 1. That value does not distinguish initialization from reuse, notification from no notification, or positive from negative dependency outcomes. It is not an established success indicator.

## Interface

### Arguments and result

The function receives two integer-carried object references, represented as 32-bit quantities:

- **Primary reference:** used for accesses at byte offsets 0xF4 and 0xF8, and passed to either notification dependency.
- **Secondary reference:** passed to two dependencies and used for an access at byte offset 0x0C.

The normal result is a four-byte value equal to 1.

### Object layout and additional input

| Input location | Width established here | Use |
|---|---:|---|
| Primary offset 0xF4 | 4 bytes | Initially checked for zero; conditionally overwritten; subsequently reread for two dependency calls |
| Primary offset 0xF8 | 1 byte | Read only on the initialization path and supplied to F-1058afc38c04 |
| Secondary offset 0x0C | 4 bytes | Read after the two initial secondary-object calls and supplied to F-aa672b15efb7 |
| Global configuration location | Unknown | Read only on the initialization path to supply the fourth argument to F-1058afc38c04 |

The global configuration location is an observable input distinct from the two object references. Its contents, width, representation, and domain meaning are not established. Its access timing is established and forms part of the contract.

The primary resource field is treated as a four-byte value whose zero/nonzero state controls initialization. No pointer validity or resource validity follows from a nonzero value.

The result of F-aa672b15efb7 is interpreted as a signed 32-bit outcome. The result of F-be0ad8f71bac is represented as a 32-bit integer and tested only for zero versus nonzero.

### Preconditions and limits

Execution requires the reached object accesses and global access to be valid. The primary resource field must be readable and, if initialization occurs, writable. The initialization byte and global configuration location need be readable only when their accesses are reached. The secondary field must be readable when its later access occurs.

Dependencies may impose additional requirements on the references and forwarded values. Those requirements are unavailable.

There are no explicit checks for null references, object size, alignment, address overflow, allocation failure, or dependency failure. The supplied representation does not establish address-wrap behavior, endianness, or invalid-access handling. The contract therefore requires valid effective addresses rather than prescribing behavior for overflowing address calculations.

No ownership transfer, release obligation, or exclusive-access guarantee is established.

## Behavior

### 1. Decide whether to initialize

The first explicit data access is a four-byte read of primary offset 0xF4.

- If the value is nonzero, initialization is skipped.
- If the value is zero, the initialization sequence below is performed once.

Skipping initialization also skips the initialization-byte read, the global configuration read, and all three initialization dependencies. A later change to the resource field does not cause this decision to be revisited during the same invocation.

### 2. Initialize when the initial resource value is zero

The initialization path has the following observable ordering:

1. Call F-8157a180d901 with 52, represented by the constant 0x34.
2. After that call returns, read:
   - The one-byte value at primary offset 0xF8.
   - The global configuration value.
3. Call F-1058afc38c04 with four arguments, in these argument positions:
   - The captured result of F-8157a180d901.
   - The constant 5.
   - The initialization-byte value.
   - The value obtained from the global configuration read.
4. Store the four-byte result of F-1058afc38c04 at primary offset 0xF4.
5. Call F-991ae23131ae with that captured result and 64, represented by the constant 0x40.

Both the initialization-byte read and the global configuration read occur after F-8157a180d901 returns and before entry into F-1058afc38c04. Their relative evaluation order is not established. The argument-position list does not imply a left-to-right read order.

F-8157a180d901 must not be assumed to preserve either input. If it changes the initialization byte or the global configuration location, a subsequent read obtains the value present when that read occurs. The global input is not established to be a constant or an entry-time snapshot.

The value passed to F-991ae23131ae is the captured initialization result, not a fresh read of primary offset 0xF4. The store occurs before that dependency is entered.

Neither initialization result is checked for failure or zero. If F-1058afc38c04 returns zero normally, zero is stored and passed to F-991ae23131ae, and execution continues. There is no same-invocation retry.

### 3. Perform secondary-object operations

After initialization completes or is skipped, the function performs these calls in order:

1. Call F-c0a513a969e0 with the secondary reference and retain its four-byte result as the intermediate value.
2. Call F-8bb3020950cd with the secondary reference. Its result is unused.
3. Call F-aa672b15efb7 with:
   - A fresh four-byte read of primary offset 0xF4.
   - A four-byte read of secondary offset 0x0C.
   - The retained intermediate value.
4. Call F-be0ad8f71bac with:
   - Another fresh four-byte read of primary offset 0xF4.
   - The same retained intermediate value.

The two field reads supplying F-aa672b15efb7 occur after F-8bb3020950cd returns and before entry into F-aa672b15efb7. Their relative evaluation order is not established.

Consequently, the secondary field is not an entry-time snapshot. A change made by either preceding secondary-object dependency can affect the value subsequently read.

The resource field is also not retained across the two consuming calls. F-aa672b15efb7 may change it, and F-be0ad8f71bac receives the value read afterward. Neither call is guaranteed to receive the initially observed or newly initialized resource value.

The intermediate value, by contrast, is retained from F-c0a513a969e0. It is not recomputed after F-8bb3020950cd or F-aa672b15efb7.

### 4. Select a notification

Let **outcome** denote the captured signed 32-bit result of F-aa672b15efb7, and **selector** denote the result of F-be0ad8f71bac.

| Selector | Outcome | Action |
|---|---|---|
| Zero | Zero | No notification |
| Zero | Nonzero | Call F-d1af0b2a6297 with the primary reference and the exact outcome |
| Nonzero | Positive | Call F-bff44ce51e3d with the primary reference and exactly 1 |
| Nonzero | Zero | Return 1 immediately without notification |
| Nonzero | Negative | Call F-bff44ce51e3d with the primary reference and exactly 0 |

The outcome remains the value returned by F-aa672b15efb7 even though F-be0ad8f71bac executes afterward. The selector’s sign is irrelevant.

For a zero selector, the notification argument preserves the outcome’s magnitude and sign; it is not converted to a Boolean. For a nonzero selector, the notification argument is normalized to 0 or 1, and a zero outcome bypasses notification entirely.

The negative branch requires no negation or absolute-value computation. The signed minimum, −2,147,483,648, therefore follows the negative branch without an arithmetic-overflow issue in this selection logic.

At most one notification dependency is called. Every normally completing path returns 1.

## Side effects

The function’s only explicit memory write is the conditional four-byte store to primary offset 0xF4. There is no explicit write to the initialization byte, secondary field, or global configuration location.

The store precedes F-991ae23131ae. There is no explicit restoration or clearing of the resource field afterward.

Dependency effects remain unrestricted by the supplied evidence:

- F-8157a180d901 may change the initialization byte or global configuration before their reads.
- Initialization dependencies may change the resource field after it is stored.
- Secondary-object dependencies may change either object or shared state.
- F-aa672b15efb7 may change the resource field before its next read.
- Either notification dependency may change state beyond its explicit arguments.

No dependency is established to preserve globals, retain or release memory, or leave object lifetimes intact. Reached accesses require the relevant storage to remain valid despite preceding calls.

There is no visible release operation, rollback, or cleanup path. This does not prove that dependencies never release memory. It also does not establish that the result of F-8157a180d901 is allocated memory or that the stored resource belongs to the caller.

If an access faults or a dependency does not return normally, the function’s normal-return guarantee does not apply. The supplied evidence establishes no recovery or reversal of earlier effects.

No thread-safety, reentrancy, synchronization, or atomicity guarantee follows from this sequence. The described reads identify access points, not a promise of an isolated snapshot.

## Dependencies

These are call-site contracts only.

| Opaque function ID | Inputs supplied by this function | Result use |
|---|---|---|
| F-8157a180d901 | Constant 52 | Captured and forwarded as the first argument to F-1058afc38c04 |
| F-1058afc38c04 | Captured preceding result, constant 5, initialization byte, value read from the global configuration location | Four-byte result stored in the resource field and supplied to F-991ae23131ae |
| F-991ae23131ae | Captured initialization result, constant 64 | Unused |
| F-c0a513a969e0 | Secondary reference | Four-byte result retained for two later calls |
| F-8bb3020950cd | Secondary reference | Unused |
| F-aa672b15efb7 | Fresh resource value, secondary field value, retained intermediate value | Signed 32-bit outcome |
| F-be0ad8f71bac | Newly read resource value, retained intermediate value | Zero/nonzero selector |
| F-d1af0b2a6297 | Primary reference, exact nonzero outcome | Unused |
| F-bff44ce51e3d | Primary reference, normalized 0 or 1 | Unused |

Argument ownership, retention, lifetime, validation, error conventions, and preservation of external state remain unspecified. The constant 52 alone does not establish allocation semantics or a byte count.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Unless stated otherwise, they assume valid accesses and normally returning dependencies.

### Existing resource skips all initialization inputs

Given an initially nonzero resource field:

- F-8157a180d901, F-1058afc38c04, and F-991ae23131ae are not called.
- Neither the initialization byte nor the global configuration location is read by this function.
- Execution proceeds to F-c0a513a969e0.

This path does not require those two skipped input locations to be readable on account of this function’s own accesses.

### Global configuration changes during the first initialization call

Given an initially zero resource field, suppose F-8157a180d901 changes the global configuration location and returns normally.

If no intervening change occurs before the global read, the fourth argument to F-1058afc38c04 is the changed value, not the value present at function entry. The same rule applies independently to a changed initialization byte.

An acceptance check must not require either of those two reads to precede the other. Their established ordering is only after F-8157a180d901 and before F-1058afc38c04.

### Initialization returns zero

Given an initially zero resource field and a zero result from F-1058afc38c04:

- Zero is stored at primary offset 0xF4.
- F-991ae23131ae receives zero and 64.
- Secondary-object processing still follows.
- Initialization is not retried.

If subsequent dependencies leave the field unchanged, both later resource reads yield zero. Whether their callees accept zero is unknown.

### Zero selector preserves a negative outcome

Given outcome −7 and selector zero:

- F-d1af0b2a6297 receives the primary reference and −7.
- F-bff44ce51e3d is not called.
- The function returns 1.

Any other nonzero signed outcome is likewise forwarded without Boolean normalization.

### Nonzero selector normalizes a positive outcome

Given outcome 12 and selector −1:

- The selector is treated as nonzero.
- F-bff44ce51e3d receives the primary reference and exactly 1.
- F-d1af0b2a6297 is not called.
- The function returns 1.

### Nonzero selector with the signed minimum

Given outcome −2,147,483,648 and any nonzero selector, F-bff44ce51e3d receives exactly 0 as its second argument. No absolute value is required. The function returns 1.

### Zero outcome still requires the selector call

Given outcome zero, F-be0ad8f71bac still executes. Neither notification dependency is then called, regardless of the selector result. The function returns 1.

### Resource mutation between consuming calls

Suppose F-aa672b15efb7 receives one resource value and changes primary offset 0xF4 before returning. F-be0ad8f71bac receives the value from the later read, not a cached copy of the earlier argument.

The notification decision nevertheless uses the captured outcome returned by F-aa672b15efb7.

### Secondary-field mutation after the intermediate is captured

If F-8bb3020950cd changes secondary offset 0x0C, the subsequent field read observes the value present after that call. The intermediate argument still comes from the earlier F-c0a513a969e0 result; it is not recomputed.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the objects, resource, intermediate value, outcome, and selector.
- Contents, width, representation, and domain meaning of the global configuration input.
- Relative evaluation order of the global configuration read and initialization-byte read.
- Whether F-8157a180d901 allocates memory and whether 52 represents a byte size.
- Whether F-1058afc38c04 returns an address, a handle, or another four-byte value.
- Signedness and any argument-promotion details for the one-byte initialization input.
- Dependency failure conventions, nonlocal exits, and invalid-input behavior.
- Ownership, retention, aliasing, and release rules.
- Exact ABI, alignment, endianness, and effective-address overflow behavior.
- Relative order of the two field reads supplying F-aa672b15efb7.
- Concurrent mutation, synchronization, and memory-visibility guarantees.

Lazy resource initialization is inferred intent, not established domain semantics. In contrast, the global configuration read and its placement between two dependency calls are established behavior despite the value’s unknown meaning.

## Evidence

This contract is based on the supplied static function body and opaque dependency mapping. Prior prose and reviewer feedback are not independent evidence.

Established static evidence includes:

- Object-field offsets and access widths.
- The conditional initialization branch and single explicit store.
- The global read supplying the fourth initialization argument.
- Placement of that global read and the initialization-byte read after F-8157a180d901 and before F-1058afc38c04, without an established relative order.
- Dependency call order, retained results, and separate resource reads.
- Signed outcome classification, exact versus normalized notification arguments, and the constant normal return value.

No dependency implementations or runtime observations were supplied. No runtime tests have been performed. Acceptance examples express conditional consequences of the static behavior, not observed execution results.
