<!-- Trusted provenance: F-3faaf6d7241a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3faaf6d7241a

## Purpose

F-3faaf6d7241a performs one update step on a receiver associated with a selected service. It increments a receiver field, invokes two unconditional dependencies, and then conditionally performs an embedded-region adjustment or terminal handling.

Every explicit normal return produces the 32-bit value **1**. This result does not distinguish absent state, adjustment, error dispatch, unsuccessful lookup, or ready dispatch.

The established contract concerns accesses, arithmetic, branch conditions, and calls. Terms such as “service,” “progress,” “suppression,” “lookup,” and “notification” are inferred role labels, not independently established application semantics.

## Interface

### Inputs and result

| Position | Representation | Established use |
|---|---|---|
| First | Receiver pointer | Supplies fields, an embedded-region address, and virtual dispatch targets. |
| Second | Opaque 32-bit value | Not explicitly used or forwarded. |
| Third | Opaque 32-bit value | Copied into a local slot whose address is passed on notification paths. |
| Fourth | Opaque 32-bit value | Copied into a separate local slot whose address is passed to construction and post-dispatch dependencies. |

The third and fourth inputs are not established as strings, pointers, handles, or owned values. Their copies are separate local storage locations, not aliases of the caller’s argument storage.

Normal completion returns the 32-bit value **1**. No alternative explicit return value is present.

### Receiver layout

All offsets below are byte offsets.

| Offset | Access | Established use |
|---|---|---|
| `0x00` | Virtual-table reference | Locates receiver virtual methods. |
| `0xB0` | Address of an embedded region | Passed to the adjustment dependencies. Its complete extent is unknown. |
| `0xDC` | 32-bit field narrowed to a character | Zero/nonzero suppression test. |
| `0xE0` | 32-bit integer field | Incremented by 50, compared as unsigned, and conditionally overwritten. |
| `0xE4` | 32-bit field narrowed to a character | Selects the initial provider and later selects a ready identifier. |

The narrowed character value, rather than the complete 32-bit value, controls each flag test. Under an 8-bit-character interpretation, only the low eight bits affect zero versus nonzero. Exact character width and byte order remain unresolved; this contract does not substitute an addressed-byte read for the represented field read and narrowing.

Receiver storage offset `0xB0` and receiver virtual-table offset `0xB0` describe different locations.

### Global identities

Three distinct 32-bit global storage locations have these stable opaque identities:

| Identity | Established use |
|---|---|
| **G-01** | Threshold used for progress comparison, gap calculation, and a saved replacement value. |
| **G-02** | Supplies the **second argument** to F-b3bed2dfc5ec. |
| **G-03** | Supplies the **third argument** to F-b3bed2dfc5ec. |

These identities name storage locations, not particular values. Their initialization, application meaning, and synchronization are unknown.

### Preconditions

The receiver must support every reached field access and virtual dispatch. The selected service must support its reached virtual dispatches. Neither pointer receives an explicit null check.

The adjustment paths require the receiver’s embedded region to be valid for the called dependencies. The conditional 16-bit write requires a valid writable location at offset `0x80` of the object returned by F-fdd8589eca2d.

The body provides no explicit validation or recovery for invalid pointers, invalid virtual tables, inaccessible storage, or invalid dependency results that it directly dereferences.

## Behavior

### Entry and unconditional work

The function copies the third and fourth inputs into their respective local slots before calling any dependency.

It then tests the narrowed receiver selection field at `0xE4`:

- Zero selects F-6da76fc58c80.
- Nonzero selects F-af667771d6ce.

The selected provider’s result is retained as the service object. The service’s virtual method at byte offset `0x9C`, identified here as F-v01, is called with that service as receiver. Its result is retained as an opaque 32-bit state value.

The following actions occur next, in order:

1. Read the receiver’s current progress field at `0xE0`, add decimal 50, and write the result back.
2. Call F-6ac70711bbb5.
3. Call F-5d641e9c768c.

The increment therefore observes the progress value after the provider and initial service call. These calls are not assumed to preserve receiver state.

The increment is represented as signed 32-bit addition. Its result is established when the mathematical sum is representable. Signed-overflow behavior is unresolved; wrapping is not an acceptance requirement.

If the retained state value is zero, the function returns 1 after both unconditional calls. This path performs no predicate call, adjustment, terminal-status call, or notification.

### Predicate-controlled adjustment

For nonzero state, F-8ddad704d968 receives that state value.

If its result is nonzero, F-9f27c73ea83b is skipped and threshold handling follows.

If its result is zero, F-9f27c73ea83b receives the same retained state value:

- A zero result leads to threshold handling.
- A nonzero result selects the fixed adjustment.

The fixed adjustment consists of F-6904f7699031 receiving the embedded-region address at receiver offset `0xB0` and the value 50, followed by F-ec4cfa552cce receiving that same address. The function then returns 1.

This branch performs no explicit threshold replacement, terminal-status call, notification, or common terminal-tail call.

### Threshold handling

On the remaining nonzero-state paths, the function reads G-01 and saves its 32-bit value.

The receiver’s current progress field is then compared with G-01 using an **unsigned 32-bit comparison**. This is a later progress read, not a guaranteed reuse of the increment result. Intervening dependencies may have changed progress or global state.

If progress is below the threshold:

1. Form the 32-bit unsigned difference between G-01 and the current progress field.
2. Pass the embedded-region address and that difference to F-6904f7699031.
3. Pass the embedded-region address to F-ec4cfa552cce.
4. Overwrite progress with the previously saved G-01 value.
5. Return 1.

The threshold save, comparison, and subtraction refer to separate global accesses in the supplied representation. They must not be collapsed into a contractual guarantee that all three use one saved value. Relative operand-read order within an individual expression is not established.

With stable relevant storage, the adjustment amount is the unsigned gap to the threshold. The final progress assignment still uses the saved threshold if either adjustment dependency changes G-01 or progress.

If progress is equal to or greater than G-01 under the unsigned comparison, no gap adjustment occurs. Terminal handling follows.

### Terminal handling with zero status

F-4e6b58fde38f receives the retained nonzero state value.

When its result is zero, the retained service’s virtual method at byte offset `0xAC`, F-v02, is invoked. Only after that call does the function test the receiver’s narrowed suppression field at `0xDC`.

If suppression is zero:

1. F-9b9c1c7483f4 receives, in positional order:
   - The address of the local slot initially containing the fourth input.
   - The identifier `GotoErrorLayout`.
   - The address of the local slot initially containing the third input.
2. Its returned opaque value is passed to receiver virtual method F-v04 at byte offset `0x11C`.
3. F-8a5aadac202f receives the fourth-input local slot’s address.
4. The function returns 1 immediately.

This return skips the common terminal tail.

If suppression is nonzero, construction and error dispatch are skipped, and the common terminal tail runs.

### Terminal handling with nonzero status

When F-4e6b58fde38f returns nonzero, receiver virtual method F-v03 at byte offset `0xB0` is called.

If its result is zero, F-b3bed2dfc5ec is skipped and no ready handling occurs. The common terminal tail still runs.

For a nonzero result, F-b3bed2dfc5ec receives exactly these four positional arguments:

| Position | Value |
|---|---|
| First | The nonzero result from F-v03. |
| Second | The value read from **G-02**. |
| Third | The value read from **G-03**. |
| Fourth | The 32-bit all-ones value `0xFFFFFFFF`. |

**Argument position does not establish evaluation order.** The supplied representation does not establish whether G-02 is read before G-03 or vice versa. Both supply arguments to this lookup invocation; neither is a cached value obtained before the lookup-source call. Their meanings and signed interpretation are unknown.

If the lookup returns zero, ready handling is skipped and the common terminal tail runs.

For a nonzero lookup result:

1. Test the receiver’s current narrowed suppression field.
2. If suppression is zero, call F-fdd8589eca2d and write the **16-bit value 1** at byte offset `0x80` of its result.
3. Read and narrow the receiver’s current selection field.
4. Select `GeniusPlaylistReady` for zero or `GeniusMixPlaylistReady` for nonzero.
5. Call F-9b9c1c7483f4 with the fourth-input local slot’s address, the selected identifier, and the third-input local slot’s address, in that positional order.
6. Call F-05765260b698 with the receiver, the constructed value, and the nonzero lookup result, in that positional order.
7. Call F-8a5aadac202f with the fourth-input local slot’s address.
8. Run the common terminal tail.

Nonzero suppression skips only F-fdd8589eca2d and the 16-bit write. It does **not** suppress ready construction or dispatch.

The ready identifier uses a fresh selection-field read after the optional 16-bit write. It need not correspond to the provider selected at entry.

### Common terminal tail

Receiver virtual method F-v05 at byte offset `0xB8` receives the receiver and the argument `0x10`. The function then returns 1.

This tail runs after:

- Zero terminal status with nonzero suppression.
- Nonzero terminal status with a zero lookup-source result.
- A zero lookup result.
- Completed ready handling.

It does not run on missing-state paths, adjustment-only paths, or the immediate return after error dispatch.

## Side effects

### Explicit writes

The function explicitly:

- Initializes the third-input and fourth-input local slots.
- Adds 50 to the receiver’s 32-bit progress field.
- On the threshold-gap path, later overwrites progress with the saved threshold.
- On successful lookup with zero suppression, writes exactly 16 bits containing 1 at offset `0x80` of the F-fdd8589eca2d result.

The last assignment is not a 32-bit write. Its explicit width does not authorize modification of adjacent storage; independent dependency effects remain unknown.

### Ordering and mutable state

No dependency is assumed to preserve receiver fields, globals, virtual tables, service state, or referenced storage.

Consequential sequencing includes:

- The initial service query precedes the progress increment.
- Both unconditional calls precede state-dependent predicates and threshold handling.
- The service terminal call precedes error suppression testing.
- The lookup-source call precedes the lookup-global argument reads.
- The relative order of the G-02 and G-03 reads is unspecified.
- Successful lookup precedes ready-path suppression testing.
- The optional F-fdd8589eca2d call and 16-bit write precede ready-message selection.
- Post-dispatch cleanup precedes the common terminal-tail call on the ready path.

The initial service object is retained; the provider is not selected again. Later state-dependent calls use the retained initial state value. Virtual target identities are described by their reached receiver and table offset, not by an assumption that dependencies preserve virtual tables.

### Ownership and lifetime

There is no explicit direct allocation or deallocation operation in this body.

Each reached construction call is followed, after normal dispatch return, by one F-8a5aadac202f call on the fourth-input local slot. This pattern suggests temporary-value cleanup, but it does not establish allocation, reference counting, destruction, release, or ownership transfer.

The contract does not establish that dependencies refrain from retaining local-slot addresses. Those addresses designate invocation-local storage; validity after this invocation is not established. No exception or nonlocal-exit cleanup guarantee is visible.

## Dependencies

Roles below describe only use by F-3faaf6d7241a.

| Opaque ID | Established use |
|---|---|
| F-6da76fc58c80 | Provider selected by an initially zero narrowed selection field. |
| F-af667771d6ce | Provider selected by an initially nonzero narrowed selection field. |
| F-6ac70711bbb5 | First unconditional call after increment. |
| F-5d641e9c768c | Second unconditional call after increment. |
| F-8ddad704d968 | First predicate on retained nonzero state. |
| F-9f27c73ea83b | Second predicate, reached only when the first returns zero. |
| F-6904f7699031 | Embedded-region adjustment with a supplied amount. |
| F-ec4cfa552cce | Embedded-region call following adjustment. |
| F-4e6b58fde38f | Terminal branch selection from retained state. |
| F-b3bed2dfc5ec | Lookup with arguments: source result, G-02 value, G-03 value, `0xFFFFFFFF`. |
| F-fdd8589eca2d | Supplies the target of the conditional 16-bit write. |
| F-9b9c1c7483f4 | Construction using two local-slot addresses and an identifier. |
| F-05765260b698 | Ready dispatch using receiver, constructed value, and lookup result. |
| F-8a5aadac202f | Post-dispatch operation on the fourth-input local slot. |

The following opaque IDs are document-local identities for virtual dependencies, not additional known implementations:

| Opaque ID | Dispatch location | Established use |
|---|---|---|
| F-v01 | Service virtual-table offset `0x9C` | Initial state query. |
| F-v02 | Service virtual-table offset `0xAC` | Zero-status terminal operation. |
| F-v03 | Receiver virtual-table offset `0xB0` | Lookup-source query. |
| F-v04 | Receiver virtual-table offset `0x11C` | Error dispatch. |
| F-v05 | Receiver virtual-table offset `0xB8` | Common terminal operation with argument `0x10`. |

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless stated otherwise, dependencies return normally and preserve relevant fields and globals.

1. **Missing state.** Starting progress is 100 and F-v01 returns zero. Progress becomes 150, F-6ac70711bbb5 precedes F-5d641e9c768c, and the result is 1. No predicates or terminal calls occur.

2. **Fixed adjustment.** State is nonzero, the first predicate returns zero, and the second returns nonzero. F-6904f7699031 receives 50, followed by F-ec4cfa552cce. There is no explicit threshold replacement or terminal call.

3. **Short-circuit predicate.** The first predicate returns nonzero. The second predicate is not called; threshold handling follows.

4. **Threshold gap.** Starting progress is 70 and stable G-01 is 200. After increment, progress is 120. If fixed adjustment is not selected, the adjustment amount is 80, its companion call follows, and progress is explicitly set to 200.

5. **Saved replacement survives mutation.** In the preceding case, an adjustment dependency changes G-01 to 300 and progress to 900. The final explicit progress write is still 200.

6. **Equality.** Starting progress is 150 and stable G-01 is 200. The increment reaches equality. There is no zero-gap adjustment; terminal-status handling begins.

7. **Error dispatch.** Terminal status is zero and suppression remains zero after F-v02. `GotoErrorLayout` is constructed, dispatched through F-v04, and followed by F-8a5aadac202f. F-v05 is skipped.

8. **Suppressed error.** Terminal status is zero and suppression is nonzero after F-v02. No error construction occurs. F-v05 receives `0x10`.

9. **Missing lookup source.** Terminal status is nonzero and F-v03 returns zero. F-b3bed2dfc5ec and ready construction are skipped; F-v05 still runs.

10. **Lookup argument mapping.** F-v03 returns a nonzero value S, G-02 contains 17, and G-03 contains 29. F-b3bed2dfc5ec receives S, 17, 29, and `0xFFFFFFFF`, respectively. Either relative read order for G-02 and G-03 is compatible with the contract.

11. **Ready with suppression.** Lookup succeeds and suppression is nonzero. The accessor and 16-bit write are skipped, but ready construction, dispatch, cleanup, and F-v05 still occur.

12. **Changed selection.** Entry selection is zero, but a dependency changes its narrowed value to nonzero before ready selection. F-6da76fc58c80 supplies the service, while `GeniusMixPlaylistReady` is selected.

13. **Unsigned boundary.** Progress at comparison has bit pattern `0xFFFFFFFF` and G-01 is 200. Progress is not below the threshold under unsigned comparison, so terminal handling follows. This example does not require producing that bit pattern through signed overflow.

## Uncertainties

The following remain **documented_only**:

- Concrete object types, full layouts, pointer width, calling convention, and complete virtual signatures.
- Exact character width and byte order.
- Progress-increment behavior on signed 32-bit overflow.
- Global initialization, volatility, synchronization, and concurrent or asynchronous mutation behavior.
- Relative evaluation order of G-02 and G-03 argument reads; no ordering guarantee is established.
- Application meanings of predicates, lookup arguments, adjustment amounts, and `0x10`.
- Construction input formats, slot mutations, returned-value lifetime, and ownership.
- Dependency effects, pointer retention, exceptions, faults, and nonlocal exits.
- Recovery from invalid receiver, service, virtual-table, or dependency-returned addresses.

The constant normal return does not imply success, validation, retry, rollback, or fault recovery.

## Evidence

Established static evidence consists of the supplied function representation and dependency-ID mapping. It supports the explicit widths, offsets, positional arguments, conditional calls, sequencing between statements, identifiers, and normal return value described here.

In particular, the lookup’s second argument is supplied by G-02 and its third by G-03. That positional mapping is established independently of the unresolved relative evaluation order of those reads.

Role labels express inferred intent only. No runtime tests have been performed. The acceptance examples are conditional consequences of the static behavior, not observed executions.
