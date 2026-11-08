<!-- Trusted provenance: F-4cc0ed70c43d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4cc0ed70c43d

## Purpose

F-4cc0ed70c43d conditionally processes a record whose discriminator has the low 16-bit value 4. The processing path obtains a provider, checks its status, acquires and tests a candidate, derives a count, and conditionally invokes an external three-argument operation. Two notification-like calls and another helper follow that operation.

“Provider,” “candidate,” “descriptor,” “transfer,” and “notification” describe roles inferred from argument flow. They do not establish application-domain semantics, memory-copy behavior, allocation, or resource ownership.

The established outcomes are:

- A nonmatching discriminator produces a normal return of the 32-bit value 4.
- A matching discriminator also returns 4 when the provider status is nonzero.
- A matching discriminator with a captured zero field at record offset 0 reaches a helper marked non-returning.
- A zero provider status leads to another helper marked non-returning, provided intervening operations return normally.

The normal-return value 4 is not established as a success or error code.

## Interface

### Inputs and record layout

The function accepts an opaque 32-bit argument and a record pointer. The first argument is forwarded unchanged to F-5ca71e377766 if that call is reached.

The representation exposes these record accesses:

| Byte offset | Represented width | Use |
|---|---:|---|
| 0 | 32 bits | Captured after two initial helpers; checked for zero and subsequently passed to dependencies. |
| 8 | 32 bits | Passed to F-c25a2fcc75b5. |
| 12 | 32-bit word narrowed to 16 bits | Compared with 4. |

No access to the record word at offset 4 is shown.

The discriminator comparison uses a signed 16-bit interpretation after narrowing. Only the low-halfword pattern `0x0004` matches; the upper 16 bits do not affect that decision. This describes the supplied representation, not a guarantee about the width of a physical machine load.

### Outputs and termination

Every established normal return yields the 32-bit value 4. There is no explicit output parameter.

F-a817d8d58c10 and F-feee9156a257 are marked non-returning. Their termination mechanisms are documented_only unknowns: the representation does not distinguish termination, exception propagation, abort, or another control transfer. Behavior if either unexpectedly returns is not established.

### Temporary storage layout and identity

Two distinct temporary storage regions are relevant:

- **Primary region:** passed to the first descriptor-producing call and later passed again to the final descriptor-producing call. These calls receive the **same storage address**.
- **Secondary region:** passed only to the conditional middle descriptor-producing call. Its address differs from the primary region’s address.

For each region, the representation associates a base address with a 32-bit slot at byte offset 4. The caller reads that offset-4 slot after the first descriptor production and, when taken, after the conditional middle production.

This establishes an address relationship, not a complete descriptor size. It does **not** establish that either descriptor occupies exactly two words, that either producer writes eight bytes, or that dependencies restrict their accesses to those locations.

No initialization or clearing of either region is shown before its descriptor-producing call. No clearing or reinitialization of the primary region is shown between its first and final uses.

After F-5ca71e377766 returns, the caller explicitly writes the selected 32-bit count at the secondary region’s base. That write occurs even when the conditional middle descriptor production was skipped.

### Preconditions

The record must support the discriminator access. A matching discriminator additionally requires the offset-8 and offset-0 reads.

Reached indirect calls require usable provider or candidate objects, dispatch-table access, and callable entries. The provider is not explicitly checked for null. A candidate is checked for null, but not otherwise validated.

Temporary-storage producers and consumers must be compatible with the actual storage supplied. Their required extent, alignment, initialization requirements, and access footprints are not established. In particular, the caller does not establish valid initial contents through explicit initialization before passing these addresses.

No source-level recovery for invalid pointers, malformed dispatch tables, or incompatible descriptor accesses is shown.

## Behavior

### Discriminator and initial acquisition

The discriminator is evaluated before any dependency call. A nonmatching low halfword returns 4 without consuming the record’s offset-0 or offset-8 fields.

For a matching discriminator, the function:

1. Reads the 32-bit field at record offset 8 and passes it to F-c25a2fcc75b5.
2. Retains that helper’s result.
3. Calls F-b7af80e829ec to obtain a provider pointer.
4. Reads and captures the 32-bit field at record offset 0.
5. Calls F-a817d8d58c10 if the captured value is zero.

The zero check therefore occurs **after** both acquisition helpers. They may produce effects before that check. If they modify the offset-0 field through shared state or aliases, the later capture can reflect those modifications.

Subsequent uses of the offset-0 value use the captured value; no later reread of that field is shown.

### Provider gate

For a nonzero captured value, the function invokes the provider’s indirect operation selected at dispatch-table byte offset `0x120`, supplying the provider as an explicit argument.

A nonzero status causes normal return of 4. No candidate acquisition or final non-returning candidate call occurs on this path.

A zero status causes an indirect provider call selected at dispatch-table byte offset `0xF4`. Its explicit arguments are the provider and the retained result from F-c25a2fcc75b5. The returned value is retained as the candidate pointer.

These offsets identify dispatch entries, not offsets of methods directly embedded in the object. The representation resolves the operations through the provider’s dispatch-table pointer; it does not guarantee that dependencies leave that pointer or its entries unchanged.

### Candidate gate and destination-like acquisition

A null candidate skips candidate testing and all subsequent count and transfer preparation.

For a non-null candidate, the function invokes the candidate’s operation at dispatch-table byte offset `0x08`. No explicit arguments are shown for that call. A zero result skips further processing; any nonzero result permits it.

An accepted candidate leads to:

- F-1a836ac93559 with the captured record value and constant 4. Its result becomes the initial unsigned 32-bit count.
- F-fab3799b3190 with no explicit arguments. Its 32-bit result is tested for zero.

A zero result from F-fab3799b3190 skips descriptor production, the external operation, both notification-like calls, and F-58ba61416cc9.

A nonzero result enables those operations. This retained value is described as destination-like because of its later argument positions; no allocation or destination-buffer guarantee follows from that description.

All completed provider-status-zero paths reach F-feee9156a257 with the retained candidate pointer, including null, rejected-candidate, and zero-destination-like paths. There is no retry or second candidate acquisition.

### First descriptor observation and unsigned comparison

With an accepted candidate and nonzero destination-like value, the candidate’s indirect operation at dispatch-table byte offset `0x18` receives the primary region’s base address as its sole explicit argument.

After that call returns:

- The caller reads the primary region’s 32-bit slot at offset 4.
- F-8810f53aaa0b receives that slot’s value.
- Its result is retained as an unsigned 32-bit comparison value.
- F-289ec4311c64 receives the primary region’s base address.
- The retained comparison value is compared with the initial count using unsigned ordering.

The comparison occurs after F-289ec4311c64 returns. It uses the retained result, not a fresh descriptor-slot read.

If the comparison value is greater than the initial count, the initial count remains selected without narrowing.

### Conditional middle descriptor observation

If the first comparison value is less than or equal to the initial count, the candidate operation at offset `0x18` is invoked again, this time with the **secondary region’s distinct base address**.

The caller then reads that region’s offset-4 slot and passes its value to F-8810f53aaa0b. The selected count becomes the low 16 bits of this new result, zero-extended to 32 bits. F-289ec4311c64 then receives the secondary region’s base address.

The new result is not the value used in the preceding comparison. Consequently:

- Equality takes this branch.
- The second result may differ from the first.
- The selected count can exceed the initial count.
- Only this replacement result is masked with `0xFFFF`.

No count multiplication, addition, unit conversion, or subsequent clamp is shown. The behavior is not generally equivalent to taking a minimum against one stable candidate length.

### Final descriptor use and subsequent calls

The candidate operation at offset `0x18` is invoked once more, passing the **same primary-region address used for the first observation**.

This is storage reuse, not production into a fresh independent region. Before this call, the primary region has already been exposed to its producer and F-289ec4311c64. The caller does not clear or restore its contents. Dependencies may have modified or retained access to that storage; no preservation or invalidation rule is supplied.

Following the final production, the established order is:

1. F-44362605ab79 receives the primary-region base and supplies a 32-bit result.
2. The unmapped external operation receives, in order, the retained destination-like value, that result, and the selected unsigned 32-bit count.
3. F-289ec4311c64 receives the primary-region base.
4. F-5ca71e377766 receives the function’s first argument, the retained destination-like value, and the selected count.
5. The selected count is written at the secondary-region base.
6. F-dece790ddc92 receives the record pointer, captured offset-0 value, 0, 4, `0x25`, and retained destination-like value, in that order.
7. F-58ba61416cc9 receives the retained destination-like value.
8. F-feee9156a257 receives the candidate pointer.

The caller does not read the primary region’s offset-4 slot during this final observation. F-44362605ab79’s own reads and writes remain unknown.

No return value from the external operation or the subsequent notification-like calls is inspected. A selected count of zero does not bypass this sequence.

## Side effects

The caller’s established direct writes include the selected-count write at the secondary-region base. The temporary regions are also exposed to dependencies that may read or write them.

No direct store into the input record or a named global is shown. This is not a preservation guarantee: F-dece790ddc92 receives the record pointer, and other dependencies may access globals or aliases.

Temporary-storage identity is potentially observable to dependencies:

- The first producer call, first F-289ec4311c64 call, final producer call, F-44362605ab79 call, and final F-289ec4311c64 call receive the same primary base address.
- The optional middle producer and its F-289ec4311c64 call receive the distinct secondary base address.
- A dependency retaining the secondary address may be able to observe the later selected-count write, subject to its actual access and lifetime behavior.

These relationships do not grant ownership, authorize retention beyond the storage lifetime, or guarantee that retained references remain valid. No allocation, ownership transfer, release, or reference-count contract is established for either region, the provider, the candidate, or the destination-like value.

F-289ec4311c64 follows each descriptor observation that reaches its normal consumer sequence. Calling it “cleanup” or “finalization” would be inferred intent, not established release behavior.

No explicit recovery, rollback, or compensating action is shown if a dependency does not return normally. Later calls occur only when preceding calls return.

## Dependencies

The following opaque IDs have established local uses; their implementations are not supplied.

| Opaque ID | Established use |
|---|---|
| F-c25a2fcc75b5 | Receives the record’s offset-8 value; supplies an argument for candidate acquisition. |
| F-b7af80e829ec | Supplies the provider pointer. |
| F-a817d8d58c10 | Non-returning call when the captured offset-0 value is zero. |
| F-1a836ac93559 | Supplies the initial count from the captured value and constant 4. |
| F-fab3799b3190 | Supplies the value whose nonzero result enables descriptor processing. |
| F-8810f53aaa0b | Supplies comparison or replacement values from offset-4 slot contents. |
| F-289ec4311c64 | Receives descriptor-region base addresses after their relevant use. |
| F-44362605ab79 | Supplies the external operation’s second argument from the final primary-region observation. |
| F-5ca71e377766 | Receives the first input, destination-like value, and selected count. |
| F-dece790ddc92 | Receives the record, captured value, constants, and destination-like value. |
| F-58ba61416cc9 | Receives the destination-like value after both notification-like calls. |
| F-feee9156a257 | Non-returning final call on completed provider-status-zero paths. |

Opaque function IDs are unavailable for the provider operations at `0x120` and `0xF4`, candidate operations at `0x08` and `0x18`, and external three-argument operation. They remain identified by dispatch location or call role; no dependency implementation is inferred from those identifiers.

## Acceptance examples

These are conditional consequences of the static representation, not executed tests. Dependencies are assumed to return normally where continuation is described and to support the accesses involved.

1. **Discriminator rejection.** A low discriminator halfword of 3 returns 4 without dependency calls. A discriminator word of `0xABCD0004` instead selects the processing path.

2. **Late zero check.** If the captured offset-0 value is zero, F-c25a2fcc75b5 and F-b7af80e829ec have already run before F-a817d8d58c10. No provider status call follows under its non-returning annotation.

3. **Provider rejection.** A nonzero provider status returns 4 without candidate acquisition or F-feee9156a257.

4. **Candidate short-circuits.** A null acquired candidate reaches F-feee9156a257 with null and invokes no candidate methods. A non-null candidate whose test returns zero reaches that helper without count acquisition.

5. **Zero destination-like value.** An accepted candidate followed by a zero F-fab3799b3190 result causes no descriptor production, external operation, notifications, or F-58ba61416cc9 call. The final candidate call remains.

6. **Count retained at full width.** An initial count of 70,000 and first comparison value of 80,000 select 70,000. There are two producer calls, both receiving the primary-region address.

7. **Equality and narrowing.** Initial and first comparison values of 100 trigger the middle observation. A second result of `0x00010002` selects 2. Producer-address order is primary, secondary, primary.

8. **No stable minimum guarantee.** Initial count 10, first comparison value 9, and second result 60,000 select 60,000. No later clamp to 10 is shown.

9. **No zero-count shortcut.** A selected count of zero is still passed to the external operation and F-5ca71e377766, then written at the secondary-region base before F-dece790ddc92.

10. **Storage identity is preserved.** If dependencies record producer arguments, the first and final addresses must compare equal; the conditional middle address must be distinct. A consumer must not require the final primary region to be freshly initialized by the caller: no intervening clearing is shown.

11. **Secondary write without middle production.** If the first comparison exceeds the initial count, no producer receives the secondary address. Nevertheless, the selected-count write at that region’s base still occurs after F-5ca71e377766.

## Uncertainties

**Established static evidence:** conditional field reads, represented widths, call ordering, unsigned comparison, conditional 16-bit masking, explicit argument order, non-return annotations, temporary-address reuse, distinct middle storage, absence of shown initialization, and the later secondary-base write.

**Inferred intent:** provider-based selection, descriptor-based extent inspection, transfer-like behavior, notifications, and cleanup-like helper roles.

**Documented_only unknowns:**

- Actual record and descriptor types, byte order, alignment, and count units.
- Complete temporary-region extents and dependency read/write footprints.
- Initial temporary contents and what each producer initializes or preserves.
- Whether dependencies retain addresses or inspect earlier contents.
- Whether repeated descriptor productions yield stable data.
- Implicit receiver conventions for candidate operations.
- Object identity, validity, lifetime, ownership, and release rules.
- Global or aliased-memory mutation by any dependency.
- External-operation bounds, overlap handling, data movement, and errors.
- Non-returning helpers’ mechanisms and behavior after invalid accesses.

## Evidence

This contract derives from the supplied static function representation and opaque dependency-ID map. Temporary-storage claims rely on the represented address identities, the offset-4 reads, the absence of shown initialization, and the explicit later base-slot write—not on an assumed complete descriptor layout.

No runtime tests have been performed. Acceptance examples are conditional static expectations, not observed execution results.
