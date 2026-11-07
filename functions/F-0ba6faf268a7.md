<!-- Trusted provenance: F-0ba6faf268a7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0ba6faf268a7

## Purpose

F-0ba6faf268a7 performs a synchronous dispatch surrounded by independently conditional operation groups. Two bytes in a caller-supplied object control the groups before and after an indirect method call. Each selector enables its group only when its value is exactly 1.

The established behavior comprises ordered dependency calls, conditional writes to a shared byte destination, an unconditional indirect invocation, and a notification sequence enabled by an indirect result equal to 1. Every visible normal-return path returns the 32-bit value zero.

The surrounding groups suggest preparation and restoration, but that interpretation is not an established guarantee. The function does not visibly enforce balanced execution, restore a saved byte value, validate its inputs, or handle failures.

## Interface

### Arguments and relevant object layout

| Input | Established use |
|---|---|
| Object address | Provides two byte selectors and interior addresses passed to dependencies. |
| Context word | An opaque 32-bit value forwarded unchanged to selected dependencies and the indirect method. |

The object is not copied. Relevant offsets are measured in bytes:

| Offset | Use |
|---|---|
| 0 | One-byte selector controlling the second initial group and second later group. |
| 1 | One-byte selector controlling the first initial group and first later group. |
| 16 | Interior address supplied to four dependencies associated with the offset-1 groups. |
| 40 | Interior address supplied to three dependencies associated with the offset-0 groups. |

Each selector is tested twice, at separate points in execution. There is no entry-time snapshot of both selector values. Values other than 1, including 2 and 255, do not enable a group. Whether a byte is interpreted as signed or unsigned does not change this equality condition.

The addresses at offsets 16 and 40 are formed by adding byte displacements to the object address. They do not represent reads of the contents at those offsets by this function. The sizes, alignments, and internal layouts required by the receiving dependencies are unknown.

### Shared inputs

Three distinct shared inputs participate:

- **Shared-byte pointer:** a global pointer whose value is captured before the first selector test. When the initial offset-1 group executes, the global pointer is read again to obtain the destination for the write of 1. The later write of 0 instead uses the entry-captured pointer.
- **Dispatch-argument pointer:** a shared pointer dereferenced to obtain an argument for the indirect method.
- **Notification argument:** a shared value read for the final notification call.

The two reads of the shared-byte pointer are separate operations. The source does not establish that they produce equal addresses. Capturing the pointer does not read the destination byte, and neither conditional write depends on that byte’s previous value.

### Return value and errors

The visible normal return is a 32-bit zero. The indirect method’s result is retained for a later comparison, not returned to the caller.

There is no visible error return, input rejection, null check, pointer validation, or target validation. Faults, abnormal exits, or nonreturning dependencies are not converted into a specified result.

### Preconditions for normal completion

The execution environment must provide valid memory and callable targets for the path taken:

- Selector bytes must be readable at each test.
- Each destination actually used for a byte write must be writable.
- The object’s interior addresses must satisfy the requirements of their recipients.
- The dispatch-argument pointer and its referenced storage must support the indicated read.
- The provider-returned object, its table reference, and the selected table entry must support the indirect invocation.
- Invoked dependencies must return normally for execution to reach the final return.

The visible offsets do not establish a sufficient allocation size for the caller’s object. Address arithmetic must produce valid addresses for its uses; the function exposes no overflow or bounds handling.

## Behavior

### Entry capture and initial offset-1 group

Before testing either selector, the function reads and retains the shared-byte pointer.

It then reads the object byte at offset 1. If that byte is not 1, the initial offset-1 group is skipped. If it is 1, the following sequence occurs:

1. The shared-byte pointer is read again from its global storage.
2. The byte value 1 is written through this newly read pointer.
3. F-680bd70ebd5c is called with no visible arguments.
4. F-50328f2e60da is called with no visible arguments.
5. Its result selects one of two alternatives:
   - For zero, F-30801baebbd2 supplies an opaque 32-bit value, then F-97ef388f4f7f receives that value, zero, and the context word.
   - For nonzero, F-30801baebbd2 supplies an opaque 32-bit value, then F-e630fadb44b4 receives that value and the context word.
6. F-4f181c08df34 receives the object address plus 16 bytes.
7. F-466811e3e0c0 receives the same interior address.

No dependency call is shown between the entry pointer capture and the second pointer read. Nevertheless, pointer stability is not established. In particular, the contract does not collapse these two reads into a single snapshot.

Only zero versus nonzero matters for the predicate result. Its magnitude does not select further behavior. Other return values from this group are not visibly inspected.

### Initial offset-0 group

After the first group completes or is skipped, the function reads the object byte at offset 0. If it is exactly 1, it performs:

1. F-33f2700caeca with the object address plus 40 bytes.
2. F-5e87f77b7f19 with that same address.
3. F-30801baebbd2 to obtain a fresh opaque 32-bit value.
4. F-97ef388f4f7f with that value, zero, and the context word.

This selector read occurs after the initial byte write and all calls in the initial offset-1 group. Both dependency effects and aliasing by the function’s own write can therefore affect whether this group runs.

### Unconditional indirect dispatch

The function calls F-U01 with no visible arguments. It uses the returned address as a dispatch object:

- The object’s first represented integer-sized field supplies the table reference.
- A callable entry is obtained at byte displacement 24 from that table reference.
- The selected method receives the provider-returned object address, the value obtained by dereferencing the shared dispatch-argument pointer, and the original context word.

The indirect result is saved for the final comparison.

This dispatch occurs regardless of all selector values. There is no visible null or validity check.

The established table displacement is 24 bytes, not a portable method-slot number. Deriving a slot number requires an independently established entry width and table layout.

The dispatch argument is obtained for this invocation after the provider call. The source does not establish an ordering among the table lookup and other argument-evaluation reads within the indirect-call expression.

### Later offset-1 group

After the indirect method returns, the function rereads the object byte at offset 1. If it is exactly 1, it performs:

1. F-680bd70ebd5c.
2. F-50328f2e60da.
3. A new F-30801baebbd2 call, followed by:
   - F-97ef388f4f7f with the returned value, zero, and the context word when the new predicate result is zero; or
   - F-e630fadb44b4 with the returned value and the context word when that result is nonzero.
4. A one-byte write of zero through the shared-byte pointer captured at entry.
5. F-b06a84dae121 with the object address plus 16 bytes.
6. F-740936998497 with that same address.

The predicate is queried again; the earlier result is not reused. The two offset-1 groups may therefore select different context-related dependencies.

The zero write does not reload the global shared-byte pointer. A dependency changing that global pointer does not redirect this particular write. The destination is the entry-captured address.

No previous destination-byte value is restored. Zero is written unconditionally once this group reaches the write.

### Later offset-0 group

The function next rereads the object byte at offset 0. If it is exactly 1, it calls:

1. F-3609a8b2bc1a with no visible arguments.
2. F-f08c49572cd8 with no visible arguments.
3. F-U02 with the object address plus 40 bytes.

This test occurs after the later offset-1 group, including its zero write. Earlier calls or an aliased write may change its outcome.

Execution of this group does not require that the initial offset-0 group ran.

### Notification and return

After both later groups complete or are skipped, the saved indirect result is compared with exactly 1.

For a matching result:

1. F-8fc1650ac01f receives the integer constant 5.
2. F-U03 receives the shared notification argument and zero.

The notification argument is read after F-8fc1650ac01f returns. Its value must not be assumed to match any earlier shared-state value.

For every other indirect result, both notification calls are skipped. The function then returns the 32-bit value zero.

## Side effects

### Direct writes and aliasing

The only explicit data stores visible in the body are one-byte writes:

- A write of 1 through a fresh global-pointer read in the initial offset-1 group.
- A write of 0 through the entry-captured pointer in the later offset-1 group.

There are no explicit stores addressed through the object’s selector fields. This does **not** establish that the selectors remain unchanged: either shared-byte destination may alias a selector.

Aliasing may also affect other state used later. No non-aliasing relationship is established between the byte destinations, the caller’s object, or shared storage. Consequently, even dependencies that leave selectors unchanged are insufficient by themselves to guarantee stable selector values.

The two writes are independently conditional. Possible outcomes include neither write, only the write of 1, only the write of 0, or both writes. When both occur, their destinations need not be equal.

### Dependency effects, ownership, and lifetime

The function visibly passes the caller’s object interior addresses and opaque returned values to dependencies. It does not establish whether recipients read, modify, retain, release, or otherwise interpret those values.

No ownership transfer, allocation, deallocation, lock acquisition, or resource release can be concluded from call ordering alone. In particular:

- The lifetime of values returned by F-30801baebbd2 is unknown.
- Ownership and lifetime of the F-U01 object are unknown.
- F-U02 cannot be classified as a release operation from this body.
- No dependency is assumed to preserve globals or selector bytes.
- No local allocation is visible.

### Failure and interruption

No visible cleanup mechanism guarantees completion of the later groups after failure. A fault or abnormal exit following the write of 1 can leave that write without a later zero write.

Even normal completion does not guarantee paired groups, matching write destinations, or restoration of prior state.

## Dependencies

Opaque IDs describe call sites, not independently verified callee semantics.

| ID | Established role |
|---|---|
| F-680bd70ebd5c | Precedes each offset-1 predicate query. |
| F-50328f2e60da | Supplies each independent zero/nonzero predicate result. |
| F-30801baebbd2 | Supplies an opaque 32-bit value for a subsequent context-related call. |
| F-97ef388f4f7f | Receives that value, zero, and the context word. |
| F-e630fadb44b4 | Receives that value and the context word for a nonzero offset-1 predicate result. |
| F-4f181c08df34 | First initial offset-16 operation. |
| F-466811e3e0c0 | Second initial offset-16 operation. |
| F-33f2700caeca | First initial offset-40 operation. |
| F-5e87f77b7f19 | Second initial offset-40 operation. |
| F-b06a84dae121 | First later offset-16 operation, after the zero write. |
| F-740936998497 | Second later offset-16 operation. |
| F-3609a8b2bc1a | First later offset-0 operation. |
| F-f08c49572cd8 | Second later offset-0 operation. |
| F-8fc1650ac01f | Receives 5 when the saved indirect result equals 1. |

Three dependencies lack supplied opaque mappings. The following are document-local IDs only:

| ID | Established interface |
|---|---|
| F-U01 | No visible arguments; returns the dispatch-object address. |
| F-U02 | Receives the interior address at offset 40. |
| F-U03 | Receives the shared notification argument and zero. |

The indirect method has no supplied fixed identity; its target is obtained through the provider-returned object.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless an example states otherwise:

- All dependencies return normally.
- Dependencies and external actors leave selectors and relevant shared state unchanged.
- Both possible shared-byte destinations are disjoint from the selectors and all other storage on which the stated expectations depend.
- Required memory and callable targets are valid.

### Both selectors disabled

With both selectors zero:

- The entry pointer capture still occurs.
- All selector-controlled groups are skipped.
- No second shared-byte-pointer read for the initial write occurs.
- F-U01 and the indirect method still execute.
- No direct byte write occurs.
- A saved indirect result other than 1 causes no notification.
- The function returns zero.

Selector values 2 or 255 also leave their respective groups disabled.

### Only offset 1 enabled

With offset 1 equal to 1 and offset 0 equal to zero:

- The initial group rereads the shared-byte pointer and writes 1 before F-680bd70ebd5c.
- Both initial offset-16 calls precede dispatch.
- The later group makes a fresh predicate query.
- Zero is written through the entry-captured pointer before F-b06a84dae121.
- No offset-40 dependency runs.

If the first predicate result is zero and the second is nonzero, the initial group uses F-97ef388f4f7f and the later group uses F-e630fadb44b4.

The absence of offset-40 calls depends on the stated non-aliasing condition, not merely on dependencies preserving selectors.

### Only offset 0 enabled

With offset 0 equal to 1 and offset 1 equal to zero:

- F-33f2700caeca and F-5e87f77b7f19 precede the initial F-30801baebbd2 and F-97ef388f4f7f calls.
- Dispatch follows.
- F-3609a8b2bc1a, F-f08c49572cd8, and F-U02 then execute in order.
- No direct byte write occurs.

### Own writes change the offset-0 selector

Override the non-aliasing assumption: both shared-byte-pointer reads identify the object byte at offset 0. Initially, offset 0 is zero and offset 1 is 1. Dependencies leave the selectors unchanged.

The initial write changes offset 0 to 1, enabling the initial offset-0 group. The later write changes offset 0 back to zero before its later test, so the later offset-0 group is skipped.

This behavior follows from the function’s own writes, without a dependency modifying either selector.

### Distinct write destinations

Suppose the entry pointer read yields destination A, while the second read yields destination B, with both offset-1 tests observing 1. Assuming valid destinations and no relevant aliasing:

- The write of 1 targets B.
- The later write of 0 targets A.

The source does not guarantee the conditions under which the two pointer reads could differ; this example specifies the resulting behavior if they do.

### Selectors change during dispatch

If dispatch changes offset 1 from 1 to zero, the initial group runs but the later offset-1 group is skipped. No corresponding zero write is issued by this function.

If dispatch changes it from zero to 1, the later group runs without the initial group. Its zero write still uses the pointer captured at entry.

### Notification boundary

A saved indirect result of exactly 1 causes F-8fc1650ac01f with 5, followed by F-U03 with the then-read notification argument and zero. Results of zero, 2, or a negative integer do not enable that sequence.

Every normally completing example returns zero.

## Uncertainties

### Established static evidence

The supplied representation establishes byte selector accesses, exact equality conditions, two distinct shared-byte-pointer reads, conditional byte writes, interior offsets, inter-call order, indirect-table displacement, the saved-result comparison, and the constant normal return.

The context, normal return, and F-30801baebbd2 values are explicitly represented as four-byte quantities. Selectors and direct stores are one byte.

### Inferred intent

The surrounding groups resemble preparation and restoration, and the byte writes resemble an activity indicator. These interpretations do not establish balance, ownership, synchronization, or recovery semantics.

### documented_only unknowns

- Independently verified pointer width, target ABI, integer-result width, and table-entry representation.
- Full object layout, alignment, bounds, and dependency-specific region requirements.
- Aliasing relationships among byte destinations, selectors, and shared storage.
- Callee semantics, ownership, retention, failures, and global effects.
- Concrete identities of the document-local dependencies and indirect target.
- Concurrency, reentrancy, atomicity, and memory-order guarantees.
- Machine-level read scheduling beyond the supplied representation.
- Ordering among reads within a single call expression where not independently established.
- Abnormal-exit mechanisms not represented in the body.

## Evidence

This contract is derived solely from the supplied static function representation and dependency-ID mapping. Embedded comments, prior prose, apparent pairing, and naming were not treated as proof of behavior.

No runtime tests, memory observations, independent implementation access, or dependency-body inspections were performed. Acceptance examples are conditional consequences of the visible operations, not observed execution results.
