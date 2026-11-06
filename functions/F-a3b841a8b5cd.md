<!-- Trusted provenance: F-a3b841a8b5cd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a3b841a8b5cd

## Purpose

F-a3b841a8b5cd performs one of two call sequences according to whether a signed 32-bit selector equals −1. After the selected sequence returns normally, it performs three indirect calls through the primary object's dispatch table.

The established contract concerns branch selection, dispatch locations, explicit arguments, temporary-storage exposure, and the ordering of calls and global reads. Application-level meanings are unknown. The evidence does not establish that the selector is an index, that the temporary storage owns a resource, or that the final calls are notifications.

A critical distinction in the final sequence is that one global cell is read twice before the second primary call: first to retain a value for the third call, then separately to supply an argument to the second call. Those two values are not unconditionally equal.

## Interface

The function accepts four inputs and exposes no specified return value.

| Input | Representation | Established use |
|---|---|---|
| Primary object | Pointer represented by a 32-bit pointer-bearing word | Supplies the secondary-object pointer and the dispatch table used by the final calls |
| Selector | Signed 32-bit integer | Compared with −1 and explicitly forwarded on the alternate branch |
| Payload A | Uninterpreted 32-bit value | Copied into local temporary storage |
| Payload B | Uninterpreted 32-bit value | Copied into local temporary storage |

All offsets below refer to the supplied 32-bit representation. They do not define a corresponding layout for a platform with wider pointers.

### Object and dispatch layouts

The primary object provides these accessible fields:

| Byte offset | Required contents |
|---|---|
| +0x00 | Pointer to the primary dispatch table |
| +0x158 | Pointer to the secondary object |

The secondary-object field is at word position 0x56; multiplying by the four-byte word width gives byte offset 0x158.

The secondary object provides its own dispatch-table pointer at byte offset +0x00. Dispatch entries are addressed by byte offsets within their respective tables, not by the numerical offsets interpreted as entry counts.

The primary table is used at +0x58. The secondary table is used at +0x38 and +0x5C on the sentinel branch, or +0x110, +0x13C, +0x44, and +0xDC on the alternate branch.

These requirements do not establish complete object sizes, types, inheritance relationships, or meanings for other fields.

### Temporary storage

Before selecting a branch, the function initializes four local 32-bit words in this order:

| Relative byte offset | Initial value |
|---|---|
| +0x00 | Copy of the primary-object pointer |
| +0x04 | Copy of the selector |
| +0x08 | Payload A |
| +0x0C | Payload B |

The supplied representation places these words consecutively. On the alternate branch, dependencies receive the address of the first word. No record type, ownership discipline, or callee-required extent is established merely by that address being passed.

### Preconditions for normal execution

Normal execution requires readable primary-object fields, valid dispatch tables and compatible callable entries whenever they are used. Secondary-object storage must remain valid for each reached secondary dispatch.

On the sentinel branch, the global context-root cell and the storage reached through it must support the specified reads.

Callbacks may change object fields or globals. Accordingly, validity is required at each later access, not just at entry. The function performs no explicit validation of pointers, table entries, selector ranges, or global-backed storage.

Invalid inputs have no specified error result. The evidence does not establish a particular fault, exception, or recovery behavior.

## Behavior

The following ordering applies when all earlier dependencies return normally.

### Initialization and branch selection

The four input values are copied into the temporary storage before branch selection.

The selector is compared exactly with signed −1, whose 32-bit representation is all ones:

- −1 selects the sentinel branch.
- Every other signed 32-bit value selects the alternate branch.

There is no additional range check, normalization, increment, or selector-based address calculation. In particular, −2 and the minimum signed 32-bit value both select the alternate branch. The comparison itself introduces no arithmetic-overflow boundary.

### Sentinel branch

When the selector equals −1:

1. The function obtains the secondary object from primary-object offset +0x158, obtains its dispatch-table pointer, and invokes the entry at table offset +0x38. No explicit arguments are displayed. The result is not explicitly consumed.

2. It resolves the secondary object and its dispatch table again, then invokes the entry at +0x5C with these explicit arguments:
   - The secondary-object pointer.
   - Integer 1.
   - Integer 0.

3. After those calls return, it reads a pointer from a global context-root cell. It then reads the 32-bit value at byte offset +0x30 from the pointed-to storage and passes that value as the sole explicit argument to **F-3a461e364722**.

The first two dependencies may affect the global state subsequently read by the third operation. The function does not capture the context-derived value before those calls.

This branch does not explicitly expose the temporary-storage address and does not invoke F-ee5919ca4be6. The displayed results of its dependencies are not inspected.

### Alternate branch

When the selector differs from −1:

1. The function resolves the secondary object and its dispatch table, then invokes the entry at +0x110. No explicit arguments are displayed.

2. It resolves the secondary object and dispatch table again, then invokes the entry at +0x13C with:
   - The temporary-storage address.
   - The secondary-object pointer.
   - The original selector.

3. It resolves the secondary object and dispatch table again, then invokes the entry at +0x44 with:
   - The secondary-object pointer.
   - The same temporary-storage address.

4. It resolves the secondary object and dispatch table again, then invokes the entry at +0xDC. No explicit arguments are displayed.

5. It invokes **F-ee5919ca4be6** with the same temporary-storage address as its sole explicit argument.

The function does not restore or reinitialize the temporary words between these calls. Changes made through the exposed address can therefore remain visible to later dependencies receiving that address.

The original primary-object input remains the basis for subsequent object lookups and final dispatches. Changing the temporary copy of that pointer does not itself replace the original input. Likewise, the +0x13C call explicitly receives the original selector, rather than a selector reread from the temporary region.

Results from the displayed dependencies are not explicitly inspected.

### Common final sequence

After either branch completes normally, the function invokes the primary dispatch entry at +0x58 three times.

For this contract, **A**, **B**, **C**, **D**, and **E** designate five global 32-bit value cells. These labels imply no application meaning.

**First primary call**

The explicit argument sequence is:

1. Original primary-object pointer.
2. Value read from A.
3. Value read from B.

**Capture followed by second primary call**

After the first call returns, the function reads C and retains that value locally for later use.

It then performs a separate read of C as an argument to the second primary call. That call receives:

1. Original primary-object pointer.
2. Value from this second, separate read of C.
3. Value read from D.

The retained value is not the displayed source of the second call's C-derived argument. There are two reads, in order: the capture first, then the argument read.

**Third primary call**

After the second call returns, the third call receives:

1. Original primary-object pointer.
2. The value retained from the first read of C.
3. Value read from E.

There is no displayed third read of C for this call.

Consequently:

- The second and third calls receive equal C-derived values only if the two reads of C yield equal values.
- C remaining unchanged between those reads is sufficient for equality.
- A change to C during the second call does not, by itself, alter the third call's retained argument.
- E is read for the third call after the second call returns.

The primary dispatch table is resolved separately for each call. A dependency that changes the primary object's dispatch-table pointer may therefore change a later call target.

Within an individual call expression, the supplied representation does not establish the relative evaluation order of the dispatch-target lookup and independent argument reads. It does establish that the retained read of C precedes the second call's argument read of C, and that the first call has returned before either read occurs.

After the third call returns normally, the function returns without a specified result. Exact machine-level properties of the final transfer remain uncertain.

## Side effects

### Directly established writes

The function initializes the four local temporary words and stores the captured C value locally.

It contains no explicit assignment to primary-object fields, secondary-object fields, the global context root, or global value cells A through E. This absence of direct writes does not imply that those locations remain unchanged across dependencies.

### Dependency-mediated changes

No preservation guarantee is established for any dependency. Dependencies may affect globals, either object's fields, dispatch tables, or temporary contents accessible to them.

Repeated secondary-object lookups are significant. If a completed dependency replaces the pointer at primary-object offset +0x158 with another valid secondary object, the next secondary dispatch resolves through the replacement. The function does not establish a single cached secondary receiver for the entire branch.

Within an individual call, target resolution and a separately displayed secondary-object argument are not established as an atomic snapshot under concurrent mutation. Synchronization behavior remains unknown.

### Lifetime and ownership

The temporary region is local to the invocation. The function does not provide persistent caller-managed storage for it or establish that retaining its address beyond the invocation is safe.

Passing its address does not establish allocation, release, ownership transfer, or borrowing conventions. In particular, the position of F-ee5919ca4be6 after the alternate branch's indirect calls does not prove that it is a destructor or cleanup function.

Whether any dependency retains an address or releases an object is **documented_only**.

### Errors and abnormal control flow

There is no explicit status check, error return, retry, rollback, or exception handler.

If a dependency does not return normally, later operations are not guaranteed to execute. F-ee5919ca4be6 is reached only after the preceding alternate-branch calls return normally; it is not established as unconditional cleanup.

## Dependencies

The supplied identity mapping establishes two direct dependencies:

| Opaque function ID | Invocation contract visible here |
|---|---|
| **F-3a461e364722** | Sentinel branch; receives the 32-bit value read through the global context root at offset +0x30 |
| **F-ee5919ca4be6** | Alternate branch; receives the temporary-storage address after four secondary dispatches |

The indirect targets have no established opaque identities. Their lookup locations and explicit arguments are:

| Dispatch table | Byte offset | Explicit arguments, in order |
|---|---:|---|
| Secondary | +0x38 | None displayed |
| Secondary | +0x5C | Secondary object, 1, 0 |
| Secondary | +0x110 | None displayed |
| Secondary | +0x13C | Temporary-storage address, secondary object, original selector |
| Secondary | +0x44 | Secondary object, temporary-storage address |
| Secondary | +0xDC | None displayed |
| Primary | +0x58 | Original primary object, two 32-bit values |

“No explicit arguments displayed” does not establish a complete machine-level signature. Hidden receiver state, register arguments, calling conventions, and target-specific effects remain **documented_only**.

## Acceptance examples

These are conditional criteria derived from static evidence, not runtime observations.

### Sentinel selection

Given selector −1, valid storage and callable entries, and normally returning dependencies:

- Secondary dispatch +0x38 precedes +0x5C.
- F-3a461e364722 follows those calls.
- No alternate-branch dispatch or F-ee5919ca4be6 call occurs.
- The three primary +0x58 calls follow.

A context-root or context-field change made by either secondary dependency can affect the value subsequently supplied to F-3a461e364722.

### Non-sentinel negative selector

Given selector −2 under the same validity conditions, the alternate branch executes, and +0x13C receives −2 as its third explicit argument. No clamping or rejection is performed by this function.

The minimum signed 32-bit selector has the same branch outcome.

### Temporary-storage continuity

Suppose +0x13C modifies the temporary word initially containing payload A and returns normally. The +0x44 dependency receives the same address without caller-side restoration.

F-ee5919ca4be6 later receives that same region, including any intervening changes. This criterion does not specify which changes those dependencies actually make.

### Secondary-object replacement

Suppose +0x110 replaces the secondary-object pointer with another valid pointer and returns normally. If that pointer remains stable while the next call is prepared, +0x13C resolves through the replacement and receives it as the secondary-object argument.

No ownership or release behavior for the old object follows from this replacement.

### Equal values from the two C reads

Suppose C contains X when captured after the first primary call and remains unchanged until the second call's separate argument read:

- The second call receives X.
- The third call receives retained X.

If the second call then changes C to Y and E to Z, the third call still receives X in its C-derived position and receives Z from E, provided E remains Z until its read.

Equality here depends on the two reads of C producing the same value; it is not guaranteed merely because the third call uses a retained value.

### Different values from the two C reads

Conditionally, if the first read of C yields X and the later argument read yields Y:

- The second primary call receives Y.
- The third primary call receives retained X.

The evidence does not establish a mechanism causing such a change between reads or define synchronization for concurrent access. The criterion specifies which read supplies each argument, not that a differing-read execution is guaranteed.

### Nonreturning dependency

If alternate-branch +0x44 does not return normally, no guarantee is established that +0xDC, F-ee5919ca4be6, or any final primary call executes.

## Uncertainties

The following remain **documented_only**:

- Application-level meaning of the selector, payloads, global cells, and operations.
- Complete signatures and calling conventions of dependencies.
- Identities of dynamically resolved targets.
- Ownership, allocation, release, and address-retention behavior.
- Whether dependencies mutate global state, object fields, or dispatch tables.
- Whether the adjacent temporary words form a formal aggregate expected by callees.
- Fault, exception, and nonlocal-exit behavior.
- Synchronization requirements and behavior under concurrent mutation.
- Relative evaluation order of independent target and argument reads within a call.
- Exact machine-level control-transfer behavior of the final indirect operation.

“Selector-dependent work followed by common follow-up calls” is a possible intent description only. It adds no semantic guarantees.

## Evidence

The established behavior comes from the supplied static function representation:

- An exact comparison of a signed 32-bit selector with −1.
- Initialization of four local 32-bit words before branch selection.
- Secondary-object access at primary-object offset +0x158.
- The specified indirect-dispatch offsets and explicit argument sequences.
- Two direct dependencies with supplied opaque identities.
- Three common primary dispatches at +0x58.
- After the first primary call, a read of C retained locally, followed by a separate read of C supplying the second call's argument.
- Explicit use of the retained first C value only in the third primary call.
- No displayed validation, status inspection, rollback, or specified return value.

The supplied representation flags uncertainty in recovery of the final indirect transfer. Its displayed arguments support the third operation described here; exact transfer mechanics remain unresolved.

No runtime tests have been performed. All acceptance examples are conditional static-contract examples, not observed executions.
