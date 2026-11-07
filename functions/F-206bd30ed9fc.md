<!-- Trusted provenance: F-206bd30ed9fc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-206bd30ed9fc

## Purpose

F-206bd30ed9fc conditionally acquires an object-associated resource and records an owner token and a 16-bit acquisition count. It also has a same-owner path that increments the count without invoking the acquisition dependency, and a same-owner rejection path.

This acquisition interpretation is **inferred intent**. The established behavior is a sequence of field reads, conditional dependency calls, two possible field writes, and status returns. The supplied material does not establish the larger resource protocol, the meaning of owner tokens, or a corresponding release operation.

The function returns zero on its explicit success paths. It returns fixed statuses for a null argument and two same-owner rejection conditions, or propagates a nonzero status from a dependency.

## Interface

### Argument and result

The function accepts one object pointer and returns a signed 32-bit status value under the representation shown in the supplied material.

- A null pointer is accepted as an error case and returns decimal **26**.
- A non-null pointer must support the reads and writes described below.
- The function does not validate the pointer’s allocation, extent, alignment, or lifetime.
- Dependency results used as statuses are returned unchanged when nonzero.

The visible representation uses 32-bit integer fields, 16-bit halfword fields, and byte offsets obtained through a 32-bit pointer-to-integer representation. Portability to different integer widths or pointer representations is not established.

### Observable object layout

Offsets are relative to the supplied object pointer.

| Byte offset | Width | Role in this function |
|---|---:|---|
| 0 | 32 bits | Header value compared with a global discriminator |
| 4 | 32 bits | Owner token; compared with a dependency result and conditionally replaced |
| 14 | 16 bits | Flags word from which a two-bit mode is extracted |
| 18 | 16 bits | Acquisition count; conditionally read and written |
| 20 | Unspecified | Address passed to F-6f7f27926f9e |

The mode is the value of bits 4–5 of the flags word, yielding a value from 0 through 3. Other flag bits do not affect this function’s branch selection.

The counter is read as an **unsigned 16-bit** value. Its update is stored through a **signed 16-bit** field representation. The relevant stored result is the low 16-bit pattern of the increment, subject to the representation caveat in **Uncertainties**.

The function itself does not access the intervening object bytes. That does not establish that dependencies leave those bytes unchanged.

The explicit accesses extend through byte 19. Passing the address at offset 20 to a dependency imposes additional, unknown storage requirements; a 20-byte object is not thereby sufficient.

### Preconditions

For defined operation on a non-null argument:

- Each explicitly accessed field must be readable, and each field reached by a write must be writable.
- The object must satisfy the target’s alignment requirements.
- The object and the region beginning at offset 20 must satisfy the contracts of any dependencies actually called.
- The global discriminator must be readable.

No allocation, reference transfer, ownership transfer, or lifetime extension is established by the visible function. Dependency ownership behavior remains unspecified.

## Behavior

### 1. Null handling

A null argument causes an immediate return of **26**.

This path performs no object access and calls none of the three dependencies.

### 2. Conditional preparation

For a non-null argument, the function compares the 32-bit header at offset 0 with a global discriminator.

If the values differ, preparation is skipped.

If they match, the function calls **F-621dae46a146** with:

1. The original object pointer.
2. An integer zero.

A nonzero result is returned immediately and unchanged. A zero result permits processing to continue.

Preparation precedes the mode snapshot, owner-token lookup, and owner comparison. Consequently, changes made by the preparation dependency can affect those later observations. Its successful return does not, by itself, establish that the object was unchanged.

The relative evaluation order of the header read and the global-discriminator read is not established. Both participate in the comparison before any preparation call.

### 3. Mode snapshot and owner-token lookup

After preparation has either been skipped or returned zero, the function:

1. Reads the 16-bit flags field at offset 14.
2. Extracts bits 4–5 and retains that mode value.
3. Calls **F-196812b4b122**, with no arguments, and retains its 32-bit result.
4. Reads the 32-bit owner field at offset 4 and compares it with the retained result.

The owner comparison therefore uses an owner-field read performed **after** the token dependency returns, but a mode value captured **before** that call.

If F-196812b4b122 modifies the flags, the later decision still uses the earlier mode snapshot. If it modifies the owner field, the comparison observes the owner field after that modification. No preservation guarantee for this dependency is assumed.

The token dependency is not treated as a status-returning operation here. Every returned bit pattern participates in the owner comparison and may later be written as the owner token.

### 4. Same-owner decisions

The following special paths apply only when the owner field equals the retained token.

#### Mode 2: increment the existing count

The function reads the unsigned 16-bit count at offset 18.

If the count is **65,535**, it returns **39** without writing the counter and without calling F-6f7f27926f9e.

For every other count, it writes the incremented 16-bit value to offset 18 and returns zero. This includes:

- Zero becoming one.
- 32,767 becoming the bit pattern for 32,768.
- 65,279 becoming 65,280.
- 65,534 becoming 65,535.

The displayed range check does **not** reject the whole range beginning at 65,280. Its two conditions together reject only 65,535: within that upper range, the difference from 65,280 must exceed 254.

This successful path does not explicitly write the owner field and does not call the acquisition dependency.

#### Mode 1: reject

The function returns **15** immediately.

It does not explicitly read or write the counter and does not call F-6f7f27926f9e.

#### Modes 0 and 3: continue to acquisition

An owner match alone does not bypass acquisition. For modes 0 and 3, processing continues through the common acquisition path.

### 5. Common acquisition path

This path is taken when either:

- The owner field does not equal the retained token, regardless of mode; or
- The owner field equals the retained token and the mode is 0 or 3.

The function calls **F-6f7f27926f9e** with the address at object offset 20.

If that call returns nonzero, the function returns its result unchanged. The function performs neither of its subsequent explicit field writes. Mutations already performed by dependencies are not undone.

If the call returns zero, the function performs these writes in order:

1. Writes the retained 32-bit token to the owner field at offset 4.
2. Writes the 16-bit value one to the counter field at offset 18.

It then returns zero.

The owner token is the result retained from the earlier F-196812b4b122 call. It is not refreshed after acquisition. The owner field, flags, and counter are not reread to choose a different path after F-6f7f27926f9e returns.

The prior counter value is irrelevant on this path: successful acquisition replaces it with one.

## Side effects

### Explicit writes

F-206bd30ed9fc has only two kinds of explicit object update:

- A counter-only increment on the matching-owner, mode-2 success path.
- An owner-token write followed by a counter reset to one on the common acquisition success path.

There are no explicit writes to the header, flags, global discriminator, or region beginning at offset 20.

The function contains no explicit allocation, deallocation, release operation, logging, or rollback.

### Dependency effects and failure state

Dependencies may have additional effects not specified by the supplied material. In particular:

- Preparation may change the object before its mode and owner are examined.
- Token lookup may change state before the owner is read.
- Acquisition may change the region passed to it, other object fields, globals, or external state.

A nonzero return does not imply that all state is unchanged. On a dependency failure, this function propagates the failure without restoring earlier state.

Even the fixed same-owner errors occur after any applicable preparation and after token lookup. Their lack of explicit field writes does not imply absence of dependency side effects.

### Ordering and concurrency

The ordering described above is the function’s visible control-flow ordering. No atomic read-modify-write, memory barrier, or concurrency guarantee is established.

The matching-owner mode-2 path bypasses the acquisition dependency. Safe concurrent use of that path requires guarantees outside the supplied function; none can be inferred solely from the counter update.

## Dependencies

| Opaque function ID | Invocation | Use of result |
|---|---|---|
| **F-621dae46a146** | Original object pointer and integer zero; called only when the header matches the global discriminator | Zero continues; nonzero is returned unchanged |
| **F-196812b4b122** | No arguments; called after mode extraction on every path that passes preparation | Result is retained as an owner token, not checked as a status |
| **F-6f7f27926f9e** | Pointer to object offset 20; called only on the common acquisition path | Zero enables owner/count writes; nonzero is returned unchanged |

The inferred roles are preparation, owner-token lookup, and resource acquisition, respectively. Their actual contracts, possible blocking behavior, storage requirements, and ownership effects are **documented_only** unknowns.

The global discriminator is an additional data dependency. Its value and lifecycle are not supplied.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not results of runtime tests. Field values refer to the point at which the function actually reads them; they need not equal entry values if a dependency has changed the object.

1. **Null argument.**  
   The result is 26. No object read or dependency call occurs.

2. **Preparation failure.**  
   The header matches the discriminator, and F-621dae46a146 returns a nonzero status. That exact status is returned. Mode extraction, token lookup, acquisition, and the function’s explicit field writes do not occur.

3. **Preparation skipped.**  
   The header differs from the discriminator. F-621dae46a146 is not called. Mode extraction and token lookup still occur.

4. **Matching owner, mode 2, ordinary increment.**  
   The retained mode is 2, the owner matches the returned token, and the count reads as seven. The function writes eight to the counter and returns zero. F-6f7f27926f9e is not called.

5. **Matching owner, mode 2, upper-range increment.**  
   Under the same owner and mode conditions, a count of 65,280 is accepted and becomes 65,281. A count of 65,534 is accepted and becomes 65,535.

6. **Matching owner, mode 2, maximum count.**  
   A count of 65,535 produces status 39. There is no explicit counter write and no acquisition call.

7. **Matching owner, mode 1.**  
   The result is 15, without an explicit counter read or acquisition call. The counter’s contents do not affect this decision.

8. **Different owner, mode 1 or 2.**  
   The special same-owner branches do not apply. The acquisition dependency is called with the address at offset 20.

9. **Matching owner, mode 0 or 3.**  
   Acquisition is still attempted. Owner equality does not make these modes immediate-success cases.

10. **Acquisition failure.**  
    F-6f7f27926f9e returns a nonzero status. The exact result is returned, with no subsequent explicit owner or counter write. Dependency effects remain possible.

11. **Acquisition success.**  
    F-6f7f27926f9e returns zero. The function writes the previously retained token to offset 4, then writes one to offset 18, and returns zero, regardless of the old count.

12. **Mutation during token lookup.**  
    If F-196812b4b122 changes the flags, branch selection still uses the mode read before that call. The owner comparison uses the owner field read after the call.

## Uncertainties

The following are **documented_only** rather than guessed:

- The object’s public type, full size, packing, and intended lifetime.
- Target byte order and exact alignment requirements.
- The discriminator’s value and semantic meaning.
- Whether the token identifies a thread, process, execution context, or something else.
- Whether any dependency blocks, allocates, releases storage, changes globals, or mutates the supplied object.
- Whether dependency failures leave their own operations partially completed.
- Whether concurrent or reentrant calls are supported.
- Whether external invariants constrain modes or count values. The function itself permits a matching-owner mode-2 count of zero to become one.
- Whether callers interpret the count as signed or unsigned. This function reads it unsigned but stores through a signed 16-bit representation.
- The original language-level guarantees for conversion of values above 32,767 into that signed representation. The visible target-level effect is a 16-bit store; portable source-language conversion behavior is not established.

Removed-unreachable-region notices provide no usable behavioral contract for those regions. They do not justify adding alternative paths or side effects.

## Evidence

**Established static evidence:** The supplied function body supports the offsets, widths, branch conditions, dependency arguments, call ordering, explicit write ordering, and return values specified above. The maximum-count rejection follows from evaluating the shown arithmetic over the unsigned 16-bit input domain.

**Inferred intent:** The combination of an owner comparison, same-owner counter increment, acquisition dependency, and count initialization suggests an owner-tracked acquisition protocol with mode-dependent repeat-acquisition behavior. This interpretation does not establish the dependencies’ semantics.

**Verification boundary:** No implementation access, dependency inspection, or runtime tests have been performed. The acceptance examples are static contract conditions, not observed executions.
