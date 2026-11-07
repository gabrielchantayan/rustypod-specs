<!-- Trusted provenance: F-1a5e4b9bacb5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1a5e4b9bacb5

## Purpose

F-1a5e4b9bacb5 selects among record traversal, two ordered indirect-call sequences, and no branch-specific work. Every invocation first calls two dependencies to obtain a retained receiver, even when the selected operation does not subsequently use that receiver.

Every path reaching the explicit return produces the four-byte value `1`. This result does not distinguish an unrecognized operation, empty traversal, suppressed record forwarding, or downstream failure reported through an ignored return value.

**Inferred intent:** the function contributes operation-dependent values to a caller-supplied destination. The destination’s concrete type and the effects of those contributions are not established.

This contract describes static evidence and conditional behavior. It does not assert that any execution has been observed.

## Interface

### Arguments and result

| Input | Established representation | Visible use |
|---|---|---|
| Source address | Signed 32-bit integer used in address formation | Source plus `0x28` is passed to F-538044b67754 in traversal mode. |
| Operation selector | Signed 32-bit integer | Selects branch behavior through comparisons and subtraction. |
| Destination token | Opaque four-byte value | Forwarded to F-2484a607f41b and to indirect calls. |

The source is not directly dereferenced by this function. Its derived address is supplied to a dependency. No destination fields are directly accessed, and the destination token is not established to be a pointer rather than a handle or another representation.

The explicit result is a four-byte value equal to `1`. There is no explicit alternative result or local error-reporting branch. An invalid access or a dependency that does not return normally may prevent that result.

### Storage and layouts

The visible body uses:

- A local 20-byte workspace whose internal layout is opaque.
- A separate four-byte yielded-address slot, initialized to zero before traversal initialization.
- A four-byte starting record address at offset `0x3C` from a yielded address.
- A four-byte end address at offset `0x40` from a yielded address.
- Record cursor increments of `0x10`.
- A four-byte table address at offset zero in the retained receiver.
- An indirect callable entry at table offset `0x54`.

The record stride does not establish a record schema. Individual record fields are not read directly.

Six distinct global storage locations supply the second explicit arguments of the indirect calls. They are designated **A1**, **A2**, **A3**, **B1**, **B2**, and **B3**. These names identify storage locations, not fixed argument values. Their declarations and exact load types are not supplied. The evidence establishes reads of their stored values, not passage of their addresses.

Byte order, alignment requirements, complete calling conventions, and global operand types are not established.

### Arithmetic qualification

The displayed source uses signed 32-bit expressions for:

- Operation-selector subtraction by `0x6600`.
- Source-address addition by `0x28`.
- Record-cursor advancement by `0x10`.

The yielded-base field-offset expressions also use the signed integer holding the yielded address.

These expressions must not silently be replaced with a portable unsigned-address model. Where signed results are representable, ordinary arithmetic describes the displayed computation. Where an expression exceeds the signed 32-bit range, the supplied representation does not establish portable overflow behavior.

A wrapping 32-bit machine-address interpretation may be useful for conditional examples, but it is an explicit assumption in those examples, not an established language-level guarantee.

## Behavior

### Initial calls

Before operation selection:

1. F-9d0ecc5f6adf is called without explicit arguments.
2. Its four-byte result and selector `0x80` are passed to F-682adfd5a561.
3. The returned receiver address is retained.

Neither result is checked for nullness or another failure indication.

These calls occur for traversal operations and unrecognized operations as well as for indirect-call operations. They may affect global storage or other state; no preservation guarantee is supplied.

### Operation selection

For selector arithmetic with defined ordinary results, the branches are:

| Operation | Behavior |
|---|---|
| `0x6604` | Record traversal |
| `0x6612` | Record traversal |
| `0x6613` | Three A-family indirect calls, then decimal-digit forwarding |
| `0x6614` | Three B-family indirect calls |
| Other values | No branch-specific work |

The `0x6604` case bypasses the selector subtraction. Other values undergo subtraction by `0x6600`; a difference of `0x12` selects traversal.

Consequently, the no-work statement for other selectors is qualified for inputs whose displayed signed subtraction overflows. The representation does not establish portable behavior for those inputs, nor justify adding further recognized operations.

### Traversal initialization and advancement

For either traversal operation:

1. The yielded-address slot is set to zero.
2. F-538044b67754 receives the workspace address, the source address plus `0x28`, and `0xFFFFFFFE`.
3. F-6e4651b532dc receives the workspace address and the address of the yielded-address slot.

The third initialization argument is the 32-bit bit pattern `0xFFFFFFFE`, equivalent to signed `-2`. Its semantic meaning is unknown. No initialization result is inspected.

A zero advancement result ends traversal. Any nonzero result starts processing through the address then held in the yielded-address slot. The function does not validate that address.

The slot is not explicitly reset between advancement calls. Its contents after a nonzero result therefore depend on F-6e4651b532dc and any other effects capable of reaching that storage.

### Inner-loop boundary reads

At entry to processing each yielded result, the function reads the starting cursor from offset `0x3C` using the yielded-address slot’s current value.

Before every record visit, including the first, it reads the end address from offset `0x40` using the slot’s **then-current value**:

- Equality between cursor and end address finishes this yielded result.
- Inequality processes the current cursor as a record.
- After record processing returns, the cursor advances by `0x10` and the condition is evaluated again.

Only the starting cursor is captured at loop entry. There is no separately retained yielded-object base for the boundary checks.

Two kinds of change can therefore affect the next condition:

- A dependency may change the end field at the currently selected base.
- A dependency may change the yielded-address slot, redirecting the next end-field read to another base.

The slot’s address has been exposed to F-6e4651b532dc. Whether that dependency retains it is unknown. Later dependencies cannot be assumed incapable of modifying the slot through retained or shared state.

A slot change does not reset the active cursor. Likewise, a change to a starting-address field does not reload the cursor during the current inner loop. The next condition combines the advancing cursor with the end field selected through the slot at that time.

There is no less-than comparison, range-order validation, alignment check, or record-count limit.

### Per-record processing

For every visited cursor address:

1. F-fc7bc176e791 receives that record address.
2. F-6d70e45e29c9 is called without explicit arguments.
3. A nonzero result suppresses further processing for this record.
4. A zero result causes:
   - A second call to F-fc7bc176e791 with the same cursor address.
   - A call to F-44362605ab79 without explicit arguments.
   - A call to F-2484a607f41b with the destination token first and the retrieved four-byte value second.

The second record-address call is observable behavior. It cannot be eliminated on an assumption that the first call’s effects survive the gate call unchanged.

The gate’s semantic meaning is unknown. Zero permits retrieval and forwarding; every nonzero result suppresses both. Each permitted record causes a separate forwarding call. There is no visible batching, deduplication, or filtering based on the retrieved value.

After the inner loop ends, advancement is called again. When advancement returns zero, F-944b595e25e9 receives the workspace address once on that normal path.

### Operation `0x6613`

This branch performs three indirect calls using the retained receiver. Each call obtains a callable through the receiver’s current table pointer and the entry at offset `0x54`.

Each call receives:

1. The retained receiver address.
2. A value read from one global storage location.
3. The destination token.

The storage locations are used in order: A1, then A2, then A3.

The reads are associated with their respective calls, not a snapshot taken before the sequence. In particular:

- The A2 value is obtained after the first call returns.
- The A3 value is obtained after the second call returns.
- Earlier dependencies may modify the values later read from those locations.

Distinct locations do not imply distinct values, immutable contents, pointer-valued contents, or passage of the locations’ addresses.

The table pointer and callable entry are also obtained anew for each call. An earlier call may change the target used by a later call. The representation does not establish a relative evaluation order between the target lookup and global operand read within a single call expression.

After the third indirect call returns, F-2484a607f41b receives the destination token and the statically designated decimal-digit operand `0123456789`.

No indirect-call result is inspected. A returned failure indicator alone does not suppress subsequent calls. Abnormal non-return can prevent them.

### Operation `0x6614`

This branch uses the same receiver and fresh table-entry lookup mechanism, but reads B1, B2, and B3 for its three calls, in that order.

B2 is read after the first call returns; B3 is read after the second returns. Earlier calls may change later operand values or dispatch targets. The three global locations need not contain different values.

There is no subsequent decimal-digit forwarding call and no iterator initialization or finalization.

### Other operations

Subject to the signed-arithmetic qualification above, unrecognized operations perform only the initial two dependency calls and then return `1`.

This is not an explicit error path. It also does not establish absence of side effects, because the initial dependencies still execute.

## Side effects

### Direct and dependency-mediated changes

The visible function directly changes its local yielded-address slot and loop-control state. It exposes local storage to dependencies.

There is no explicit direct assignment to source storage, record storage, yielded-object fields, the retained receiver, destination storage, global operand storage, or other globals. Called dependencies may nevertheless change any accessible state.

Established sequencing includes:

- Context acquisition before receiver lookup.
- Receiver lookup before branch-specific work.
- Workspace initialization before advancement.
- Record-address submission before gate evaluation.
- Second record-address submission before retrieval and forwarding on a zero gate.
- Each global operand read for an indirect call after preceding calls have returned.
- Decimal-digit forwarding after the third A-family call.
- Workspace finalization after advancement returns zero.

Calls without explicit arguments may rely on shared state established by earlier calls. The state’s location and preservation rules are unknown.

### Ownership and lifetime

No allocation or release semantics are established for any dependency.

The workspace and yielded-address slot are local. Their addresses are passed outward, but retention behavior is unknown. The finalization call establishes invocation of F-944b595e25e9, not a particular resource-release guarantee.

No ownership transfer can be established for the receiver, source-derived addresses, record addresses, destination token, retrieved values, global operand values, or digit operand. Dependencies may borrow, copy, retain, consume, or release referenced resources according to contracts not supplied here.

No abnormal-exit cleanup is visible. Finalization is established only when traversal reaches the normal zero advancement result.

## Dependencies

| Dependency | Established role | Not established |
|---|---|---|
| F-9d0ecc5f6adf | Produces the initial four-byte value | Meaning, failure representation, side effects |
| F-682adfd5a561 | Receives that value and `0x80`; returns the retained receiver | Validity, ownership, lookup semantics |
| F-538044b67754 | Receives workspace, source plus `0x28`, and `0xFFFFFFFE` | Source layout, mode meaning, failure behavior |
| F-6e4651b532dc | Receives workspace and yielded-slot address; zero ends traversal | Yield order, slot retention, exhaustion versus error |
| F-944b595e25e9 | Receives workspace after normal traversal termination | Release effects, failure behavior |
| F-fc7bc176e791 | Receives each visited record address, twice when the gate is zero | State effects, validation, ownership |
| F-6d70e45e29c9 | Supplies the signed 32-bit zero/nonzero gate | Semantic meaning, shared-state contract |
| F-44362605ab79 | Supplies a four-byte value | Type, lifetime, ownership |
| F-2484a607f41b | Receives destination and retrieved value or digit operand | Destination effects, encoding, retention, errors |
| Indirect entry at `0x54` | Receives receiver, a global-storage value, and destination | Target identity, operand interpretation, mutations |

No opaque function ID is supplied for the indirect target. It must not be identified with another dependency without additional evidence.

## Acceptance examples

All examples are conditional static expectations, not runtime observations.

### Unrecognized operation

For operation `0x6600`, assuming both initial calls return normally:

- F-9d0ecc5f6adf precedes F-682adfd5a561.
- No traversal, indirect call, or direct forwarding call follows.
- The explicit result is `1`.

The initial dependencies may still have side effects.

### Empty traversal

For `0x6604` or `0x6612`, assume source-address arithmetic is defined and initialization and advancement return normally. If the first advancement returns zero:

- The yielded-address slot was initialized to zero before initialization.
- No yielded-object boundary fields are read.
- No record is processed.
- F-944b595e25e9 is called once.
- The explicit result is `1`, if finalization returns.

### Two records with different gate results

Assume the yielded-address slot remains stable throughout the inner loop, the selected object’s start and end remain `0x1000` and `0x1020`, all accesses are valid, and all dependencies return normally.

If the gate is nonzero at `0x1000` and zero at `0x1010`:

- Both addresses receive an initial F-fc7bc176e791 call.
- Only `0x1010` receives the second such call.
- Only `0x1010` causes retrieval and forwarding.
- The cursor next becomes `0x1020`, matches the end field, and traversal advances.

These assumptions include slot stability; stable fields at the original object alone are insufficient.

### Redirected boundary read

Assume the initial slot selects an object whose start is `0x1000`, and the first condition permits that record. If a dependency changes the slot to select another valid object before the next condition:

- The cursor still advances from `0x1000` to `0x1010`.
- The next end-field read uses the replacement base plus `0x40`.
- The replacement object’s starting field is not loaded to reset the cursor.
- If its end is `0x1010`, the inner loop ends.

This is conditional on such mutation being possible; no dependency implementation proving it is supplied.

### Equal boundaries and unreachable end

With a stable slot and equal start and end at the first comparison, no record is processed for that yielded result.

Separately, assume wrapping 32-bit arithmetic, a stable slot, start `0x1000`, and end `0x1008`. Repeated increments of `0x10` cannot equal that end. The function has no explicit malformed-range exit. Normal termination is not guaranteed; inaccessible records or nontermination are possible.

This wrapping example does not establish portable signed-overflow behavior.

### Mutable indirect-call operands

For `0x6613`, suppose the first indirect call changes A2 before returning. The second call receives the value read from A2 afterward, not a value captured before the sequence. The corresponding rule applies to A3 after the second call.

All three A-family locations may contain the same value without violating the contract. The required sequence remains three calls using the respective location reads, followed by digit forwarding.

For `0x6614`, the same rules apply to B1, B2, and B3, with no digit forwarding. In either branch, changes to the receiver’s table may also change later callable targets.

## Uncertainties

The following remain **documented_only**:

- Concrete types and semantic roles of source, destination, receiver, and records.
- Contents, declared widths, types, and interpretation of the six global operands.
- Digit-operand storage length, terminator, encoding, and downstream interpretation.
- Iterator initialization failure and exhaustion-versus-error distinctions.
- Retention of workspace or yielded-slot addresses.
- Global preservation, callbacks, reentrancy, and thread safety.
- Ownership and release obligations for dependency-produced values.
- Behavior after invalid addresses, inaccessible storage, or malformed tables.
- Portable overflow behavior for the displayed signed arithmetic.
- The exact shared-state relationship, if any, among record submission, gate evaluation, and retrieval.
- Evaluation order between independent target and operand expressions within one indirect call.

Missing checks do not imply safe acceptance of malformed inputs.

## Evidence

The contract derives from the supplied static function representation and opaque dependency mapping.

Established evidence includes the branch constants, unconditional initial calls, argument order, local storage sizes, four-byte boundary fields, offsets `0x28`, `0x3C`, `0x40`, and `0x54`, cursor stride, gate polarity, repeated record-address call, per-condition use of the yielded-address slot, six global-storage operand reads, fresh indirect dispatch expressions, normal-path finalization, and constant explicit return.

“Iterator,” “gate,” “retrieval,” and “forwarding” describe observed roles rather than imported dependency contracts. The contribution-oriented purpose is inferred intent.

No runtime tests have been performed.
