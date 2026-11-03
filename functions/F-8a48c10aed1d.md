<!-- Trusted provenance: F-8a48c10aed1d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8a48c10aed1d

## Purpose

F-8a48c10aed1d collects four-byte values from iterator-provided records into a temporary table of 15 slots. A record contributes only if its one-byte selector identifies a slot from 0 through 14. Each accepted record directly replaces the contents of its selected slot.

After collection ends, the function finalizes the iterator, passes the table to an intermediate dependency, and passes that same table to a final dependency together with the caller’s second argument and two fields read from the caller’s object.

The apparent intent is indexed collection followed by downstream processing. The evidence does not establish the application meaning of the table, records, object fields, or downstream operations. In particular, it does not establish that table entries are pointers or that either table-processing dependency validates, sorts, releases, or preserves them.

This contract describes static evidence. No runtime tests have been performed.

## Interface

### Arguments and result

The function accepts two arguments:

| Argument | Represented form | Observable use |
|---|---|---|
| Object address | Integer-held address in the supplied representation | Base for three four-byte field reads |
| Forwarded value | Opaque four-byte quantity | Passed unchanged as the second argument to the final dependency |

The function has no explicit return value. It does not produce a success flag, propagate a dependency status, or construct a caller-visible return structure.

Any externally meaningful outcome depends on dependency side effects. Their details are not supplied.

### Accessed object layout

| Offset from object base | Width | Use and timing |
|---|---:|---|
| 0xA8 | 4 bytes | Passed by value to iterator initialization, after the initial table operation |
| 0xC4 | 4 bytes | Passed as the fourth argument to the final operation |
| 0xC8 | 4 bytes | Passed as the third argument to the final operation |

Offset 0xC8 is decimal 200.

The function does not take a snapshot of these fields at entry. The 0xA8 field is read before iteration; the other two fields are read after iteration, iterator finalization, and the intermediate table operation. Earlier dependency effects can therefore influence later field reads.

### Accessed record layout

For each advance result that continues iteration, the supplied record address is used as follows:

| Offset from record base | Width | Access condition |
|---|---:|---|
| 0x10 | 1 byte | Read for every continuing advance result |
| 0x08 | 4 bytes | Read only when the selector is accepted |

The four-byte value is copied without an established semantic conversion or dereference. No other record fields are directly accessed.

### Temporary storage and arithmetic

The invocation supplies:

- A contiguous table of 15 four-byte slots, totaling 60 bytes.
- An eight-byte iterator-state area.
- An integer-held record-address output location supplied to the advance dependency.

The supplied representation uses 32-bit integer operations and four-byte opaque values. This establishes the widths relevant to table indexing and value copying in the represented model; it does not establish a complete portable ABI.

An accepted selector identifies a slot at a byte displacement of four times the selector. The largest displacement is 56, for slot 14, whose four bytes occupy table offsets 56 through 59. The accepted index arithmetic therefore remains within the declared 60-byte table and requires no large intermediate value.

Object and record address additions have no visible overflow or validity checks. Pointer representation, alignment requirements, and behavior at address-space boundaries remain documented_only.

### Preconditions for normal execution

Normal execution requires:

- Each object field to be readable when its read occurs.
- Every continuing advance result to leave a usable record address in the output location.
- The selector byte at that record address to be readable.
- The four-byte record value to be readable whenever its selector is accepted.
- Dependencies to use supplied temporary storage within valid bounds and lifetimes.
- Address calculations to designate valid storage under the execution environment.

The function provides no explicit null-address, object-size, record-size, alignment, or address-overflow validation.

## Behavior

### Initial table operation

The first dependency call is F-EXTERNAL-01. It receives the table address and the numeric value 60.

This call precedes the read of the object field at 0xA8. Consequently, an object change caused through an alias or shared state during this call can affect the value subsequently supplied to iterator initialization.

The dependency’s semantics are unknown. Its arguments are consistent with an operation over the table’s 60-byte extent, but they do not establish that it clears the table, initializes every byte, or even interprets the second argument as a byte count.

No result from this operation is visibly tested.

### Iterator initialization

After the initial operation returns, the function reads four bytes at object offset 0xA8. It calls F-8ca21d3a8cca with:

1. The iterator-state address.
2. The four-byte value just read.

The object field is passed by value, not by its address.

No initialization result is visibly tested. The function proceeds to iterator advancement if initialization returns normally.

### Record collection and its two decisions

The function repeatedly calls F-b38a3aba0e18 with the iterator-state address and the record-address output location.

Two distinct decisions control collection:

- **Advance-status decision:** A zero return ends the loop. Any nonzero return enters record processing.
- **Selector-range decision:** Within record processing, the selector determines whether the payload is read and the table is written.

There is no separate negative-status error path. A negative nonzero advance result, where representable by the dependency’s result type, is treated as continuation.

On each continuing result, the function first obtains the record address from the output location and reads the selector byte at record offset 0x10. It accepts precisely the byte values 0 through 14.

The selector expression includes a character conversion and both nonnegative and upper-bound checks. These do not admit high-bit selector bytes: if represented as negative characters they fail the nonnegative check; if represented as nonnegative values from 128 through 255 they fail the upper-bound check. Thus the accepted byte set remains 0 through 14.

For an accepted selector, the function reads four bytes at record offset 0x08 and writes that value to the corresponding table slot. The selector read precedes the payload read, and the payload read precedes the table assignment.

For a rejected selector, the function makes no direct payload read and no direct table assignment. Rejection does not terminate collection; the next advance call follows.

### Repeated and unassigned slots

Successive accepted records selecting the same slot cause successive assignments to that slot. The later direct assignment replaces the earlier one.

This is a rule about this function’s writes, not a guarantee about table contents surviving arbitrary dependency effects. Advance calls occur between record assignments, and dependencies are not established to preserve the table through aliases or retained addresses.

A slot never selected by an accepted record receives no direct collection assignment. Its contents depend on the initial operation and any dependency effects. No zero default is established.

There is no visible record-count limit. Normal loop termination depends on F-b38a3aba0e18 eventually returning zero.

### Iterator finalization

After a zero advance result, the function calls F-d0d3f100effc with the iterator-state address.

This call occurs even if the first advance result is zero. The function does not directly read the output record address or any record fields on that terminating result.

The call’s placement establishes a post-iteration operation, but not its resource semantics. Whether it releases resources, invalidates records, mutates shared state, or performs other work is unknown.

No finalization result is visibly tested.

### Intermediate table operation

After iterator finalization returns, F-fe6a65ef0b1e receives the table address. No explicit length accompanies this call.

Its interpretation of the table and its mutation behavior are unknown. The function makes no preserved copy of the collection-phase table for the final operation. The final dependency receives the same storage after this intermediate call returns.

No result from the intermediate operation is visibly tested.

### Final operation and return

After the intermediate operation returns, the function reads the four-byte object fields at 0xC8 and 0xC4. It invokes F-5fe1270f6b01 with these arguments in these positions:

1. Table address.
2. Original forwarded four-byte argument.
3. Value read at object offset 0xC8.
4. Value read at object offset 0xC4.

Both object-field reads occur after the intermediate call. The supplied call expression does not establish their relative evaluation order. Argument positions must not be interpreted as proof of left-to-right memory reads or an atomic snapshot of the two fields.

The final dependency’s result is not visibly tested. After it returns normally, the function returns without further direct processing.

## Side effects

### Direct accesses

The function directly reads the three object fields, the record-address output location on continuing iterations, each processed selector, and accepted-record payloads.

Its direct table writes are limited to four-byte assignments for accepted selectors. It contains no direct assignment to object fields or record contents.

These limits describe the visible function body. They do not guarantee that dependencies leave object fields, records, table contents, or globals unchanged.

### Dependency ordering

On a normally returning execution, the dependency order is:

1. Initial table operation.
2. Iterator initialization.
3. Repeated iterator advances, with conditional record reads and table writes between them.
4. Iterator finalization.
5. Intermediate table operation.
6. Final operation.

The 0xA8 object read occurs between the first two calls. The 0xC8 and 0xC4 reads occur between the last two calls.

No dependency is established to preserve shared state. These sequencing points therefore remain significant even when a dependency’s internal behavior is unknown.

### Ownership and lifetime

The table, iterator state, and record-address output location are temporary storage belonging to this invocation. Dependencies receive addresses into that storage.

The visible body establishes no ownership transfer, heap allocation, explicit deallocation, or reference-count adjustment. Copying a four-byte record value into a table slot does not itself establish ownership of any resource that value might represent.

Iterator finalization must not be equated with releasing record storage or other resources without additional evidence.

The temporary addresses have no established usable lifetime after this function returns. Retaining them for later access would require a lifetime mechanism not shown here. The contract does not assume dependencies copy their contents or refrain from retaining their addresses.

### Errors and exceptional control flow

Among dependency return values, only the iterator-advance result is visibly tested. Separately, the selector-range check controls whether a payload read and table write occur.

There is no visible status-handling branch for initial operation failure, initialization failure, finalization failure, or failure of either downstream operation. There is also no independent validation of a record address returned with a nonzero advance status.

If a dependency fails to return normally, the remaining calls are not established to occur. In particular, no abnormal-exit path visibly guarantees iterator finalization after an interruption during collection.

Invalid memory accesses and nonlocal control transfers have no defined recovery behavior in the supplied evidence.

## Dependencies

| Opaque function ID | Visible invocation contract | Semantics not established here |
|---|---|---|
| F-EXTERNAL-01 | Receives table address and numeric value 60 before the 0xA8 object read | Initialization, extent interpretation, mutations, ownership, failures |
| F-8ca21d3a8cca | Receives eight-byte iterator-state address and the four-byte value from object offset 0xA8 | Construction, validation, allocation, state invariants |
| F-b38a3aba0e18 | Receives iterator-state address and record-address output location; zero stops, nonzero continues | Record ordering, output guarantees, status meanings, alias effects |
| F-d0d3f100effc | Receives iterator-state address after normal loop termination | Cleanup, resource release, invalidation, other effects |
| F-fe6a65ef0b1e | Receives table address after iterator finalization | Table interpretation, mutations, failures |
| F-5fe1270f6b01 | Receives table address, forwarded value, field at 0xC8, and field at 0xC4 | Externally meaningful action, ownership, mutations, failures |

F-EXTERNAL-01 is a document-local opaque identifier for the initial external dependency. Its identity is not supplied by the dependency mapping.

## Acceptance examples

These are conditional consequences of the static behavior, not executed tests. Unless stated otherwise, dependencies return normally and required accesses are valid.

### Empty iteration

If the first advance returns zero, no record address or record field is directly read for processing, and collection makes no table assignment.

Iterator finalization, the intermediate operation, and the final operation still occur in that order. An all-zero final table is not required.

### Lowest and highest accepted slots

If continuing advances supply selectors 0 and 14, their payloads are directly assigned to table bytes 0–3 and 56–59 respectively.

Both selectors are accepted. The function neither rejects slot 14 nor writes a sixteenth slot.

### Rejected selector with unreadable payload

If a valid record has readable selector 15 but its payload region is unreadable, the collection path does not directly access that payload. It makes no table assignment for that record and proceeds to another advance.

The same direct-access rule applies to selector bytes 0x80 and 0xFF. This example does not guarantee that dependencies themselves avoid the payload region.

### Duplicate selector

If two processed records select slot 4 with payloads A and B, the function directly assigns A and later B.

If no intervening dependency changes that slot, its value immediately after the second assignment is B. Its value at the final call remains conditional on subsequent dependency effects.

### Negative nonzero advance status

If an advance returns a negative nonzero status and supplies a usable record address, the function processes that record’s selector. The status is not treated as termination or as a separately handled error.

If the same status leaves an unusable record address, safe continuation is not established.

### Late object mutation

If the intermediate dependency changes the object’s 0xC8 field and that change remains in effect until its read, the final dependency receives the changed value as its third argument.

The corresponding rule applies to 0xC4 and the fourth argument. No relative order or atomicity of these two reads is required.

### Intermediate table mutation

If F-fe6a65ef0b1e replaces a table slot before returning, the final dependency receives the same table storage containing that replacement, absent another intervening effect. It does not receive a preserved pre-intermediate copy.

## Uncertainties

The following remain documented_only:

- Whether the initial external operation initializes all, part, or none of the table.
- The semantic types of object fields, record payloads, and table entries.
- Iterator ordering, stability, and guarantees about output addresses.
- Whether advance statuses encode errors despite all nonzero results taking the continuation path.
- Dependency allocation, release, ownership, aliasing, retention, and global-state behavior.
- The meaning and allowed mutations of the intermediate and final operations.
- Complete ABI details, including address representation, endianness, and alignment.
- The relative evaluation order of the final two object-field reads.
- Behavior following invalid accesses, abnormal dependency exits, or nonlocal control transfer.

The indexed-collection interpretation is inferred intent. It does not resolve these unknowns or justify stronger guarantees about defaults, cleanup, validation, or final output.

## Evidence

Established static evidence is the supplied function representation and opaque dependency mapping. It shows:

- Fifteen four-byte table elements and eight bytes of iterator state.
- An initial external call with the table address and numeric value 60.
- Object-field reads at 0xA8, 0xC8, and 0xC4.
- An advance-result branch that continues on nonzero and terminates on zero.
- A separate selector-range branch accepting only byte values 0 through 14.
- A selector read at record offset 0x10 before any conditional payload read.
- A four-byte payload read at record offset 0x08 and a corresponding indexed table assignment only on the accepted path.
- Iterator finalization, intermediate table processing, and the final four-argument call in that order.

No dependency implementations or runtime observations were supplied. The acceptance examples express conditional requirements derived from the static evidence; they do not claim observed execution.
