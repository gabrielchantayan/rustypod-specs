<!-- Trusted provenance: F-fc900472ddea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fc900472ddea

## Purpose

F-fc900472ddea transforms a four-word state descriptor obtained through dependency calls, then returns the first 32-bit word of a final helper-populated result area.

The observable transformation combines:

- A signed, four-byte-scaled difference between two descriptor words.
- A displacement selected through F-d908bb83bef8 and F-d7be9cfb4000.
- Either a four-byte backward adjustment or a table-entry transition followed by recalculation of descriptor fields.

**Inferred intent:** the descriptor resembles a position within segmented storage, and the operation resembles moving that position backward by one four-byte element, crossing a segment boundary when necessary. This interpretation is not needed for the contract. The supplied evidence does not establish the descriptor’s public type, the meaning of its table entries, or the arithmetic semantics of F-d7be9cfb4000.

## Interface

### Input

The function takes one address-sized input represented as a 32-bit integer in the supplied evidence. It passes the address obtained by adding 16 bytes to that input to F-b956a7a6b43c.

The function itself does not directly inspect the input object. Its required extent, layout beyond that offset, and validity requirements depend on F-b956a7a6b43c.

### Output

The function returns one 32-bit word. That word is read from offset zero of a 16-byte automatic result area after a final invocation of F-b956a7a6b43c.

This is not established to be a success code, pointer, Boolean, or complete descriptor. Its interpretation depends on what F-b956a7a6b43c writes.

### Working areas and descriptor layout

The function provides three separate 16-byte automatic areas to dependency calls:

1. An initial preparation area.
2. An intermediate preparation area.
3. A final result area.

No initialization of those areas is visible before the calls receiving them.

The return value of the second preparation call is interpreted as an address of a descriptor with four consecutive 32-bit words:

| Byte offset | Contract name | Observed use |
|---|---|---|
| 0 | Current address | Participates in the initial difference; later replaced with the resulting address |
| 4 | Base address | Participates in the initial difference; may be replaced by a table-entry value |
| 8 | Limit address | Written on the nonzero-displacement path |
| 12 | Table-slot address | Adjusted on the nonzero-displacement path and used to locate a 32-bit table entry |

These names describe observed roles, not established source-level types or invariants. In particular, the function does not check that the current address lies between the base and limit addresses.

The descriptor address need not equal the intermediate preparation area. It is specifically the address returned by the second preparation call. Whether it aliases either automatic area, the input object, or another allocation is unknown.

### Width and validity requirements

The visible representation uses 32-bit words, four-byte word spacing, and signed 32-bit arithmetic for the initial offset, helper-derived displacement, and associated arithmetic.

For ordinary execution, the dependencies must return usable values, and every descriptor or table access actually reached must reference accessible memory. The function contains no explicit null, bounds, alignment, or arithmetic-range checks.

A portable interpretation additionally requires that relevant signed arithmetic remain representable and that the initial right shift have the intended signed behavior. Overflow and negative-shift portability are addressed under **Uncertainties**.

## Behavior

### Preparation and initial offset

F-b956a7a6b43c is first called with the initial preparation area as its first argument and the input address plus 16 bytes as its second argument. The return value of this call is not used.

F-b956a7a6b43c is then called with the intermediate preparation area as its first argument and the initial preparation area as its second argument. The returned address identifies the working descriptor.

The function reads the descriptor’s current and base words and forms an initial signed element offset:

\[
N = \bigl((\text{current address} - \text{base address}) \mathbin{\text{signed right shift}} 2\bigr) - 1
\]

The subtraction precedes the shift, and the subtraction of one follows the shift. This computed value is retained for later calculations; it is not recomputed after dependency calls.

For a nonnegative, four-byte-aligned difference, this is the number of four-byte elements between the two addresses, minus one. A negative or unaligned difference is not rejected. Its result follows the exposed signed-shift operation, subject to the target-semantic uncertainty described below.

The relative order of the two descriptor reads within the initial subtraction is not established by the supplied expression. Both occur before the following dependency calls.

### Displacement selection

The sign of the initial offset determines the dependency-call sequence.

**When the initial offset is nonnegative:**

- F-d908bb83bef8 is called once.
- F-d7be9cfb4000 is called with the initial offset as its first argument and the preceding dependency result as its second argument.
- The signed 32-bit result of F-d7be9cfb4000 becomes the displacement.

**When the initial offset is negative:**

- F-d908bb83bef8 is called twice, in sequence.
- The first result supplies the second argument to F-d7be9cfb4000.
- The second result is used to construct its first argument: the second result minus the initial offset, then minus one.
- The signed 32-bit result of F-d7be9cfb4000 is negated to obtain the displacement.

The two F-d908bb83bef8 results on the negative path are separate observations. They must not be treated as interchangeable or assumed equal.

The evidence establishes the argument construction and result use, but not what F-d7be9cfb4000 computes. In particular, its name or apparent calling pattern does not establish signed division, rounding behavior, or handling of a zero second argument.

### Zero-displacement update

If the displacement equals zero, the function reads the descriptor’s current-address word again, subtracts four, and writes the resulting value back to that word.

This read occurs after all displacement-selection dependency calls. It is therefore not necessarily the same value used in the initial offset calculation.

On this path, the function itself does not write the base, limit, or table-slot words. Dependencies may nevertheless have changed those words.

### Nonzero-displacement update

If the displacement is nonzero, the function performs a table transition and recalculates the descriptor.

The table-slot address is advanced by four times the signed displacement. The updated slot address is stored in the descriptor **before** the table entry at that address is read.

The 32-bit value read from that table entry is then stored as the descriptor’s base address.

Next, F-d908bb83bef8 is called. After it returns, the function reads the descriptor’s base address and stores the following value as the limit address:

\[
\text{base address} + 4 \times \text{dependency result}
\]

F-d908bb83bef8 is called again. After that call returns, the function reads the descriptor’s base address again and computes the new current address:

\[
\text{base address} + 4 \times
\bigl(N - \text{displacement} \times \text{dependency result}\bigr)
\]

The initial offset and displacement in this expression are the previously computed values. The base address is a later descriptor read, not a guaranteed reuse of the table-entry value.

Consequently:

- The two post-transition dependency results may differ.
- Either call may change descriptor memory or other state.
- The base used for the limit calculation may differ from the base used for the current-address calculation.
- The limit calculation is not guaranteed to use the same scale as displacement selection.

The function finally stores the computed current address at descriptor offset zero.

### Result production

After either update path, F-b956a7a6b43c is called with the final result area as its first argument and the working descriptor address as its second argument.

The return value of this call is ignored. The function then reads and returns the first 32-bit word of the final result area.

The supplied evidence does not establish that this call copies all four descriptor words, that its first argument is its return address, or that it leaves the descriptor unchanged.

## Side effects

### Direct memory effects

The function directly writes descriptor memory as follows:

- **Zero displacement:** current-address word only.
- **Nonzero displacement:** table-slot word, then base-address word, then limit-address word, then current-address word.

On the nonzero path, the table-entry read occurs after the table-slot write and before the base-address write. The first post-transition dependency call occurs after the base write; the second occurs after the limit write.

The function reads a table entry but does not directly write that entry. Aliasing can make physical effects overlap: for example, the updated slot may point into the descriptor itself. No disjointness condition is checked.

The three preparation/result calls may produce additional writes not visible in this function’s body.

### Dependency invocation counts

For an execution that reaches the normal return:

| Condition | F-b956a7a6b43c | F-d908bb83bef8 | F-d7be9cfb4000 |
|---|---:|---:|---:|
| Initial offset nonnegative; displacement zero | 3 | 1 | 1 |
| Initial offset negative; displacement zero | 3 | 2 | 1 |
| Initial offset nonnegative; displacement nonzero | 3 | 3 | 1 |
| Initial offset negative; displacement nonzero | 3 | 4 | 1 |

These counts do not imply that calls are pure, inexpensive, or interchangeable.

### Ownership and lifetime

The three 16-byte working areas have automatic lifetime within this invocation. No explicit allocation, deallocation, ownership transfer, or cleanup call is visible.

Passing an address to a dependency does not establish whether that dependency retains it, allocates other memory, releases memory, or transfers ownership. The returned descriptor’s ownership is also unknown.

If the descriptor aliases an automatic working area, its storage ceases to be usable after return. The function returns only one word, not an explicit descriptor pointer, although that word’s meaning remains dependency-defined.

### Errors and partial updates

There is no explicit error return or rollback.

A failure while reading the selected table entry can occur after the table-slot word has already changed. A failure in a later dependency can occur after additional descriptor writes. The evidence provides no transactional guarantee and no recovery path.

Whether dependencies signal errors by returning values, trapping, unwinding, or another mechanism is unknown.

## Dependencies

### F-b956a7a6b43c

Used three times for preparation and final result production.

Established interaction:

- Receives a 16-byte automatic area as its first argument on each call.
- Receives the input-plus-16 address, initial preparation area, and working descriptor address as its respective second arguments.
- Only the second call’s return value is used, as a descriptor address.
- The final call must produce a usable first result word for a defined return value.

Unknown: copying behavior, access extent, initialization guarantees, aliasing, ownership, error handling, and effects on global state.

### F-d908bb83bef8

Supplies arithmetic values during displacement selection and, on the nonzero path, during descriptor recalculation.

Its results are consumed as 32-bit arithmetic values. The contract does not assume constancy across calls or preservation of descriptor memory, globals, or table contents.

### F-d7be9cfb4000

Receives two values constructed as described under **Displacement selection** and supplies the signed displacement result, with negation on the negative-offset path.

Its numerical operation, exceptional inputs, side effects, and preservation guarantees are not established.

## Acceptance examples

These are conditional examples derived from the static behavior, not runtime observations. Each assumes valid memory, representable arithmetic, helper behavior as stated, and no unmentioned dependency mutations.

### Interior position with zero displacement

Suppose preparation yields a descriptor with current address `0x1010` and base address `0x1000`.

The initial offset is 3. If F-d908bb83bef8 returns 8, F-d7be9cfb4000 receives arguments 3 and 8.

If that call returns zero, the current-address word becomes `0x100C`. The function itself leaves the other three descriptor words unchanged.

If final result production places `0xAABBCCDD` in its first word, the function returns `0xAABBCCDD`; it does not necessarily return `0x100C`.

### Negative offset with a backward table transition

Suppose current and base are both `0x2000`, and the table-slot word is `0x3004`. The initial offset is −1.

If the first two F-d908bb83bef8 calls both return 8, F-d7be9cfb4000 receives 8 and 8. If it returns 1, the displacement is −1.

The table-slot word becomes `0x3000`. If that slot contains `0x1000`, the base becomes `0x1000`. If both subsequent F-d908bb83bef8 calls return 8, the limit becomes `0x1020` and the current address becomes `0x101C`.

### Positive table transition

Suppose the initial current and base addresses are `0x1028` and `0x1000`. The initial offset is 9.

If displacement selection produces 1, the table-slot address advances by four bytes. If the selected entry contains `0x2000` and both later dependency results are 8, the limit becomes `0x2020` and the current address becomes `0x2004`.

This demonstrates that the observed implementation is not restricted to negative table movement.

### Mutation before a zero-displacement write

Suppose the initial current address is `0x1010`, but a displacement-selection dependency changes it to `0x5000` and the resulting displacement is zero.

The final current-address write is `0x4FFC`, because the zero-displacement path rereads that word after the dependency calls.

### Different values on the negative path

With an initial offset of −1, suppose the first F-d908bb83bef8 result is 8 and the second is 12.

F-d7be9cfb4000 receives 12 as its first argument and 8 as its second. Replacing these observations with one cached value would not preserve the established interaction.

## Uncertainties

The following remain **documented_only**:

- The public input type and the meaning of its offset-16 region.
- Whether the descriptor is an iterator, cursor, or another structure.
- The exact behavior and ownership contracts of all three dependencies.
- Target behavior for signed overflow, out-of-range address arithmetic, and negation of the minimum signed 32-bit value.
- The precise machine semantics of right-shifting a negative signed difference. An arithmetic shift would round downward rather than toward zero.
- Alignment requirements and invalid-address behavior.
- Whether dependency calls can modify descriptor or table storage, retain automatic-area addresses, or alter global state.
- The meaning and initialization guarantees of the returned word.
- Concurrency and reentrancy guarantees. No explicit synchronization is visible.

No behavior is inferred from a conventional division or segmented-container implementation where the supplied evidence does not establish it.

## Evidence

The established contract comes solely from the supplied static function representation: its dependency calls, argument construction, four-word descriptor accesses, conditional writes, arithmetic, and final 32-bit result read.

The segmented-position interpretation is inferred intent, not established type information. Acceptance examples are conditional arithmetic consequences, not observed executions.

No runtime tests, dependency inspection, or independent implementation access have been performed.
