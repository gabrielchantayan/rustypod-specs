<!-- Trusted provenance: F-2bce23af7fb1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2bce23af7fb1

## Purpose

F-2bce23af7fb1 updates a caller-supplied state record through one of three mutually exclusive paths, in priority order:

1. If the linked-chain head is nonzero, replace it with the first word of the addressed object and return.
2. Otherwise, if the storage cursor differs from the storage end, advance the cursor by 12 bytes.
3. Otherwise, obtain two addresses through F-871e64ddc6de, initialize a descriptor, publish new storage bounds, and advance the new cursor by 12 bytes.

There is no returned value. Observable results consist of memory accesses and updates, plus two ordered dependency calls on the third path.

**Inferred intent:** The record resembles a pool of 12-byte entries, with a reusable-entry chain and a separate chain of storage descriptors. This interpretation does not establish ownership, entry initialization, or how callers obtain an entry address.

## Interface

### Inputs and widths

The routine receives:

- An address identifying a mutable state record.
- A represented 32-bit integer mode selector. Only whether it equals zero affects behavior.

The supplied representation uses 32-bit integer words and addresses represented through those words. This establishes the layout described below, not a portable ABI for other address widths.

The state record has four fields:

| Byte offset | Width | Meaning established by use |
|---|---:|---|
| 0 | 32 bits | Descriptor-head address, or zero |
| 4 | 32 bits | Linked-chain-head address, or zero |
| 8 | 32 bits | Storage cursor |
| 12 | 32 bits | Storage end address |

A newly initialized descriptor has three fields:

| Byte offset | Width | Meaning established by use |
|---|---:|---|
| 0 | 32 bits | Link copied from the state’s descriptor head |
| 4 | 32 bits | Selected count, subsequently interpreted as unsigned |
| 8 | 32 bits | Storage address obtained from the second dependency call |

The 12-byte entry size is established by the cursor advance and storage-request calculation. No internal entry layout is established, except that an object reached through the linked-chain head must support reading its first 32-bit word.

### Output and preconditions

The routine returns no address, status, or success indicator.

Each executed access requires accessible memory and valid address interpretation for that access. In particular:

- The state must support the fields reached on the selected path.
- A dereferenced linked-chain head must support a 32-bit read.
- A descriptor consulted for growth must support a 32-bit count read at offset 4.
- The first dependency result must support three descriptor-word writes and a subsequent storage-address read.
- Integer-to-address conversions, address calculations, and signed arithmetic must be valid for the execution environment.

No explicit checks enforce these requirements.

The routine establishes no synchronization, reentrancy protection, or general aliasing safety. Statements about one logical field changing while others remain unchanged require distinct memory and no intervening external mutation.

## Behavior

### Linked-chain priority

The first decision reads the linked-chain-head field and tests it against zero.

If the test observes a nonzero value, the routine reads the field again. That second value supplies the address from which the first 32-bit word is read. The resulting word replaces the linked-chain-head field, and the routine returns.

Thus, the address used for dereferencing is not necessarily the value observed by the initial test. With a stable head, this removes the current head from the represented chain. If the head changes between observations, the second observation determines the dereferenced address.

On this path:

- Mode has no effect.
- Cursor and end are not compared.
- No descriptor count is read.
- F-871e64ddc6de is not called.
- The cursor is not advanced.
- No explicit release or clearing of the former head occurs.
- The former head’s address is not returned.

A nonzero linked-chain head takes priority even when cursor equals end.

### Storage equality decision

Only when the linked-chain test observes zero does the routine compare cursor and end.

If they differ, it skips count selection and both dependency calls, proceeding to the final cursor advance.

The comparison is exact equality, not an available-capacity check. A cursor beyond the end, or fewer than 12 bytes before the end, still takes the existing-storage path when cursor and end differ.

If cursor and end are equal, the routine selects a count and obtains new storage. The representation does not establish a relative evaluation order between the cursor and end reads.

### Count selection

Count selection and growth calculations use unsigned 32-bit arithmetic.

For every nonzero mode, the selected count is 1. No previous descriptor count is inspected.

For zero mode, the routine reads the descriptor-head field and tests it against zero:

- If that observation is zero, the selected count is 32.
- Otherwise, the routine reads the descriptor-head field again and reads the unsigned count at offset 4 from the address supplied by that second observation.

The checked head is therefore not retained as the count-bearing address. A nonzero first observation does not guarantee that the second observation is nonzero or usable.

Let \(n\) be the previous count actually loaded. Define:

\[
A=(n+32)\bmod 2^{32}
\]

\[
P=\left(n+\left\lfloor n/2\right\rfloor+
\left\lfloor n/8\right\rfloor\right)\bmod 2^{32}.
\]

The selected count is the larger of \(A\) and \(P\) under unsigned comparison. Equality produces their common value. The divisions correspond to logical unsigned shifts.

This is a wrapped integer calculation, not an unbounded growth guarantee. The selected count can be smaller than the previous count.

**The selected count cannot be zero.** The fixed-count branches produce 1 or 32. In the growth branch, a zero maximum would require both candidates to be zero. The additive candidate is zero only for \(n=4,294,967,264\), where the proportional candidate is 2,684,354,508.

### Dependency calls and request sizes

After count selection, the routine calls F-871e64ddc6de twice, in this order:

| Call | First argument | Second argument |
|---|---:|---:|
| First | 12 | 0 |
| Second | Selected count multiplied by 12, modulo \(2^{32}\) | 0 |

The first result is used as a descriptor address. The second result is initially used as the descriptor’s storage-address value.

The second call precedes the routine’s first dereference of the first result. There is no explicit success check between the calls. If the first call returns an unusable address but returns normally, the routine still attempts the second call before accessing the descriptor.

Request-size multiplication is separate from count selection. A nonzero count does not, by itself, establish a nonzero or unwrapped product. For unsigned 32-bit multiplication by 12, a zero product requires a count divisible by \(2^{30}\). This arithmetic condition does not establish that such a count is reachable through this selection rule.

There is no explicit multiplication-overflow check.

### Descriptor initialization and publication

After both calls return normally, the routine performs these memory effects in order:

1. Write the second result into the descriptor’s storage-address field.
2. Read the state’s descriptor-head field and write the observed value into the descriptor’s link field.
3. Write the selected count into the descriptor’s count field.
4. Write the descriptor address into the state’s descriptor-head field.
5. Read the descriptor’s storage-address field back.
6. Write that read-back address into the state’s cursor field.
7. Write the read-back address plus the wrapped request size into the state’s end field.

The descriptor link captures a head observed **after both dependency calls**. It need not match either observation used during count selection. A dependency-induced head change does not cause the selected count to be recalculated.

The storage base used for publication comes from descriptor memory after the preceding writes, not directly from a retained copy of the second call’s result. Aliasing can therefore affect the published base.

The end-address addition uses unsigned 32-bit arithmetic because the request-size operand is unsigned. Its result is reduced modulo \(2^{32}\). The resulting value is stored through a signed integer field; portable interpretation of out-of-range unsigned-to-signed conversion is not established.

The selected count is also written through signed integer storage and later read as unsigned. The representation establishes this use, but does not independently establish portable conversion behavior for all values outside the signed range.

### Final cursor advance

After either retaining existing storage or publishing new storage, the routine reads the cursor, adds 12, writes the result back, and returns.

This addition is represented as signed integer arithmetic. The supplied material does not establish a portable result for signed overflow. It must not be assigned the unconditional wrapping guarantee of the unsigned count and request-size calculations.

Under stable, nonaliasing conditions with valid arithmetic, installing a block with count \(c\) and base \(B\) leaves:

- Cursor at \(B+12\).
- End at \(B+12c\), subject to the unsigned calculation and storage-conversion qualifications above.

The cursor advance does not initialize or otherwise write the entry bytes passed over.

## Side effects

### Direct memory effects

| Selected path | Direct writes |
|---|---|
| Nonzero linked-chain head | Linked-chain-head field |
| Zero linked-chain head, unequal cursor/end | Cursor field |
| Zero linked-chain head, equal cursor/end | Three descriptor fields; descriptor-head, cursor, and end fields; final cursor update |

These describe field-level effects, not isolation guarantees when addresses overlap.

The new-storage path does not explicitly write the linked-chain-head field and does not recheck it after dependency calls. A dependency-induced change to that field is not explicitly undone.

### Ownership and lifetime

The routine retains addresses in state and descriptor memory, but ownership and release obligations are not established.

There is no explicit release of:

- The removed linked-chain head.
- A prior descriptor or its associated storage.
- Either dependency result if later execution fails.

This absence does not establish that F-871e64ddc6de lacks independent resource-management effects.

### Errors and partial progress

There is no explicit failure return, recovery branch, rollback, or transactional guarantee.

If execution stops partway through the new-storage path, preceding effects may remain:

- Both dependency calls precede descriptor initialization.
- Descriptor initialization precedes descriptor-head publication.
- Descriptor-head publication precedes storage-address read-back and bounds publication.
- Bounds publication precedes the final cursor advance.

Fault handling, exceptions, process termination, and dependency failure behavior remain unspecified. An unusable address is not converted into an explicit error result by this routine.

## Dependencies

### F-871e64ddc6de

**Established use:** Two ordered calls occur only after the linked-chain test observes zero and cursor compares equal to end. Both calls receive zero as their second argument. Their first arguments are 12 and the wrapped count-times-12 request size.

**Inferred intent:** The results appear to supply memory for a descriptor and an entry region.

The following remain **documented_only**:

- Exact argument meanings and result type beyond the represented uses.
- Failure representation and whether failure returns normally.
- Alignment and usable extent of returned memory.
- Zero-size request behavior.
- Ownership, release obligations, and resource lifetime.
- Global effects, caller-state mutation, callbacks, and reentrancy.

Neither call is assumed to preserve caller-accessible state, return distinct storage, or avoid releasing existing resources.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless explicitly varied, memory is accessible, stable, and nonaliasing; dependencies return usable distinct regions without changing the state; and arithmetic and conversions have the represented results without signed overflow.

### Linked head suppresses growth

Given a nonzero linked-chain head whose first word is zero, with cursor equal to end:

- The linked-chain head becomes zero.
- Neither dependency call occurs.
- Descriptor head, cursor, and end remain unchanged.
- Mode does not affect the outcome.
- No explicit release or clearing of the former head occurs.

This remains the outcome for nonzero mode. Equal cursor/end and nonzero mode do not override linked-chain priority.

### Existing storage advances

Given a zero linked-chain head, cursor 4096, and end 4120:

- Neither dependency call occurs.
- Cursor becomes 4108.
- End and descriptor head remain unchanged.

### Invalid-looking bounds are not rejected

Given a zero linked-chain head, cursor 4120, and end 4096:

- Neither dependency call occurs.
- Cursor becomes 4132.

Likewise, cursor 4096 and end 4100 produce cursor 4108 without obtaining storage. These examples establish the equality-only trigger, not safe subsequent use of those addresses.

### Initial zero-mode block

Given a zero linked-chain head, equal cursor/end, zero descriptor head, and zero mode:

- Requests are 12 and 384, each with second argument zero.
- The descriptor receives link zero, count 32, and the returned storage address.
- The state’s descriptor head becomes the new descriptor address.
- Cursor becomes storage base plus 12.
- End becomes storage base plus 384.

### Nonzero mode ignores the previous count

Given a **zero linked-chain head**, equal cursor/end, and any nonzero mode:

- Requests are 12 and 12, each with second argument zero.
- The selected count is 1, regardless of any existing descriptor count.
- No previous descriptor count is read.
- The new descriptor’s link captures the descriptor head read after the calls.
- Under the stated arithmetic assumptions, final cursor equals end.

The zero linked-chain-head prerequisite is necessary. With a nonzero head, the routine instead takes the early-return path, without either request or a cursor update.

### Ordinary zero-mode growth

Given a zero linked-chain head, equal cursor/end, zero mode, and a valid nonzero descriptor head supplying the stated previous count:

| Previous count | Additive candidate | Proportional candidate | Selected count | Storage request |
|---:|---:|---:|---:|---:|
| 32 | 64 | 52 | 64 | 768 |
| 64 | 96 | 104 | 104 | 1248 |

The descriptor request remains 12 in both cases.

### Wrapped growth and request size

Given the same growth prerequisites and a loaded previous count of 4,294,967,264:

- Additive candidate: 0.
- Proportional candidate after unsigned wrapping: 2,684,354,508.
- Selected count: 2,684,354,508.
- Storage request after multiplication modulo \(2^{32}\): 2,147,483,024.

This is not a zero-size request. The published end uses the read-back storage base plus the wrapped request size. The final cursor advance is a separate signed operation.

### Distinct descriptor-head observations

Given a zero linked-chain head, equal cursor/end, and zero mode, suppose the initial descriptor-head test observes one nonzero address but the count-address read observes another valid descriptor address.

Count selection uses the second descriptor’s count.

If a dependency subsequently changes the head again, the new descriptor’s link receives the later value read during initialization. The selected count remains unchanged.

These conditional observations do not establish that unsynchronized mutation is supported or safe.

## Uncertainties

The following remain **documented_only**:

- The dependency’s complete behavior, failure contract, and memory-management rules.
- The enclosing data structure and caller protocol.
- How callers identify the entry associated with a cursor advance.
- External invariants that might prevent overflow, wrapped requests, inconsistent bounds, or invalid addresses.
- Whether callbacks, concurrency, or aliasing occur in supported use.
- Target-specific pointer conversions, signed conversions, signed-overflow behavior, and memory-fault behavior.
- Whether a zero wrapped request size is reachable through the complete count-selection rule.

None of these unknowns supplies an implicit validation, ownership, preservation, synchronization, or recovery guarantee.

## Evidence

**Established static evidence:** The supplied representation establishes linked-head priority and early return; separate linked-head observations; exact cursor/end equality; mode-dependent unsigned count selection; separate descriptor-head observations during growth selection; two ordered dependency calls; descriptor initialization order; late head capture; storage-address read-back; bounds publication; and the final signed cursor increment.

**Arithmetic consequences:** The growth formulas, unsigned comparison, nonzero selected-count conclusion, and large-count example follow from the represented 32-bit operations. Request-size wrapping is distinct from count-selection wrapping.

**Inferred intent:** Pool, entry, block, and storage-acquisition terminology describes the apparent organization. It does not establish ownership or a broader API contract.

**Verification limit:** No runtime tests have been performed. All acceptance examples are conditional consequences of static evidence.
