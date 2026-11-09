<!-- Trusted provenance: F-1ccb50dbd69d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1ccb50dbd69d

## Purpose

F-1ccb50dbd69d dispatches a 32-bit selector against a record obtained through a dependency. Recognized selectors either:

- Read selected 32-bit record fields, convert each through a dependency, and submit the resulting values to an output dependency.
- Copy one selected field into an address-exposed, invocation-local 32-bit cell and invoke a processing dependency.
- Perform two such processing calls, followed by conversion and submission of seven record fields.

The function returns the 32-bit value **1** after completing a recognized branch, or **0** for an unrecognized selector. These values indicate dispatch recognition and normal completion, not independently verified success of the dependencies.

The static evidence establishes field offsets, operation order, argument forwarding, and return constants. It does not establish the record’s domain meaning, the output representation, or the dependencies’ ownership and error contracts.

## Interface

### Inputs

The function accepts four 32-bit arguments, described here by role rather than source identifier:

| Position | Contract name | Established use |
|---|---|---|
| First | Processing context | Forwarded only to F-3cab9feba403. |
| Second | Output target | Forwarded to F-2484a607f41b and F-3cab9feba403. |
| Third | Selector | Compared against the recognized values 0 through 19. |
| Fourth | Resolution context | Used to initialize the local cell; forwarded to F-838218ef65c4, F-e4e0c15c0e4c, and F-3cab9feba403. |

The supplied representation does not establish whether these values are pointers, handles, integers, or another kind of 32-bit token. Their operational roles do not justify stronger public types.

### Output

The return value is 32 bits:

- **1:** The selector is recognized and its branch reaches its normal return.
- **0:** The selector is unrecognized and execution reaches the default return.

There is no explicit return path for allocation failure, invalid memory, conversion failure, output rejection, or processing failure. Dependency failures may prevent normal completion; their behavior is unknown.

### Record and local-cell requirements

F-838218ef65c4 returns a value subsequently used as a record base address. The function performs 32-bit reads at byte offsets from that base.

For a selected branch to complete normally, its referenced record locations must support the required reads when they occur. Selector 18 accesses the highest starting offset, **0xD4**; a contiguous record supporting every selector would therefore require accessible bytes through **0xD7**, or **0xD8 bytes** in total. This is a derived access extent, not evidence of an allocation size.

An invocation-local 32-bit cell is initialized from the resolution context. Its address is passed to F-838218ef65c4. Later writes to that same cell are significant to the execution trace, even though subsequent processing calls do not explicitly receive its address.

## Behavior

### Common initialization

Every selector, including an unrecognized selector, executes this prelude:

1. Initialize the local 32-bit cell with the resolution context.
2. Call F-838218ef65c4 with:
   - The resolution context.
   - A 32-bit value obtained from a fixed global datum.
   - The numeric argument **0x80**, or **128**.
   - The address of the local cell.
3. Retain the return value as the record base.
4. Read a separate fixed global 32-bit datum and retain its value as the **captured conversion token**.
5. Dispatch on the selector.

The meaning of the lookup datum and the argument 128 is unknown. In particular, 128 is not established as a record size. Several branches read beyond byte offset 127.

F-838218ef65c4 may write the local cell through its supplied address. The function does not validate the returned record base before dispatch. An unrecognized selector does not directly dereference that base, but still executes the lookup and conversion-token read.

### Conversion and submission branches

Selectors 0 through 5 read one or more 32-bit fields, convert each value, and submit each conversion result.

For every selected field, the observable sequence is:

1. Read the 32-bit value from the record.
2. Call F-e4e0c15c0e4c with the resolution context, a conversion token, and that field value.
3. Call F-2484a607f41b with the output target and the conversion result.
4. Only after that submission call returns, proceed to the next field, if any.

The return value of F-2484a607f41b is not used.

| Selector | Field starting offsets, in order | Conversion calls | Submission calls |
|---|---|---:|---:|
| 0 | 0x00 through 0x2C, spaced by 4 bytes | 12 | 12 |
| 1 | 0x30 through 0x5C, spaced by 4 bytes | 12 | 12 |
| 2 | 0x60 through 0x78, spaced by 4 bytes | 7 | 7 |
| 3 | 0x7C through 0x94, spaced by 4 bytes | 7 | 7 |
| 4 | 0x98 | 1 | 1 |
| 5 | 0x9C | 1 | 1 |

Selectors 0 through 3 use the captured conversion token throughout the branch. They do not directly reread the global token between elements.

Selectors 4 and 5 read their record field and use a **fresh read of the conversion-token global** for their single conversion. The earlier pre-dispatch read still occurs. No dependency call intervenes between these two global-token reads, but the distinction remains relevant if that storage can change asynchronously.

Record fields are read incrementally, not copied into a complete snapshot before conversion begins. A dependency’s mutation of a later field can therefore affect a subsequent field read. The function does not reacquire the record base.

Each branch returns 1 after its final submission call returns normally.

### Single processing branches

Selectors 6 through 18 each read one record field, store it into the local cell, and call F-3cab9feba403 once.

| Selector, decimal | Record field offset |
|---:|---:|
| 6 | 0xA4 |
| 7 | 0xB0 |
| 8 | 0xA8 |
| 9 | 0xB4 |
| 10 | 0xB8 |
| 11 | 0xBC |
| 12 | 0xC0 |
| 13 | 0xC4 |
| 14 | 0xC8 |
| 15 | 0xCC |
| 16 | 0xAC |
| 17 | 0xD0 |
| 18 | 0xD4 |

The offset for selector 14 is **200 decimal**, equivalent to **0xC8**.

After the selected value has been stored, F-3cab9feba403 receives, in order:

1. Processing context.
2. Resolution context.
3. Record base.
4. Output target.

The local cell’s value is not an explicit argument to this call. Its earlier address exposure is established; any mechanism by which the processing dependency observes that cell is not.

The processing dependency’s return value is ignored. The function returns 1 if the call returns normally. These branches make no direct calls to F-e4e0c15c0e4c or F-2484a607f41b, although F-3cab9feba403 may have its own effects.

### Composite branch

Selector 19 performs the following sequence:

1. Read the 32-bit field at offset **0xB4**.
2. Store that value into the local cell.
3. Call F-3cab9feba403 with the four arguments listed above.
4. After that call returns, read the field at offset **0xD0**.
5. Replace the local cell’s value with this newly read value.
6. Call F-3cab9feba403 again with the same explicit argument values.
7. Read, convert, and submit seven fields at offsets **0x7C through 0x94**, spaced by four bytes, in increasing address order.
8. Return 1.

The seven conversions use the conversion token captured before selector dispatch. They do not refresh it after either processing call.

The read at 0xD0 occurs after the first processing call. The seven output-field reads occur after both processing calls. Consequently, mutations by those calls can affect the later record reads. The record base itself is not reacquired.

### Unrecognized selectors

Every 32-bit selector value outside 0 through 19 takes the default branch.

After the common initialization, that branch returns 0 without:

- A direct record-field read.
- A call to F-e4e0c15c0e4c.
- A call to F-2484a607f41b.
- A call to F-3cab9feba403.

It is therefore not a side-effect-free rejection path: F-838218ef65c4 has already run.

## Side effects

### Directly established effects

The function directly:

- Initializes an invocation-local 32-bit cell.
- Exposes that cell’s address to F-838218ef65c4.
- Reads the fixed lookup datum and conversion-token datum.
- Reads selector-dependent record fields.
- Overwrites the local cell for selectors 6 through 19.
- Invokes dependencies in the order specified above.

No direct write to the record or a global datum is shown.

All record accesses are 32-bit reads at byte offsets that are multiples of four. Alignment of the record base, byte order, and target behavior for misaligned or invalid addresses are not established. The bounded field traversals use four-byte spacing and small fixed counts; no count is derived from record contents.

### Dependency effects and ownership

No allocation, release, reference-count adjustment, or ownership transfer is directly established.

The following must remain unspecified:

- Whether the returned record is borrowed, shared, newly allocated, or otherwise managed.
- Whether a conversion result is an object, handle, pointer, or scalar.
- Whether submission consumes, retains, copies, or releases that result.
- Whether any dependency mutates globals, record storage, output state, or other reachable state.
- Whether a dependency retains the local cell’s address.

The function does not explicitly release either the record or conversion results. That absence is not enough to establish a leak or an ownership convention.

If the local cell’s address is retained beyond the invocation, lifetime safety depends on an external contract not present in the supplied evidence.

## Dependencies

| Opaque function ID | Established interaction | Unspecified behavior |
|---|---|---|
| **F-838218ef65c4** | Called once before dispatch with the resolution context, fixed-global lookup value, 128, and local-cell address. Its return is used as the record base. | Lookup meaning, validity guarantees, local-cell writes, side effects, ownership, failure behavior. |
| **F-e4e0c15c0e4c** | Receives the resolution context, a conversion token, and one 32-bit field. Its result is passed to the next submission call. | Conversion semantics, result representation, allocation, errors, global and record mutations. |
| **F-2484a607f41b** | Receives the output target and the immediately preceding conversion result. Its return is ignored. | Output ordering beyond call order, retention, consumption, failure reporting, other effects. |
| **F-3cab9feba403** | Receives the processing context, resolution context, record base, and output target after a selected local-cell write. Its return is ignored. | Interpretation of the selection, access to exposed local state, output effects, record or global mutations, ownership. |

These dependencies are not assumed to preserve globals or record contents.

## Acceptance examples

These examples are conditional acceptance criteria derived from static evidence. They are not reports of executed tests. Each assumes the relevant reads are valid and the dependencies return normally.

### Twelve-field conversion

For selector 1, the direct trace contains twelve conversion/submission pairs. The record-field reads occur at offsets 0x30, 0x34, and onward through 0x5C.

Each submission receives the result of the immediately preceding conversion. The function returns 1.

### Incremental reads under mutation

For selector 0, suppose the first submission call changes the record field at offset 0x04 before returning. The next conversion receives the value read from offset 0x04 after that mutation, not a value snapshotted before the first conversion.

This example assumes the record remains valid.

### Nonmonotonic selector mapping

For selector 16, the function selects offset 0xAC, not an offset extrapolated from selectors 15 and 17.

It stores the field into the local cell, calls F-3cab9feba403 once, and returns 1.

### Composite processing order

For selector 19, suppose the first processing call changes the field at 0xD0. The second local-cell assignment uses the field’s value after that call.

The seven conversion/submission pairs occur only after the second processing call returns, and they use the pre-dispatch captured conversion token even if either processing call changed the token global.

### Single-field token read

For selector 4, the record value comes from offset 0x98. The token argument comes from the branch’s fresh global read, rather than necessarily from the earlier captured value.

If the global remains unchanged between reads, both values coincide; this does not eliminate the two-read distinction.

### Unrecognized selector

For selector 20, F-838218ef65c4 still runs once, and the common conversion-token read still occurs. No direct record-field conversion, submission, or processing call follows. The normal return value is 0.

### Dependency failure indication

If a submission dependency returns a value that its own contract defines as failure, this function does not inspect it. If execution otherwise continues normally, the recognized branch still returns 1.

This does not specify behavior for a dependency that faults, unwinds, or does not return.

## Uncertainties

### Inferred intent

The access pattern suggests groups of record fields exposed through selector-driven output, with other selectors delegated to a processing routine. The local-cell writes may communicate a selected value through state established by the initial lookup.

Those interpretations are inferred intent, not established semantics. They do not identify field names, units, types, or output formats.

### documented_only unknowns

The supplied evidence does not establish:

- The meaning of either fixed global datum.
- The meaning of the numeric argument 128.
- The record’s declared layout, allocation extent, or field semantics.
- Byte order, alignment guarantees, or invalid-address behavior.
- The dependencies’ error, ownership, mutation, or lifetime contracts.
- How F-3cab9feba403 could observe the local-cell value.
- Whether concurrent mutation is permitted or synchronized.
- Whether hidden dependency state makes repeated invocations interact.
- Whether any supplied representation omits additional machine-level constraints.

The function contains no explicit validation that resolves these unknowns. This contract therefore does not invent null handling, bounds rejection, retries, rollback, cleanup, or transactional output.

## Evidence

Established static evidence consists solely of the supplied function representation and opaque dependency mapping.

That evidence supports:

- Four 32-bit inputs and a 32-bit return.
- An unconditional pre-dispatch dependency call and conversion-token read.
- Recognized selectors 0 through 19.
- The listed 32-bit field offsets and traversal counts.
- Interleaved conversion/submission call order.
- Local-cell writes preceding processing calls.
- Two processing calls before seven conversions for selector 19.
- Ignored processing and submission return values.
- Constant normal returns of 1 for recognized branches and 0 for the default branch.

No dependency bodies, runtime traces, or independent layout declarations were supplied. **No runtime tests have been performed.**
