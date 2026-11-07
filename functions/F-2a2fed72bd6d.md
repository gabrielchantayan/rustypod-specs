<!-- Trusted provenance: F-2a2fed72bd6d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2a2fed72bd6d

## Purpose

F-2a2fed72bd6d conditionally obtains an object address, validates metadata preceding that address, updates accounting locations associated with a captured context, and passes the metadata address to a dependency.

The established behavior is a sequence of conditional calls, memory reads, and seven accounting writes. Allocation-release bookkeeping is an inferred purpose, not a proven ownership contract. Actual memory reclamation and synchronization are not established.

Two entry conditions control whether processing begins: an incoming zero-condition flag must be clear, and an explicit integer argument must be nonzero. The explicit argument is otherwise unused.

## Interface

### Inputs

- **Explicit argument:** represented as a signed 32-bit integer. Only its zero/nonzero classification affects this function. Positive and negative nonzero values take the same path.
- **Incoming zero-condition flag:** implicit processor state consulted before the explicit argument. A set flag causes immediate return. Whether this represents a genuine external calling requirement is unresolved.
- **Global context reference:** a memory location from which the function captures a context address after its preliminary dependency calls.
- **Dependency results:** F-aabc9d8e982e supplies an object address; F-09dcb55e8e0a supplies an accounting index.

Use **P** for the nonzero object address and **C** for the captured context address. These are descriptive notation, not additional parameters.

### Outputs

The function has no specified return value. It can:

- Return without obtaining an object.
- Return after the object-producing dependency supplies zero.
- Complete the accounting and final dependency calls, then return.
- Invoke the designated non-returning error dependency.

There is no explicit success indicator, validation result, or accounting-underflow report.

### Metadata layout

All offsets are in bytes.

| Address | Access width | Interpretation established by use |
|---|---:|---|
| P − 8 | 32 bits | Header checked for zero and bit 0; later masked for accounting |
| P − 4 | Unsigned 16 bits | First context-relative accounting index |
| P − 2 | Unsigned 16 bits | Second context-relative accounting index |

The metadata address passed to a final dependency is P − 8, not P.

### Context-relative destinations

| Address | Access width | Operation |
|---|---:|---|
| C + 0x224 + 4 × first metadata index | 32 bits | Subtract the first captured quantity |
| C + 0x30c | 32 bits | Subtract that same quantity |
| C + 0xe0 + 4 × second metadata index | 32 bits | Subtract that same quantity |
| C + 0x220 | 32 bits | Subtract that same quantity |
| C + 0x310 + 4 × dependency-provided index | 32 bits | Decrement by one |
| C + 0x390 | 32 bits | Decrement by one |
| C + 0x08 | 32 bits | Subtract a separately read masked quantity |

These formulas identify accesses, not proven independent fields or bounded arrays. Some destinations can coincide with one another or overlap metadata unless an external layout contract excludes that possibility.

### Preconditions for ordinary completion

The body does not check context validity, accessible metadata extent, index bounds, destination accessibility, alignment, or available accounting balances. It does not establish that dependencies preserve the lifetime or contents of required memory.

Ordinary completion therefore requires externally supplied conditions sufficient for each executed access and arithmetic operation. The supplied evidence does not define those conditions completely, including address-overflow and signed-overflow behavior.

## Behavior

### Entry suppression and object acquisition

If the incoming zero-condition flag is set, return immediately. No explicit dependency call or application-memory access shown in the body occurs.

Otherwise, if the explicit argument is zero, return with the same absence of body-level calls and application-memory accesses.

For a clear flag and nonzero argument, call F-aabc9d8e982e without explicit arguments. If its result is zero, return. No later dependency is called, no context is captured, and no accounting write occurs.

This early-return description does not constrain effects performed inside F-aabc9d8e982e.

### Preliminary dependency and context capture

For nonzero P:

1. Call F-2fd8826557a4 without explicit arguments.
2. After that call returns, read the global context reference and capture C.

The context is therefore selected from the state existing after F-2fd8826557a4. Changes that dependency makes to the global reference can affect which context is used.

Subsequent explicit context-relative accesses and context arguments use captured C. A later change to the global reference alone does not redirect them. No conclusion follows about whether later dependencies preserve the memory addressed by C.

### Header validation

Read the 32-bit header at P − 8.

- If this observation is zero, invoke F-a817d8d58c10.
- Otherwise, read the header again. If bit 0 of this second observation is set, invoke F-a817d8d58c10.
- Otherwise, continue.

The supplied representation marks F-a817d8d58c10 as non-returning. Under that designation, either validation failure ends this function’s ordinary path before any accounting write or call to F-c0fc6d033caa. The actual failure mechanism is unknown.

The two validation conditions do not necessarily inspect one stable value. The second read is conditional on the first being nonzero. No intervening dependency call appears, but the body supplies no stabilization or concurrency guarantee.

In particular, the second observation is checked only for bit 0. If the first observation is nonzero and the second is zero, the represented validation permits continuation.

Earlier dependencies and context capture have already occurred before either validation failure.

### First accounting group

After validation succeeds, call F-c0fc6d033caa with C.

After it returns, perform the following ordered work:

1. Read the unsigned 16-bit first index at P − 4 and form its destination at C + 0x224 + 4 × index.
2. Read the header at P − 8 and capture its value masked by **0x3ffffffc**.
3. Read the first indexed destination, subtract the captured quantity, and write the result.
4. Read C + 0x30c, subtract the same captured quantity, and write the result.
5. Read the unsigned 16-bit second index at P − 2.
6. Read C + 0xe0 + 4 × that second index, subtract the same captured quantity, and write the result.
7. Read C + 0x220, subtract the same captured quantity, and write the result.

The mask retains bits 2 through 29 and clears bits 0, 1, 30, and 31. Its result ranges from zero to **0x3ffffffc**, inclusive, in multiples of four.

The quantity is captured once for these four subtractions. Changes to the header after capture do not change that captured quantity.

There is no header revalidation after F-c0fc6d033caa. Its effects can therefore alter the header, indices, destination values, or memory validity before accounting begins.

The second metadata index is read after the first two accounting writes. If those writes overlap its storage, its later read uses the resulting memory state. The function does not preserve an earlier snapshot of both indices.

Each metadata index ranges from 0 through 65,535. Scaling one by four produces at most 262,140, which fits in 32 bits. This fact does not prove that adding C and the field offset produces a valid address.

### Count updates and final subtraction

After the first four accounting writes, call F-09dcb55e8e0a without explicit arguments.

Use its result as the index for C + 0x310 + 4 × index. Then:

1. Read that indexed 32-bit count, decrement it by one, and write it.
2. Read the 32-bit count at C + 0x390, decrement it by one, and write it.
3. Read the header again, apply mask 0x3ffffffc, and subtract that quantity from the 32-bit destination at C + 0x08.

The final subtraction uses a fresh header observation, not necessarily the quantity captured for the first accounting group. F-09dcb55e8e0a, earlier writes, or another memory-changing actor may have changed the header.

The representation treats the dependency-provided index as signed 32-bit. There is no negative-value rejection or upper-bound check. Unlike the unsigned 16-bit indices, its scaling and address calculation are not established to remain within the relevant arithmetic range or storage extent.

The final subtraction follows both decrement writes and precedes both final dependency calls. The representation does not establish a relative evaluation order between the header read and destination read within that subtraction.

### Final dependencies

After all seven accounting writes:

1. Call F-cc6c9b4a0f68 with C and P − 8.
2. After it returns, call F-e586bff64fb7 with C.
3. Return.

Neither final dependency is established to preserve the updated accounting values. Consequently, the values written by this body are not unconditional post-return state guarantees.

If any dependency does not return normally, subsequent steps are not established to occur. No explicit rollback or recovery path is present.

## Side effects

On a complete normal path, the body performs seven ordered 32-bit read-modify-write operations:

- Four subtractions using one captured masked quantity.
- Two decrements.
- One subtraction using a later masked header observation.

Seven operations do not imply seven distinct addresses. When destinations coincide, later operations read the state produced by earlier operations, subject to any intervening dependency effects. No overlap detection or deduplication occurs.

The five masked subtractions use unsigned 32-bit arithmetic in the supplied representation. Results are stored modulo 2³². No underflow rejection, saturation, or zero clamp is present.

The two decrements are represented as signed 32-bit arithmetic. In-range subtraction by one is established. Behavior at signed overflow boundaries is **documented_only**; portable wraparound at the signed minimum is not asserted.

The body does not directly assign to the metadata fields as metadata. Accounting writes may nevertheless modify them through aliasing. Dependencies may also modify them.

No atomic operation, memory barrier, or explicit synchronization primitive is established by the body. The dependency ordering might serve a synchronization protocol, but that is inferred intent only.

Ownership consequences remain unknown:

- P is not proven invalid after the operation.
- P − 8 is not proven to be freed.
- Passing P − 8 to F-cc6c9b4a0f68 does not establish ownership transfer.
- No dependency is assumed to retain, release, or preserve the object or context.

## Dependencies

| Opaque ID | Established invocation |
|---|---|
| F-aabc9d8e982e | Called after both entry gates permit processing. Its result supplies P; zero ends processing. |
| F-2fd8826557a4 | Called for nonzero P, before capturing C. No explicit argument. |
| F-a817d8d58c10 | Invoked on either header-validation failure. Marked non-returning in the supplied representation. |
| F-c0fc6d033caa | Called with C after validation, before reading the metadata indices and performing accounting writes. |
| F-09dcb55e8e0a | Called after four accounting writes. Its result selects the indexed decrement destination. |
| F-cc6c9b4a0f68 | Called with C and P − 8 after all seven accounting writes. |
| F-e586bff64fb7 | Called with C after F-cc6c9b4a0f68 returns. |

Except for the object address and accounting index, dependency results are not used by this body. Unused results do not imply absent side effects.

No dependency implementation or preservation contract is supplied. Normal return alone does not establish preservation of globals, accounting fields, metadata, accessibility, or ownership.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. All examples requiring memory access assume valid storage and defined arithmetic for the path described.

### Suppressed entry

With the incoming zero-condition flag set, a nonzero explicit argument still causes immediate return without any listed dependency call.

With the flag clear and the explicit argument zero, the same call suppression applies.

### No object

With the flag clear, a nonzero argument, and F-aabc9d8e982e returning zero, only F-aabc9d8e982e is called. The body does not capture C or perform accounting writes.

### Validation failures and changing observations

For nonzero P, F-2fd8826557a4 and context capture precede validation.

- A first header observation of zero invokes F-a817d8d58c10 without a second validation read.
- A nonzero first observation followed by a second observation with bit 0 set invokes that same dependency.
- A nonzero first observation followed by a zero second observation passes the represented validation.

Under the designated non-returning interpretation, neither failure case reaches F-c0fc6d033caa or accounting writes.

### Stable metadata and distinct destinations

Assume:

- Both metadata indices remain zero.
- The header remains **0x00000042**.
- F-09dcb55e8e0a returns 3.
- The seven destinations below are mutually nonoverlapping and do not overlap metadata.
- Each destination is initialized to 100.
- No dependency or other actor changes any destination between that initialization and its corresponding body update.
- Dependencies return normally and required storage remains accessible.

The masked quantity is 64. The immediate results of the corresponding writes are:

| Destination | Value immediately after its body write |
|---|---:|
| C + 0x224 | 36 |
| C + 0x30c | 36 |
| C + 0xe0 | 36 |
| C + 0x220 | 36 |
| C + 0x31c | 99 |
| C + 0x390 | 99 |
| C + 0x08 | 36 |

These are individual write-time guarantees under the stated assumptions, not unconditional return-time guarantees. For all seven values to remain as listed after function return, no dependency or other actor may modify them after their corresponding writes through return. This includes preservation by F-09dcb55e8e0a of earlier results and by both final dependencies of all results.

### Zero masked quantity

A stable header of **0x00000002** passes validation and masks to zero. Assuming destinations and metadata do not overlap, each masked subtraction writes the same 32-bit value it reads immediately before that subtraction.

This does not guarantee equality with an earlier initialization: dependencies may have changed the destination before its read. The two count decrements still occur.

### Changed final quantity

Suppose the first captured masked quantity is 64, and F-09dcb55e8e0a changes the header so that the final header observation masks to 128. Assume no later header change and no invalidating alias effects.

Each of the first four subtraction writes stores its own just-read destination value minus 64 modulo 2³². The final subtraction stores its own just-read destination value minus 128 modulo 2³².

No claim is made about net changes from initialization unless dependencies and other actors also preserve those destinations until their respective updates.

### Unsigned underflow

If a masked subtraction reads destination value 16 and uses quantity 64, that operation writes **0xffffffd0**. This is its immediate stored result; subsequent dependencies or writes are not assumed to preserve it.

### Coincident count destinations

If F-09dcb55e8e0a returns 32, its indexed destination is C + 0x390, the same address as the following fixed decrement.

If that location contains 100 immediately before the first decrement, and no other actor changes it between the two operations, the first write stores 99 and the second stores 98. There is no deduplication.

## Uncertainties

The following remain **documented_only**:

- Whether the incoming zero-condition flag is a genuine interface requirement.
- Complete machine ABI, pointer interpretation, byte order, and alignment requirements.
- Context and object types, extents, lifetimes, and ownership rules.
- Valid ranges and semantic meanings of all three indices.
- Which overlapping layouts are permitted.
- Dependency effects on globals, metadata, accounting state, memory accessibility, and lifetime.
- Whether any dependency implements locking, unlocking, reclamation, or ownership transfer.
- The actual mechanism and independently verified non-returning behavior of F-a817d8d58c10.
- Behavior for inaccessible addresses, invalid address calculations, or signed arithmetic overflow.
- Concurrency guarantees and whether changing memory observations are supported by the execution environment.

Allocation-release bookkeeping and synchronization bracketing are inferred intent, not acceptance requirements.

## Evidence

The supplied static representation supports the entry gates, short-circuit validation, dependency-call order, context-capture timing, access offsets and widths, accounting mask, seven-write sequence, and final dependency arguments.

Its integer representations support distinguishing unsigned 16-bit metadata indices, unsigned 32-bit masked subtraction, and signed 32-bit count arithmetic. They do not establish a complete ABI or all boundary behavior.

No dependency implementation, independent layout contract, preservation guarantee, or runtime trace was supplied. Comments and apparent roles do not establish memory ownership or synchronization. No runtime tests have been performed.
