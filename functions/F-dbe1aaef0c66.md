<!-- Trusted provenance: F-dbe1aaef0c66; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dbe1aaef0c66

## Purpose

F-dbe1aaef0c66 conditionally obtains a candidate object, determines an unsigned transfer quantity, invokes an external transfer operation, and issues two subsequent notifications before cleanup-related calls.

The routine processes only records whose 16-bit discriminator equals 4. Other records are ignored and receive the same normal return value, 4.

The apparent intent is to transfer some or all of a candidate object's data into a newly obtained destination and report the result. That interpretation is inferred from the call sequence. The supplied evidence does not establish the domain meaning of the objects, the transfer operation, or the notifications.

This contract distinguishes:

- **Established static behavior:** explicit reads, branches, argument values, and call ordering.
- **Inferred intent:** interpretations such as “transfer,” “destination,” and “cleanup.”
- **Unknown behavior:** effects of dependencies, virtual operations, and the external operation. These remain `documented_only`.

No runtime tests have been performed.

## Interface

### Inputs

The routine accepts two arguments:

1. **Context token:** a 32-bit value. The routine does not dereference it directly. It forwards this value to F-5ca71e377766 on the full processing path.
2. **Record pointer:** a pointer to a record accessed using 32-bit word spacing.

The directly accessed record fields are:

| Byte offset | Access | Meaning established by use |
|---|---|---|
| 0 | One 32-bit read on the matching-discriminator path | Subject token, subsequently tested for zero and passed to dependencies |
| 8 | One 32-bit read on the matching-discriminator path | Input to F-c25a2fcc75b5 |
| 12 | A 32-bit field narrowed to 16 bits for comparison | Discriminator; processing requires its low 16 bits to equal 4 |

No direct access to the field at offset 4 is shown.

The discriminator comparison does not require the entire 32-bit field at offset 12 to equal 4. Its upper 16 bits do not affect the branch.

### Output

The normal return value is the 32-bit value 4.

This value is not a success indicator: it is also returned when the discriminator does not match, when the provider gate rejects processing, and—if relevant dependencies return normally—after several other paths that perform no transfer.

Some dependencies may prevent normal return. Two call sites carry non-returning annotations in the supplied material, but those annotations alone do not establish the dependencies' actual runtime behavior.

### Preconditions

For defined execution of the routine's direct accesses:

- The record pointer must permit the discriminator read at offset 12.
- When the discriminator matches, the record must also permit reads at offsets 8 and 0.
- The provider returned by F-b7af80e829ec must support the indirect operations used below. There is no explicit provider-null check.
- Any non-null candidate used for indirect operations must have a valid dispatch structure and the required entries.
- Dependencies writing to temporary descriptor storage must obey the storage and lifetime requirements of their actual interfaces. Those interfaces are not supplied.

There is no explicit record-pointer validation, bounds validation, or exception recovery.

## Behavior

### 1. Discriminator gate

The routine first reads the record's discriminator and compares its low 16 bits with 4.

If they differ:

- No dependency is called.
- The subject and lookup fields are not directly read.
- The routine returns 4.

If they match, processing continues.

### 2. Lookup preparation and subject capture

The following sequence is established:

1. Read the record field at offset 8 and call F-c25a2fcc75b5 with that value.
2. Retain the resulting 32-bit lookup token.
3. Call F-b7af80e829ec to obtain a provider pointer.
4. Read the record field at offset 0 and retain that value as the subject token.
5. If the subject token is zero, call F-a817d8d58c10.

The timing of the subject read matters. The first two dependencies execute before it. If either dependency changes the record through shared state, the later read observes the resulting subject value, not a value captured at entry.

F-a817d8d58c10 is annotated as non-returning. If it actually does not return, no later operation occurs. If it does return, the visible control flow continues with the zero subject token; no explicit fallback return or second zero check is shown.

### 3. Provider gate and candidate acquisition

The routine invokes the provider operation at dispatch-table byte offset 0x120, supplying the provider pointer.

If that operation returns nonzero:

- Candidate acquisition is skipped.
- Candidate finalization is skipped.
- No transfer-related operation occurs.
- The routine reaches the normal return of 4.

If it returns zero, the routine invokes the provider operation at byte offset 0xF4, supplying the provider pointer and the retained lookup token. Its result is treated as a candidate pointer.

The candidate is eligible for further processing only if:

1. The pointer is non-null.
2. Its operation at dispatch-table byte offset 0x08 returns nonzero.

The visible invocation of the operation at offset 0x08 supplies no explicit argument. The actual calling convention and any implicit receiver behavior are `documented_only`.

If either eligibility condition fails, the transfer path is skipped, but candidate finalization described below is still reached.

### 4. Initial quantity and destination acquisition

For an eligible candidate:

1. Call F-1a836ac93559 with the captured subject token and the literal 4.
2. Interpret its result as an unsigned 32-bit initial quantity.
3. Call F-fab3799b3190 to obtain a destination token.

If the destination token is zero:

- No descriptor operation occurs.
- No external transfer occurs.
- Neither notification is issued.
- F-58ba61416cc9 is not called.
- Processing proceeds to candidate finalization.

A nonzero destination token enables the remaining transfer path.

### 5. First descriptor observation and unsigned comparison

The routine invokes the candidate operation at dispatch-table byte offset 0x1C, supplying temporary descriptor storage.

The visible use of this storage treats it as having:

- A first 32-bit component at offset 0.
- A second 32-bit component at offset 4.

It passes the second component to F-8810f53aaa0b and treats the result as an unsigned 32-bit quantity. It then calls F-289ec4311c64 with the descriptor address.

Only after that call does it act on the comparison between the observed quantity and the initial quantity.

The comparison is unsigned:

- If the observed quantity is greater than the initial quantity, the initial quantity remains selected.
- If the observed quantity is less than or equal to the initial quantity, a second descriptor observation determines the selected quantity.

This is not, in general, a simple minimum operation. The second observation can produce a different value, and only that second result is narrowed to 16 bits.

### 6. Conditional second descriptor observation

When the first observed quantity is less than or equal to the initial quantity:

1. Invoke the candidate descriptor operation again, using a separate temporary descriptor location.
2. Pass this descriptor's second 32-bit component to F-8810f53aaa0b.
3. Retain only the low 16 bits of the conversion result.
4. Call F-289ec4311c64 with this descriptor's address.

The selected quantity is therefore an unsigned value in the range 0 through 65,535 on this branch.

The second observation is independent of the first. The routine neither caches the first descriptor nor requires the two conversions to agree.

On the branch that skips this second observation, the selected quantity retains its full 32-bit width.

### 7. Transfer descriptor and external operation

The routine performs another candidate descriptor operation, regardless of which quantity branch was taken.

Using that newly produced descriptor:

1. Call F-44362605ab79 with the descriptor address.
2. Retain its 32-bit result as the external operation's source token.
3. Call D-EXT-01 with:
   - The nonzero destination token.
   - The source token.
   - The selected unsigned quantity.
4. Call F-289ec4311c64 with the descriptor address.

D-EXT-01 is an opaque label assigned here to the external dependency for which no mapped function ID was supplied.

No return value from D-EXT-01 controls subsequent behavior. In particular, no explicit transfer-success check is shown before notifications are issued.

A selected quantity of zero does not skip this sequence.

### 8. Notifications and destination finalization

After descriptor finalization following D-EXT-01:

1. Call F-5ca71e377766 with the context token, destination token, and selected quantity.
2. Write the selected quantity into a local 32-bit temporary.
3. Call F-dece790ddc92 with:
   - The original record pointer.
   - The captured subject token.
   - Literal 0.
   - Literal 4.
   - Literal 0x23.
   - The destination token.
4. Call F-58ba61416cc9 with the destination token.

The selected quantity is not an explicit argument to F-dece790ddc92. The preceding local write does not establish that this dependency receives it through an additional argument or that the record is updated with it.

F-dece790ddc92 receives the original record pointer, but the subject argument remains the value captured earlier. The routine does not directly reread that field before this call.

### 9. Candidate finalization and return

Whenever the provider gate returned zero and candidate acquisition returned normally, F-feee9156a257 is reached after the applicable candidate-processing path.

Its argument is the acquired candidate pointer, including null when acquisition returned null.

This call also occurs after:

- A candidate eligibility result of zero.
- A zero destination result.
- Completion of the full transfer and notification path.

The call site is annotated as non-returning. Its actual termination, release, or ownership behavior is not established. If it returns normally, the routine reaches its normal return value of 4.

## Side effects

### Direct memory effects

The routine directly reads the three record locations described above. It does not visibly write a record field.

It writes temporary storage used for descriptors and stores the selected quantity in a local temporary. These writes do not by themselves establish any persistent record-state change.

Provider and candidate dispatch pointers are read as required for indirect calls.

### Dependency-mediated effects

Dependencies may mutate records, objects, global state, or external resources. Their implementations are unavailable. The contract therefore does not assume that any dependency:

- Preserves global state.
- Leaves the record unchanged.
- Produces stable descriptor contents across calls.
- Acquires or releases ownership.
- Completes successfully merely because it returns.
- Implements a particular failure-reporting mechanism.

The visible ordering of calls remains significant even where their effects are unknown.

### Ownership and lifetime

The routine explicitly invokes:

- F-289ec4311c64 once for each descriptor observation.
- F-58ba61416cc9 once for a nonzero destination after the full notification path.
- F-feee9156a257 once after candidate acquisition on the provider-gate-zero path.

These are finalization-shaped calls, not proof of destruction or release. Ownership transfer, reference counting, null acceptance, and retained aliases are `documented_only`.

There is no explicit finalization of the provider or lookup token.

No visible recovery path ensures later finalization if an earlier dependency fails to return normally.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-c25a2fcc75b5 | Converts or resolves the record's offset-8 value into a lookup token |
| F-b7af80e829ec | Supplies the provider pointer |
| F-a817d8d58c10 | Called for a zero captured subject; annotated non-returning |
| F-1a836ac93559 | Supplies the initial unsigned quantity from the subject and literal 4 |
| F-fab3799b3190 | Supplies the destination token; zero skips transfer processing |
| F-8810f53aaa0b | Converts each relevant descriptor's second component into an unsigned quantity |
| F-289ec4311c64 | Receives each descriptor after its immediate use |
| F-44362605ab79 | Produces the source token from the final descriptor |
| F-5ca71e377766 | Receives context, destination, and selected quantity after the external operation |
| F-dece790ddc92 | Receives the record, captured subject, fixed literals, and destination |
| F-58ba61416cc9 | Receives the destination after both notifications |
| F-feee9156a257 | Receives the acquired candidate, including null; annotated non-returning |
| D-EXT-01 | Receives destination, source, and selected quantity |

The unresolved indirect operations are assigned opaque IDs for reference:

| Opaque ID | Dispatch offset | Role established by use |
|---|---:|---|
| D-V01 | 0x120 | Provider gate |
| D-V02 | 0xF4 | Candidate acquisition using the lookup token |
| D-V03 | 0x08 | Candidate eligibility check |
| D-V04 | 0x1C | Descriptor production into supplied temporary storage |

## Acceptance examples

These are conditional contract examples, not executed tests. Unless otherwise stated, invoked dependencies return normally and provide valid objects and descriptor storage.

1. **Upper discriminator bits are ignored.**  
   A discriminator word of 0xABCD0004 enters processing. A word of 0x00010003 returns 4 without any dependency call.

2. **Subject capture follows preliminary dependencies.**  
   If the subject initially equals 9, but F-c25a2fcc75b5 changes it to 12 before returning, the subsequently captured subject is 12, provided no intervening dependency changes it again.

3. **Provider rejection bypasses candidate finalization.**  
   If D-V01 returns nonzero, D-V02 and F-feee9156a257 are not called. The normal result is 4.

4. **Null candidate still reaches finalization.**  
   If D-V02 returns null, no candidate virtual operation or quantity calculation occurs. F-feee9156a257 receives null. Whether execution then terminates or returns depends on that dependency.

5. **Zero destination skips all descriptors.**  
   For an eligible candidate, F-1a836ac93559 is called before F-fab3799b3190. If the latter returns zero, D-V04, D-EXT-01, both notifications, and destination finalization are skipped.

6. **Greater first observation preserves full width.**  
   Given initial quantity 70,000 and first converted observation 80,000, the selected quantity remains 70,000. There are two descriptor observations: the comparison observation and the transfer observation.

7. **Equality takes the narrowing branch.**  
   Given initial quantity 100 and first converted observation 100, the second observation occurs. If its conversion returns 65,538, the selected quantity is 2. There are three descriptor observations in total.

8. **The second observation need not respect the initial bound.**  
   Given initial quantity 10, first observation 5, and second conversion result 50, the selected quantity is 50. No subsequent clamp to 10 is shown.

9. **Zero quantity still reaches the external operation.**  
   If the selected quantity is zero and the destination is nonzero, D-EXT-01 receives zero as its third argument. Both notifications and destination finalization follow if preceding calls return normally.

10. **External return status is not a gate.**  
    If D-EXT-01 returns normally with any status value, that value does not alter the visible continuation: descriptor finalization precedes F-5ca71e377766, then F-dece790ddc92, then F-58ba61416cc9.

## Uncertainties

The following remain `documented_only`:

- The domain meaning and signedness of non-quantity tokens.
- The exact calling conventions of indirect operations.
- The complete descriptor format, including whether it contains more than the two visibly consumed 32-bit components.
- Whether descriptor production initializes all storage later read.
- The semantic meaning, overlap rules, and failure behavior of D-EXT-01.
- The ownership effects of all finalization-shaped dependencies.
- Whether either non-returning annotation is accurate.
- The effect of the local quantity write beyond the visible local store.
- Dependency behavior under mutation, reentrancy, invalid objects, allocation failure, exceptions, or other nonlocal exits.

The apparent transfer-and-notify purpose does not resolve these unknowns.

## Evidence

This contract is based solely on the supplied static routine representation and dependency-ID mapping.

Static evidence establishes the discriminator gate, directly accessed offsets, argument values, unsigned comparison, conditional 16-bit masking, repeated descriptor observations, call ordering, and constant normal return value.

Descriptions such as provider, candidate, destination, transfer, notification, and finalization are role labels inferred from use, not independently verified API semantics. Non-returning annotations are reported as annotations rather than accepted as runtime proof.

No dependency implementations, external interface specifications, execution traces, or runtime tests were supplied. No runtime tests have been performed.
