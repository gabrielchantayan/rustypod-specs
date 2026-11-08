<!-- Trusted provenance: F-2b0966dfc5bf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2b0966dfc5bf

## Purpose

F-2b0966dfc5bf conditionally issues a sequence of operations through an acquired handle, using a context, a descriptor, and a signed selector.

The supplied static representation establishes the call sequence, argument values, conditional branches, and several memory accesses. It does not establish the execution semantics of the operations being issued. Their apparent association with indexed data, a uniqueness check, and two adjacent numbered resources is **inferred intent**, not a complete semantic guarantee.

The function has no returned result. It stops without issuing the main sequence when either an initial dependency returns nonzero or handle acquisition returns zero. Otherwise, it performs setup, optional selector handling, an optional uniqueness-related sequence, patching, and two terminal dependency calls.

## Interface

### Inputs

The visible interface has three inputs:

1. **Context:** an address referencing a structure accessed as 32-bit words.
2. **Descriptor:** an address referencing another structure accessed as 32-bit words, except for one single-byte flag.
3. **Selector:** a signed 32-bit integer. Negative and nonnegative values select different behavior; the visible comparison is not restricted to a particular negative sentinel.

The representation uses 32-bit integers and 32-bit address-bearing words. The offsets below describe the supplied layout, not a portable native-pointer layout.

### Context layout accessed directly

| Byte offset | Width | Meaning established by use |
|---|---:|---|
| 0 | 32 bits | Address-bearing root value, captured before the initial resolver call |
| 68 | 32 bits | Number saved at entry and supplied to later dependencies |

The function also forms the saved number plus one. This specification calls the saved number **base** and the derived number **adjacent**. These names describe their arithmetic relationship, not a proven resource type.

### Descriptor layout accessed directly

| Byte offset | Width | Meaning established by use |
|---|---:|---|
| 0 | 32 bits | Argument to the initial gating dependency |
| 4 | 32 bits | Quantity read only within the optional flagged sequence |
| 16 | 32 bits | Address-bearing word identifying a nested record |
| 20 | 32 bits | Replacement selector, read only when the input selector is negative |
| 24 | 8 bits | Flag; zero disables the optional sequence, any nonzero byte enables it |
| 36 | 32 bits | Argument to the resolver |

Only the first byte at descriptor offset 24 is tested. The other three bytes of that word do not participate in the visible condition.

### Other layouts accessed directly

The nested record captured from descriptor offset 16 supplies:

- A 32-bit value at offset 20.
- A 32-bit value at offset 0.

Both are passed to F-a836e5479ebf.

The captured root supplies a 32-bit table address at offset 8. The resolver result selects an entry with a stride of 24 bytes, and the first 32-bit word of that entry is passed to the initial gate.

The acquired handle supplies a 32-bit value at offset 12. That field is read separately for each later patching call that uses it.

### Output and preconditions

There is no direct output value. Observable results are dependency calls, effects caused by those dependencies, and any faults caused by invalid accesses.

No visible validation establishes pointer validity, descriptor size, table bounds, nested-record validity, handle structure validity, or acceptable numeric ranges. Defined execution requires the accessed storage to be valid at the time of access and dependency-specific preconditions to hold.

No explicit allocation, deallocation, or ownership transfer appears in this function. Passing an address to a dependency does not establish whether that dependency retains, modifies, or releases it.

## Behavior

### 1. Capture entry values and resolve an entry

Before the resolver call, the function captures:

1. The context word at offset 68 as base.
2. The nested-record address from descriptor offset 16.
3. The root value from context offset 0.
4. Adjacent, calculated as base plus one.

It calls F-49f750c2c596 with the captured root and the descriptor word at offset 36. Call its result the **entry number**.

Using the captured root, it reads the table address at root offset 8, locates the entry at entry number multiplied by 24 bytes, and reads that entry’s first 32-bit word.

It then calls F-0055bebe0706 with these ordered arguments:

- Context.
- `0x1b`.
- Descriptor word at offset 0.
- Zero.
- The selected table word.

The captured root remains the basis for the table access even if F-49f750c2c596 changes context offset 0. Conversely, the table address and selected table word are accessed after that dependency returns. The descriptor word at offset 0 is also obtained for the later gate call, rather than being established as an entry-time snapshot.

If F-0055bebe0706 returns nonzero, the function returns immediately. It does not call the configuration or handle-acquisition dependencies.

### 2. Configure and acquire a handle

When the gate returns zero, the function calls F-a836e5479ebf with:

- Context.
- Entry number.
- Nested-record word at offset 20.
- One.
- Nested-record word at offset 0.

The nested-record address is the address captured before resolution. Its two values are read for this call after the gate has returned; the visible representation does not establish an ordering between those two argument reads.

Next, it calls F-9b3ea8470521 with the context.

If the result is zero, the function returns. The preceding configuration call has already occurred; no explicit rollback follows.

A nonzero result becomes the handle used throughout the remaining sequence. The function does not subsequently reload a replacement handle from the context.

### 3. Choose the effective selector

For a negative input selector:

1. Read descriptor offset 20 as the **effective selector**.
2. Call F-2ea344827b50 with handle, `0x5e`, effective selector, entry number, and zero.

For a nonnegative input selector:

- Use the input selector itself as the effective selector.
- Do not issue the `0x5e` call.

Then call F-71980e76ca3a with context and descriptor. Pass its result to F-0b0cd8d831b7 with:

- Handle.
- `0x09`.
- Adjacent.
- Effective selector.
- Entry number.
- The result from F-71980e76ca3a.
- The 32-bit pattern `0xfffffff7`.

If the original input selector was nonnegative, next call F-d2aa1dc1574b with handle and one. This condition depends on the original input, not the effective selector loaded from the descriptor.

### 4. Prepare subsequent operations

Call F-d0e9fafa98a9 with:

- Context.
- Base.
- Entry number.
- Captured nested-record address.
- `0x0d`.

Then call F-887ba2404a49 with handle, `0x74`, base, and zero. Save its result as the **anchor**.

Next:

1. Call F-e4b8d56f2a5d with context, saving its result as the **resource value**.
2. Call F-60262630b470 with context, descriptor, base, resource value, and one.
3. Save the latter result as the **derived value**.

These results are used without visible success checks or range validation.

### 5. Perform the optional flagged sequence

After the preceding dependencies return, read the byte at descriptor offset 24.

If it is zero, skip this entire section.

If it is nonzero:

1. Read the 32-bit quantity at descriptor offset 4.
2. Call F-2ea344827b50 with handle, `0x41`, derived value, zero, and that quantity. Save the result as the first patch reference.
3. Call F-0b0cd8d831b7 with handle, `0x22`, adjacent, zero, the sum of that quantity and the derived value, resource value, and `0xfffffff2`. Save the result as the second patch reference.
4. Call F-0b0cd8d831b7 with handle, `0x26`, `0x13`, two, zero, a diagnostic-data address, and `0xfffffffe`.
5. Read the 32-bit field at handle offset 12 and call F-a2b3e96577b2 with handle, first patch reference, and that field value.
6. Read handle offset 12 again and call F-a2b3e96577b2 with handle, second patch reference, and the newly read value.

The two patch targets are not a single cached snapshot. Effects of the first patching call can affect the field value supplied to the second.

The diagnostic-data label suggests a nonunique-indexed-columns message. Its bytes, encoding, storage duration, and interpretation by the dependency are not shown.

### 6. Issue the common tail

Whether or not the flagged sequence runs, the function next performs:

1. F-2ea344827b50 with handle, `0x68`, adjacent, resource value, and zero.
2. F-65af7db35d2d with context and resource value.
3. F-887ba2404a49 with handle, `0x67`, base, and anchor plus one.
4. A fresh read of handle offset 12, followed by F-a2b3e96577b2 with handle, anchor, and that field value.
5. F-211638bb75da with handle, `0x20`, and base.
6. F-211638bb75da with handle, `0x20`, and adjacent.

The return value of the `0x67` call is not used. The patch call uses the anchor returned by the earlier `0x74` call.

The function then returns without a result.

## Side effects

The visible body contains no direct stores through the context, descriptor, nested record, root, or handle. This does **not** establish that those objects remain unchanged: dependencies receive addresses and may modify them or other shared state.

The principal observable ordering is the dependency sequence described above. In particular:

- Configuration precedes handle acquisition.
- Negative-selector setup precedes F-71980e76ca3a.
- The nonnegative-selector call follows the `0x09` call.
- Preparation precedes creation of the anchor and resource value.
- The optional diagnostic-bearing call precedes both optional patching calls.
- F-65af7db35d2d precedes the `0x67` call.
- The two final `0x20` calls occur in base-then-adjacent order.

The visible representation does not establish the evaluation order of independent argument reads within one call. It does establish reads separated by completed calls, including the repeated reads of handle offset 12.

There is no explicit error return, rollback, exception handling, or compensating cleanup. Nonzero gating and zero handle acquisition are the only visible return-value-controlled early exits. Other dependency results are either consumed as values or ignored.

The role of F-65af7db35d2d is not proven to be memory release. Likewise, the final `0x20` calls are not proven to destroy resources.

## Dependencies

The following roles describe only how F-2b0966dfc5bf uses each dependency:

| Opaque ID | Caller-visible role |
|---|---|
| F-49f750c2c596 | Produces the entry number from root and descriptor data |
| F-0055bebe0706 | Gates all subsequent work; continuation requires zero |
| F-a836e5479ebf | Receives entry number and nested-record values before acquisition |
| F-9b3ea8470521 | Produces the handle; zero stops the main sequence |
| F-71980e76ca3a | Produces an argument for the `0x09` call |
| F-2ea344827b50 | Receives operations tagged `0x5e`, `0x41`, and `0x68` |
| F-0b0cd8d831b7 | Receives operations tagged `0x09`, `0x22`, and `0x26` |
| F-d2aa1dc1574b | Called only for a nonnegative original selector |
| F-d0e9fafa98a9 | Receives base, entry number, nested record, and `0x0d` |
| F-887ba2404a49 | Receives `0x74` and `0x67`; the first result becomes the anchor |
| F-e4b8d56f2a5d | Produces the resource value |
| F-60262630b470 | Produces the derived value |
| F-a2b3e96577b2 | Receives patch references and freshly read handle fields |
| F-65af7db35d2d | Receives the resource value during the common tail |
| F-211638bb75da | Receives the two final `0x20` operations |

None of these usage descriptions guarantees that a dependency preserves globals, leaves passed objects unchanged, succeeds, returns normally, or follows a particular ownership policy.

## Acceptance examples

These examples are conditional static expectations, not executed tests.

1. **Gate rejects continuation.** If F-0055bebe0706 returns any nonzero value, resolution and the gate call occur, but configuration, acquisition, and all later calls do not.

2. **Handle acquisition returns zero.** If the gate returns zero and F-9b3ea8470521 returns zero, configuration has occurred. No selector-dependent calls or common-tail calls occur.

3. **Negative selector.** With input selector −2 and descriptor offset 20 containing 7, the `0x5e` call receives 7, and the subsequent `0x09` call also receives 7 as its effective-selector argument. F-d2aa1dc1574b is not called.

4. **Zero selector.** Zero takes the nonnegative branch. No `0x5e` call occurs; the `0x09` call receives zero; F-d2aa1dc1574b follows it.

5. **Flag is byte-specific.** If the byte at descriptor offset 24 is zero, nonzero bytes at offsets 25–27 do not enable the optional sequence. A value of `0x80` in the tested byte does enable it.

6. **Optional arithmetic.** If the descriptor quantity is 3 and the derived value is 10, with representable arithmetic, the `0x41` call receives derived value 10 and quantity 3, while the `0x22` call receives 13 in its sum argument.

7. **Fresh patch targets.** If handle offset 12 contains 40 before the first optional patch, and that patch changes it to 41, the second optional patch receives 41. It must not receive a cached 40.

8. **Saved base.** If base is initially 12 and a dependency later changes context offset 68, the explicit later arguments remain based on saved base 12 and adjacent 13. The context itself is still passed to dependencies, which may independently inspect its changed contents.

## Uncertainties

The following remain **documented_only**:

- Exact operation-tag semantics, including whether they construct instructions, execute actions, or manipulate another representation.
- Valid entry-number ranges, table bounds, and permitted negative values.
- Exact arithmetic behavior at overflow boundaries. Base plus one, anchor plus one, quantity plus derived value, and entry-address calculation require width-aware treatment. The representation does not independently establish whether overflow wraps, traps, or lies outside valid inputs.
- Signedness and semantic domains of opaque dependency results and descriptor words.
- The interpretation of `0xfffffff7`, `0xfffffff2`, and `0xfffffffe`. They are established 32-bit patterns; interpreting them as −9, −14, and −2 requires signed two’s-complement interpretation.
- Ownership, retention, mutation, allocation, and release behavior inside dependencies.
- The diagnostic bytes and the meaning of `0x13`, two, and the diagnostic-bearing operation.
- Concurrency requirements, fault behavior, and whether dependencies can terminate execution nonlocally.

## Evidence

Established behavior comes from static inspection of the supplied representation: field widths and offsets, explicit conditions, call arguments, saved values, and call ordering.

Resource, patching, and uniqueness terminology is descriptive or inferred where dependency semantics are unavailable. It does not expand the established guarantees.

No runtime tests have been performed. The acceptance examples state conditional consequences of the visible control flow, not observed execution results.
