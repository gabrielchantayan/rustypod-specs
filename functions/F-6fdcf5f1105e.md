<!-- Trusted provenance: F-6fdcf5f1105e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6fdcf5f1105e

## Purpose

F-6fdcf5f1105e obtains a record address from F-da8adb8947ca, conditionally writes a caller-supplied 32-bit value into that record, performs ordered link-field updates around a previously captured anchor, and increments a 32-bit field in a previously captured context.

The link updates are consistent with inserting a record immediately before an anchor in a doubly linked structure. The final increment is consistent with maintaining an element count. Both descriptions are **inferred intent**. The established behavior is the sequence of accesses below; no surrounding container invariants are supplied.

There is no explicit result value, success indicator, validation path, or rollback.

## Interface

### Arguments and result

| Argument | Represented width | Established use |
|---|---:|---|
| First | 32 bits | Conditionally stored at record offset 8. |
| Second | 32 bits | Unused by the visible body; not forwarded to the dependency. |
| Third | 32 bits | Passed as both the third and fifth arguments to F-da8adb8947ca. |
| Fourth | 32 bits | Passed as the fourth argument to F-da8adb8947ca. |

The function returns without an explicit value. In particular, it does not explicitly return the dependency’s record address or the updated count.

### Widths and addressing

The supplied representation uses four-byte integer fields and four-byte spacing for pointer-valued link fields. The layouts below describe that apparent 32-bit target model.

The conditional payload destination is two four-byte fields beyond the returned record address: byte offset 8. The final increment operates on a represented 32-bit integer field.

The representation alone does not establish all target rules for pointer arithmetic, address wrapping, alignment, signed overflow, or invalid accesses. The unusual condition on the offset-8 address must be preserved rather than replaced with an ordinary record-pointer null check.

### Referenced storage

A global location supplies context addresses during the pre-call portion. The function saves one such address for its final increment and separately captures an anchor address before calling the dependency.

| Region | Byte offset | Established access |
|---|---:|---|
| Captured context | 16 | Source of the captured anchor address, subject to the separate global selection described below. |
| Captured context | 20 | Late 32-bit read, increment, and write. |
| Returned record | 0 | Written with the captured anchor address. |
| Returned record | 4 | Written with a value read from anchor offset 4. |
| Returned record | 8 | Conditionally written with the first argument. |
| Captured anchor | 4 | Read twice after the dependency returns, then written with the record address. |
| Address supplied by the second post-call read of anchor offset 4 | 0 | Written with the record address. |

“Context,” “anchor,” and “record” are descriptive labels, not established declared types or ownership roles.

For ordinary completion, the addresses actually accessed must support the required reads and writes. This includes addresses produced after dependency-side mutation and after any earlier writes that overlap later access locations. The function does not verify those requirements.

## Behavior

### Context and anchor capture

Before invoking F-da8adb8947ca, the function obtains and retains:

1. A context address read from the global location, used later for the offset-20 increment.
2. An anchor address read at offset 16 from a context selected through a separate use of that global location.

The dependency’s first argument is also obtained through the global context location before the call.

If the global value is stable throughout this pre-call sequence, the saved context, the context supplying the anchor, and the context passed to the dependency coincide. The supplied representation contains separate uses of the global location; it does not establish an atomic snapshot or synchronization against concurrent replacement.

After the dependency returns, this function does not reselect the saved context or anchor through the global location.

### Dependency invocation

F-da8adb8947ca is called once with five arguments:

| Position | Supplied value |
|---|---|
| First | Context address selected from the global location for the call |
| Second | Zero |
| Third | This function’s third argument |
| Fourth | This function’s fourth argument |
| Fifth | This function’s third argument again |

The result is treated as the record address.

No supplied dependency body establishes whether that result denotes newly allocated storage, an existing record, initialized storage, or a failure indication. Its ownership and lifetime are also unknown.

The saved context and anchor addresses survive as local selections across this call. That does **not** establish preservation of the memory they designate. The dependency may change their fields, change the global location, or affect object lifetimes; none of those possibilities is excluded by the visible caller.

### Conditional payload write

After the dependency returns, the function tests whether the address at record offset 8 is nonzero.

- If that computed destination address is nonzero, it writes the first argument there as a four-byte value.
- If that computed destination address is zero, it skips this write.

The condition is not a test that the returned record address itself is nonzero.

Under ordinary flat-address interpretation, a zero record address gives an offset-8 destination of address 8, so the condition permits a write attempt rather than recognizing dependency failure. Actual access validity and fault behavior remain environment-dependent.

Skipping this payload write does not skip the subsequent link updates or count increment. There is no separate failure return associated with the condition.

### Ordered link updates

After the conditional payload write, the visible accesses occur in this order:

1. Write the captured anchor address to record offset 0.
2. Read the four-byte value at captured-anchor offset 4.
3. Write that value to record offset 4.
4. Read captured-anchor offset 4 again and interpret this second value as an address.
5. Write the record address at offset 0 of the address obtained by the second read.
6. Write the record address to captured-anchor offset 4.

Both reads of anchor offset 4 occur after the dependency returns. The first does not use a predecessor value saved before the call. The second is a separate access, not an established reuse of the first value.

Under a conventional non-overlapping layout, with a valid predecessor address still present at anchor offset 4 when linking begins, these accesses have the familiar effect of:

- Giving the record a forward link to the anchor.
- Giving the record a backward link to the old predecessor.
- Updating the predecessor’s forward link to the record.
- Updating the anchor’s backward link to the record.

That interpretation depends on the relevant memory contents at the time of the accesses. A structure that was valid before the dependency call is not necessarily unchanged afterward.

Aliasing can also affect the outcome. The payload write and record-field writes precede later reads and may overlap their sources or destinations. The ordered accesses, rather than an assumed list-insertion operation, govern such cases.

### Final increment

Once the link writes finish, the function reads the 32-bit field at offset 20 of the originally saved context, adds one, and writes the result back to that field.

The increment uses the value present at this late point. It does not use a pre-call count snapshot. Consequently:

- Dependency-side changes to that field affect the value incremented.
- Earlier link or payload writes that overlap that field can affect it.
- Replacing the global context during the dependency call does not redirect the increment to the replacement context.

No overflow check is visible. The representation shows a 32-bit integer addition but does not establish a portable outcome for signed overflow. In particular, wrapping at the signed maximum must not be promised without additional target evidence.

The function then returns without an explicit result value.

## Side effects

### Direct memory effects

An execution that reaches all operations performs:

- Zero or one four-byte payload write at record offset 8.
- One four-byte write at record offset 0.
- One four-byte write at record offset 4.
- One four-byte indirect write through the address obtained from the second read of anchor offset 4.
- One four-byte write at anchor offset 4.
- One read and one write for the increment at saved-context offset 20.

These are access counts, not counts of distinct memory locations. Destinations can overlap, later writes can overwrite earlier results, and earlier writes can influence later reads.

The body reads the global context location and contains no direct assignment to that named location. An indirect write could nevertheless reach it if addresses alias. Dependency-side writes are separately unknown.

### Ownership and lifetime

No explicit allocation or release operation appears outside the dependency call. The function installs the returned address into link fields, but that does not prove ownership transfer, exclusive ownership, or an obligation on any particular caller to release it.

The saved context and anchor must remain usable through their post-call accesses for ordinary completion. The function does not retain, validate, or otherwise visibly protect their lifetimes.

Whether F-da8adb8947ca allocates or releases any storage is **documented_only**.

### Failure and concurrency

There are no visible checks for invalid context or anchor addresses, an unusable dependency result, invalid link contents, inconsistent structure state, or count overflow.

If a memory access fails after earlier writes have completed, no visible mechanism undoes those writes. Transactional behavior and an unchanged structure on failure are not established.

No locking, atomic update protocol, or synchronization is visible. Thread safety, interrupt safety, and safe concurrent replacement of the global context are not established.

## Dependencies

### F-da8adb8947ca

**Established use:** one call with the five argument values listed above. Its result is used as the record address.

**Unknown behavior:** allocation, initialization, validation, failure signaling, argument retention, ownership transfer, memory release, global mutation, reachable-memory mutation, reentrancy, and exceptional or non-returning behavior.

The caller supplies no visible guarantee that this dependency preserves the captured anchor’s offset-4 field. Any acceptance example requiring a particular predecessor must constrain that field at the post-call linking point or explicitly assume preservation by the dependency.

If the dependency does not return normally, this function’s normal post-call continuation does not execute.

No other called dependency is visible in the supplied body.

## Acceptance examples

These are conditional consequences of the static representation, not runtime observations.

### Ordinary insertion before an anchor

Assume:

- The global context selection is stable during the pre-call accesses.
- The saved context and anchor remain valid.
- When linking begins, anchor offset 4 contains a valid predecessor address.
- The dependency returns a valid record distinct from the anchor, predecessor, and context.
- The relevant fields do not overlap, and no concurrent mutation occurs.
- The offset-8 destination is nonzero.
- The saved context’s offset-20 field contains 7 when the final increment reads it.

Completion leaves:

- The first argument at record offset 8.
- The anchor address at record offset 0.
- The predecessor address at record offset 4.
- The record address at predecessor offset 0.
- The record address at anchor offset 4.
- The saved context’s offset-20 field equal to 8.

There is no explicit returned success value.

### Self-linked anchor at the linking point

Assume:

- The captured anchor’s offset-4 field contains its own address **when the post-call link sequence begins**. This may be ensured by dependency preservation, but pre-call self-linking alone is insufficient.
- The saved context, anchor, and returned record remain valid for every required access.
- The returned record is distinct from the anchor and context, and the relevant storage does not overlap.
- The conditional payload write does not alter the anchor’s fields.
- No concurrent mutation occurs during linking.

The record receives the anchor address at both offsets 0 and 4. The indirect write targets anchor offset 0 and stores the record address there. The final anchor-link write stores the record address at anchor offset 4.

If the dependency instead changes anchor offset 4 to another address before linking begins, this self-linked outcome is not guaranteed; the function uses the post-call contents.

This example does not establish that self-linking is the only supported empty-state representation.

### Dependency changes the count

Assume the saved context’s offset-20 field is initially 7, the dependency changes it to 12, and subsequent writes do not overlap it.

The final increment reads 12 and writes 13. An outcome based on incrementing the pre-call value to 8 would not match the visible behavior.

### Dependency changes the global context

Assume the dependency replaces the global context address while leaving the originally saved context and captured anchor valid.

The link sequence still uses the captured anchor. The final increment still targets the originally saved context. The replacement global value does not itself redirect either selection.

Relevant field contents may still have changed; retaining addresses does not imply retaining their pre-call contents.

### Zero dependency result

Under ordinary flat-address interpretation, a zero result makes the conditional destination address equal to 8. The function therefore attempts the payload write at address 8 before its unconditional record-field writes.

Whether the attempt faults depends on the execution environment. A clean failure return, successful recovery, and an unchanged structure are not established outcomes.

### Earlier writes overlap a later link source

Assume the returned record address equals the captured anchor address plus 4, all accessed storage is valid, the context is separate, and no other mutation occurs.

The write to record offset 0 then writes the anchor address into anchor offset 4. The first subsequent link read therefore obtains the anchor address, regardless of the predecessor value present before that write. Record offset 4 receives that anchor address.

This case illustrates why conventional non-overlapping insertion results cannot be promised for arbitrary aliasing.

## Uncertainties

The following remain **documented_only**:

- Exact target pointer rules beyond the apparent four-byte representation.
- Address wrapping, alignment requirements, and unusual pointer-arithmetic behavior.
- The dependency result’s validity, extent, initialization, ownership, and lifetime.
- Dependency effects on globals, captured objects, link fields, and the count field.
- Original declared types and semantic field names.
- Required container invariants and permitted aliasing.
- Signed-overflow behavior for the final increment.
- Fault delivery and external recovery mechanisms.
- Thread safety, interrupt safety, and reentrancy.

Inferred insertion and counting intent supplies none of these missing guarantees.

## Evidence

The supplied static body establishes the argument uses, separate pre-call context selections, captured anchor, single dependency call, field offsets, conditional offset-8 write, ordered link accesses, late count increment, and absence of an explicit result or recovery branch.

The conventional linked-structure interpretation follows only under the stated validity, post-call field-content, and non-overlap assumptions. Dependency internals and surrounding invariants were not supplied.

No runtime tests have been performed. All acceptance examples are conditional static deductions, not observed execution results.
