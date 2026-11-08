<!-- Trusted provenance: F-31e8feff54e1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-31e8feff54e1

## Purpose

F-31e8feff54e1 replaces a 32-bit value in an object, conditionally changes a one-byte indicator, invokes dependencies associated with the replacement, and processes the displaced value through metadata validation and accounting updates.

The established order is:

1. Store the replacement.
2. Process the indicator and any replacement-related calls.
3. If the displaced value is nonzero, validate its metadata and perform the accounting and release-related sequence.

Replacement processing precedes displaced-value validation. A later failure does not trigger any visible rollback.

**Inferred intent:** the stored value appears to identify a resource, the indicator appears to represent an active state, and the displaced-value processing resembles allocator accounting and release. These interpretations do not establish ownership, resource type, synchronization, or the behavior of any dependency.

## Interface

### Inputs and representation

The function accepts:

- An object address.
- A 32-bit replacement value.

Zero and nonzero replacements select different paths. The function does not otherwise classify or directly validate the replacement.

The supplied representation uses 32-bit object words and addresses, unsigned 16-bit metadata selectors, and an 8-bit indicator. The complete calling convention and target byte order are not established.

### Directly accessed object layout

| Byte offset | Width | Role |
|---|---:|---|
| `0x00` | 32 bits | Dispatch-table address |
| `0x34` | 32 bits | Stored value being replaced |
| `0x39` | 8 bits | State indicator |

For a nonzero replacement, the dispatch table supplies a callable entry at byte offset `0x58`.

That callback receives three arguments:

1. The object address.
2. The value read from one global data operand.
3. The value read from a second, distinct global data operand.

The second-argument and third-argument globals occupy distinct roles and are not interchangeable. Their concrete values, declared types, widths, and meanings are not supplied. They are not established constants or immutable data.

Both global operands are evaluated for the callback after the conditional transition calls. Those calls can therefore affect the argument values if they mutate the relevant globals. The relative evaluation order among the callback target and its argument operands is not established.

### Metadata associated with a nonzero displaced value

A nonzero displaced value is treated as an address with preceding metadata:

| Offset relative to displaced address | Width | Role |
|---|---:|---|
| `−8` | 32 bits | Header used for validation and accounting amounts |
| `−4` | 16 bits, unsigned | First accounting selector |
| `−2` | 16 bits, unsigned | Second accounting selector |

The metadata base supplied to a later dependency is the displaced address minus eight bytes.

### Output

There is no explicit return value. Normal completion is observable through memory writes and calls.

The invalid-header dependency F-a817d8d58c10 is represented as non-returning. Continued execution after that call is outside the established normal behavior.

### Preconditions

Normal completion requires accessible storage for every selected-path read and write, including:

- The relevant object fields.
- The dispatch table, callback entry, and two callback-argument globals for a nonzero replacement.
- Metadata preceding a nonzero displaced value.
- The global reference used to obtain the accounting context.
- The accounting locations selected by metadata and by F-09dcb55e8e0a.

Dependencies also have their own undocumented preconditions.

There is no direct null-object check, replacement-address validation, selector bounds check, or counter-range check.

## Behavior

### Replacement is stored first

The function reads and retains the 32-bit value at object offset `0x34`, then writes the replacement to that field.

The retained displaced value determines whether later displaced-value processing occurs and which address it uses. Subsequent dependency mutations of the object’s stored-value field do not replace that retained value.

The function does not compare the replacement with the displaced value. An identical-value assignment is not a special case.

### Zero replacement

For a zero replacement, the function reads the indicator at object offset `0x39`.

If the indicator is zero, it makes no transition-related call.

If the indicator is nonzero, it:

1. Writes zero to the indicator.
2. Calls F-62fe32c4ec09 with the object address.
3. Calls F-146cf87a17c7 without explicit arguments if F-62fe32c4ec09 returns nonzero.

A zero result from F-62fe32c4ec09 suppresses F-146cf87a17c7.

Every nonzero initial indicator value takes this transition path; the initial value need not be one. The zero-replacement branch does not invoke the dispatch-table callback or evaluate its two global argument operands.

### Nonzero replacement

For a nonzero replacement, the function reads the indicator.

If the indicator is zero, it:

1. Writes one to the indicator.
2. Calls F-62fe32c4ec09 with the object address.
3. Calls F-89365abd7cb1 without explicit arguments if F-62fe32c4ec09 returns nonzero.

A zero result suppresses F-89365abd7cb1.

If the indicator was already nonzero, the function does not directly rewrite it and does not invoke either transition dependency. In particular, it does not normalize a noncanonical nonzero indicator to one.

After this conditional processing, the function invokes the dispatch-table entry at offset `0x58`. The callback receives the object followed by values from the two distinct global operands, in their respective second-argument and third-argument positions.

The dispatch lookup and global operand evaluations belong to this callback operation, after any transition calls have returned. They are not captured before those calls. Consequently:

- A transition dependency can affect the dispatch target by changing the object’s dispatch-table reference or relevant table contents.
- A transition dependency can affect either callback argument by changing its corresponding global.
- Distinct global operands do not imply distinct argument values.
- No relative evaluation order is established among the callback target and argument operands.

The callback occurs for every nonzero replacement, including an already-nonzero indicator and an identical nonzero replacement.

### Displaced-value processing begins after replacement processing

If the retained displaced value is zero, the function completes after the replacement-related branch, assuming all invoked dependencies return.

For a nonzero displaced value, it calls:

1. F-187ab9ba4eb6 without explicit arguments.
2. F-2fd8826557a4 without explicit arguments.

Their return values are not used.

Only after both calls does the function read the global accounting-context reference to obtain a 32-bit context address. That address is retained for the remaining operations. Earlier transition calls, the callback, and these preliminary dependencies are not assumed to preserve the global reference.

### Header validation

The function checks the 32-bit header eight bytes before the displaced address.

It invokes F-a817d8d58c10 if:

- The header is zero; or
- The header’s least significant bit is set.

The zero test occurs first. If it does not reject the header, a separate header read supplies the low-bit test. This is not an established single-snapshot validation under concurrent mutation.

Rejection happens after the replacement write, indicator processing, any nonzero-replacement callback, both preliminary displaced-value calls, and the accounting-context read. No earlier effects are undone.

If validation accepts, the function calls F-c0fc6d033caa with the retained accounting-context address. Accounting selectors and amounts are read afterward; the dependency is not assumed to preserve them.

### Accounting updates

The following sequence uses the retained accounting-context address. Every updated field is 32 bits wide.

| Order | Operation |
|---:|---|
| 1 | Read the unsigned 16-bit first selector at displaced offset `−4`. |
| 2 | Read the header and retain its value masked with `0x3ffffffc` as an accounting amount. |
| 3 | Subtract that amount from context offset `0x224 + 4 × first selector`. |
| 4 | Subtract the same retained amount from context offset `0x30c`. |
| 5 | Read the unsigned 16-bit second selector at displaced offset `−2`. |
| 6 | Subtract the retained amount from context offset `0xe0 + 4 × second selector`. |
| 7 | Subtract the retained amount from context offset `0x220`. |
| 8 | Call F-09dcb55e8e0a without explicit arguments. |
| 9 | Decrement the field at context offset `0x310 + 4 × dependency result`. |
| 10 | Decrement the field at context offset `0x390`. |
| 11 | Read the header again, mask this fresh value with `0x3ffffffc`, and subtract the resulting amount from context offset `0x08`. |

The four amount-based updates before F-09dcb55e8e0a share one retained amount. The final amount-based update uses a fresh header read after that dependency returns and after the two decrements.

The amounts are equal only if the relevant header bits remain unchanged. Validation does not freeze or revalidate later header values.

Each update reads its target’s then-current value and writes the arithmetic result. The targets are not established to be distinct. If they alias, updates accumulate in sequence rather than operating on one initial snapshot.

Within the final subtraction expression, the relative order of the accounting-field read and fresh header read is not established.

### Final calls

After all seven accounting updates, the function calls:

1. F-cc6c9b4a0f68 with the retained accounting context and displaced metadata base.
2. F-e586bff64fb7 with the retained accounting context.

It then completes if those dependencies return.

The function contains no direct restoration of the displaced value, no final rewrite of the replacement, and no post-call enforcement of the indicator value.

## Side effects

### Direct writes and dependency effects

The function directly performs:

- One unconditional 32-bit replacement write at object offset `0x34`.
- A conditional 8-bit write of zero or one at object offset `0x39`.
- Seven 32-bit accounting updates on a successfully validated displaced-value path.

Dependencies can add writes, change later-read data, or prevent completion. No preservation guarantee is established for object fields, metadata, globals, or accounting storage.

The two callback-argument globals are directly read, not directly written, by this function. Their values can nevertheless differ from their entry-time values because preceding dependencies may mutate them.

### Ownership and lifetime

**Established:** every nonzero retained displaced value enters the metadata-validation and accounting sequence, even when equal to the replacement.

**Inferred intent:** the sequence relinquishes a displaced resource while installing a replacement.

**Not established:**

- Whether the replacement is borrowed, retained, copied, or ownership-transferred.
- Whether the callback acquires ownership.
- Whether F-cc6c9b4a0f68 frees memory, recycles storage, or performs another operation.
- Whether dependencies keep the metadata and context accessible throughout the sequence.
- Whether replacing a value with itself is safe.

No ownership or lifetime guarantee follows merely from dependency call positions.

### Widths and arithmetic

Each unsigned 16-bit selector ranges from zero through 65,535. Multiplication by four produces an offset from zero through 262,140 before addition of the fixed field offset and context address.

The mask `0x3ffffffc` retains bits 2 through 29. Its result ranges from zero through `0x3ffffffc` and is divisible by four. A header can pass validation while producing zero accounting amount; for example, `0x00000002` is nonzero and low-bit-clear but masks to zero.

The accounting operations update 32-bit words. No saturation, underflow check, or arithmetic error report is shown. The representation does not establish a guarantee of mathematically nonnegative counters or a complete source-language overflow model.

F-09dcb55e8e0a supplies a 32-bit result used in four-byte-scaled address arithmetic without a visible range check. Its permitted range and address validity requirements remain undocumented.

### Failure and concurrency

The sequence is not visibly transactional. Invalid memory access, dependency failure, or a non-returning call can leave earlier effects observable.

No atomicity, locking, or concurrency guarantee can be assigned to dependencies from their positions alone. Multiple reads of metadata or globals need not observe entry-time values.

## Dependencies

| Opaque ID | Established role at this call site |
|---|---|
| F-62fe32c4ec09 | Receives the object after a direct indicator transition; zero versus nonzero result controls another call |
| F-146cf87a17c7 | Called after a transition to zero when the preceding result is nonzero |
| F-89365abd7cb1 | Called after a transition to one when the preceding result is nonzero |
| F-187ab9ba4eb6 | First preliminary call for a nonzero displaced value |
| F-2fd8826557a4 | Second preliminary call, before accounting-context acquisition |
| F-a817d8d58c10 | Represented non-returning invalid-header path |
| F-c0fc6d033caa | Receives the context after validation and before accounting reads and updates |
| F-09dcb55e8e0a | Supplies the index for one decrement location |
| F-cc6c9b4a0f68 | Receives the context and displaced metadata base after accounting |
| F-e586bff64fb7 | Final call using the retained context |

The indirect callback has no supplied opaque identity. Its established interface is the object address followed by values read from the two distinct global operands. Its behavior, argument interpretation, and preservation properties are unknown.

## Acceptance examples

These are conditional contract examples, not executed tests. They assume accessible storage and returning dependencies except where stated otherwise.

### Zero replacement without transition or release

With a zero displaced value and zero indicator, a zero replacement writes zero to the stored-value field and completes without transition, dispatch, or displaced-value calls.

### Clearing a noncanonical indicator

With a zero replacement and indicator seven, the function writes zero before calling F-62fe32c4ec09. A nonzero result causes F-146cf87a17c7; a zero result suppresses it.

The dispatch callback and its global argument reads do not occur on this branch.

### Activating with changed callback globals

Suppose the replacement is nonzero, the indicator is zero, and the displaced value is zero. Initially, the second-argument global contains 11 and the third-argument global contains 22.

Suppose F-62fe32c4ec09 changes the first global to 33 and returns nonzero, then F-89365abd7cb1 changes the second global to 44. If those values remain stable during callback evaluation, the callback receives the object, 33, and 44—not the globals’ entry-time values.

This example does not impose an evaluation order among the callback target and its argument operands.

### Already-active indicator

With a nonzero replacement and indicator seven, the function does not directly normalize the indicator or invoke transition dependencies. It still invokes the callback using the two global values evaluated for that call.

### Invalid displaced header

With a nonzero displaced value whose header remains zero through validation, replacement processing occurs first. Both preliminary displaced-value dependencies run, the context is acquired, and F-a817d8d58c10 is invoked. Under its represented non-returning behavior, no accounting updates follow.

### Stable valid metadata

Suppose the header remains `0x00000024`, the selectors remain two and three, and F-09dcb55e8e0a returns four. Assume valid, distinct accounting fields and no dependency interference.

The amount is 36. The function subtracts 36 at context offsets `0x22c`, `0x30c`, `0xec`, and `0x220`; decrements offsets `0x320` and `0x390`; then subtracts 36 at offset `0x08`. The two final dependencies follow.

### Header changes during accounting

Suppose the first accounting header read yields `0x24`, while F-09dcb55e8e0a changes the header to `0x40`, which remains stable afterward. With otherwise noninterfering storage, the first four amount-based updates subtract 36, while the final update subtracts 64. There is no intervening header validation.

### Identical nonzero replacement

Identical nonzero displaced and replacement values do not suppress processing. The callback still occurs, followed by displaced-value validation and accounting. The contract does not establish that the resulting resource lifetime is safe.

## Uncertainties

The following remain **documented_only**:

- Object type and semantic meanings of its fields.
- Exact types, widths, values, and meanings of the two callback-argument globals.
- Relative evaluation order among callback target and argument operands.
- Callback behavior and dependency preservation guarantees.
- Accounting-context provenance and lifetime.
- Valid selector ranges narrower than their storage widths.
- Valid results from F-09dcb55e8e0a.
- Ownership and actual memory-release behavior.
- Complete ABI, byte order, and address-overflow semantics.
- Error, reentrancy, and concurrency behavior of dependencies.
- External restrictions on aliasing among object, metadata, globals, and accounting storage.
- Whether identical-value replacement is intended to be supported.

## Evidence

This contract is based solely on static examination of the supplied function representation and opaque dependency mapping.

Established evidence includes access widths and offsets, branch conditions, ordered calls, direct writes, distinct global operands for the callback’s second and third arguments, their evaluation after conditional transition calls, repeated header reads, and the absence of visible rollback or equality suppression.

Resource ownership, active-state terminology, and allocator-like roles are inferred intent, not proven dependency contracts. No runtime tests have been performed.
