<!-- Trusted provenance: F-531fdcebd26b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-531fdcebd26b

## Purpose

F-531fdcebd26b dispatches an operation against a mutable object according to the low seven bits of a 32-bit request. Depending on object state and selector, it consumes a pending flag, invokes a preliminary handler, applies a value, or searches a dependency-provided collection and attempts earlier entries.

The established behavior comprises call ordering, argument propagation, explicit memory accesses, branching, and normal returns. Navigation, playback, timing, and synchronization interpretations are **inferred intent**, not interface requirements. In particular, the constant 3000 has no established unit.

The normal result is a 32-bit zero or one. One does not uniformly indicate that a target was found or that an application operation succeeded. Zero does not imply absence of side effects.

## Interface

### Inputs and layout

The inputs are a mutable object pointer and an unsigned 32-bit request. The object must support the storage accesses and indirect calls described below. There is no initial null-pointer check.

Dispatch uses request bits 0–6. Where the request is forwarded to F-bd703d4ee601, bit 7 is set and every other bit is retained.

The supplied representation establishes these byte-relative object locations:

| Offset | Visible use |
|---|---|
| `0x000` | Primary virtual-dispatch table pointer. |
| `0x014` | Embedded interface whose notification operation occupies byte slot `0x014`. |
| `0x054` | Nullable associated-object pointer; its interface begins four bytes into that object. |
| `0x060` | Embedded state supplied to eligibility and lookup dependencies. |
| `0x074` | Embedded value supplied to a comparison-value copy dependency. |
| `0x088` | Embedded state supplied to an auxiliary operation. |
| `0x942` | One-byte pending flag. |
| `0x948` | 32-bit field supplied to F-7deb104ce23c. |
| `0x994` | 32-bit destination for a mapped value. |
| `0x998` | 32-bit destination for a predecessor value. |
| `0xA4C` | Embedded object supplied to paired region-entry and region-exit operations. |

These offsets do not define a complete object layout. Interface slots below are byte offsets within the applicable dispatch table, not object-field offsets.

### Outputs, storage, and ownership

The direct output is the 32-bit result. Additional effects occur through explicit object writes and dependency calls.

Visible temporary storage includes a 20-byte collection area, 16-byte value areas, two separate 24-byte descriptor areas, and 32-bit snapshot words. Candidate processing passes the address of local output storage represented as three 32-bit words; only its first word is visibly initialized and inspected. The dependency’s complete output-layout requirements are **documented_only**.

The caller’s object and associated object are not visibly allocated or released here. Collection and descriptor storage receives explicit initialization and cleanup calls on the paths described below. These calls do not establish the ownership behavior of their dependencies or of other callees.

All accessed objects, cached interfaces, returned addresses, and dispatch targets must remain valid when used. Unknown dependencies are not assumed to preserve globals, associated-object identity, escaped output storage, or ownership.

## Behavior

### Entry gates and selector precedence

Every invocation first calls F-b7abcf60c990 with the embedded state at `0x060`.

If that result is zero, the function returns zero without directly reading or clearing the pending byte. Otherwise, it reads `0x942`. A nonzero byte is cleared, and the function immediately returns one.

Only after these gates does selector dispatch occur:

| Selector | Action |
|---|---|
| 0 | Return zero. |
| 1 | Call F-ef61c0623fe3 with the object. Return one if nonzero; otherwise enter shared processing. |
| 2 | Execute the mapped-value path. |
| 3, 4 | Enter shared processing. |
| 5 | Execute collection search. |
| 6–127 | Return zero. |

The pending-byte clear therefore overrides all selectors. The initial eligibility call, pending-byte handling, and selector-1 preliminary call occur outside the paired regions described below.

### Selector 2: mapped value or predecessor

Eligibility is queried again through F-b7abcf60c990. Only a nonzero result permits a call to object slot `0x1A4`; only a zero result from that slot permits further processing. Failure of either condition returns zero.

The function enters the region associated with `0xA4C`. Every normal exit after this entry invokes the corresponding exit operation.

It reads the associated-object pointer at `0x054`. A zero pointer produces region exit and result zero. Otherwise, it captures the interface at that pointer plus four. That captured interface is used for the associated calls in this path; changing the object’s associated-pointer field does not redirect those calls.

The sequence is:

1. Query captured associated slot `0x138`. Zero produces region exit and result zero.
2. Obtain a 32-bit value through object slot `0x17C`.
3. Pass that value to captured associated slot `0x144`, obtaining an index-like 32-bit result.
4. Pass that result to captured associated slot `0x13C`, obtaining the first mapped value.
5. Call object slot `0x17C` again, obtaining a fresh 32-bit comparison value.
6. Load a 32-bit datum from external storage.

The external datum has unknown mutability. Its load occurs after the second slot-`0x17C` call. Subsequent use on this path uses the captured datum, not a fresh external load.

The function adds 3000 to the first mapped value using unsigned 32-bit arithmetic, then compares that sum strictly below the fresh comparison value. Addition wraps modulo 2³².

If the comparison succeeds, object slot `0x100` receives the first mapped value. The embedded interface at `0x014` is then notified through slot `0x014`, with the captured external datum and zero. Region exit follows, and the result is one.

If the comparison fails and the index-like result is nonzero:

1. Captured associated slot `0x13C` receives the index-like result minus one.
2. Object slot `0x100` receives the returned predecessor value.
3. The predecessor value is written to `0x998`.
4. The first mapped value is written to `0x994`.
5. F-069a1d444a2d receives the object and captured external datum.
6. The embedded notification receives that same captured datum and zero.
7. The region exits and the function returns one.

The decrement is a 32-bit operation in the supplied representation; no nonnegative-range validation is shown. The domain meaning of out-of-range index bit patterns remains unresolved.

If the comparison fails and the index-like result is zero, the function exits the region and returns zero without the repositioning calls, direct destination writes, or notification.

### Selectors 1, 3, and 4: shared processing

The function enters the region at `0xA4C`. A zero associated-object pointer produces region exit and result zero.

For a nonzero pointer, it captures the associated interface at pointer-plus-four **before** calling object slot `0x17C`. It then:

1. Obtains the current 32-bit value through object slot `0x17C`.
2. Calls slot `0x094` through the previously captured associated interface.
3. Compares the second result plus 3000 against the current value, using wrapping unsigned 32-bit addition and strict unsigned less-than.

If slot `0x17C` changes the object’s associated-pointer field, slot `0x094` still targets the captured interface. That interface must remain valid independently of the field’s current contents.

If the threshold comparison succeeds, the function obtains a value through object slot `0x158`, calls F-6542ebfeed7d with the object, that value, and zero, then calls F-9585ac57a521. It exits the region and returns one without testing an application result.

Otherwise, candidate processing follows.

#### Candidate acquisition and immediate output-byte decision

The first word of local output storage is initialized to zero. The function obtains a value through object slot `0x158`, supplies it with the embedded state at `0x060` to F-e41650e0b8d9, and reads a 32-bit field at offset `0x10` from the returned address.

No null check protects that field read.

F-bd703d4ee601 receives the object, the retrieved field, the original request with bit 7 set, and the local output-storage address.

A zero candidate result causes object slot `0x0EC` to be called, followed by region exit and result zero.

For a nonzero candidate result, the function reads the output word’s low byte. If nonzero, it performs this ordered sequence:

1. F-76b7ec0cf6ba.
2. F-cb37c5f052d2.
3. F-7deb104ce23c with the then-current 32-bit field at `0x948`.
4. The auxiliary operation on the embedded state at `0x088`.
5. F-b582c09a05ae with the object.

This byte read controls only whether that immediate sequence runs. It is not a permanent snapshot of the flag subsequently forwarded to the application operation.

#### Descriptors, snapshots, and completion

F-6fa4e01d8d29 receives the nonzero candidate result, `0x33`, and zero, producing an extracted value.

The function initializes the first 24-byte descriptor through F-ba5a4c1e0746 with zero and zero, then initializes the second descriptor identically. It obtains a fresh value through object slot `0x158`, then calls object slot `0x170` twice: first with that fresh value and the first descriptor, then with the extracted value and the second descriptor.

F-6e482452607b queries the first descriptor. Only a nonzero result permits querying the second descriptor. If the first result is nonzero and the second is zero, F-76b7ec0cf6ba and then F-cb37c5f052d2 are called. This pair can therefore run twice in one invocation.

Next:

1. The associated-object pointer is reloaded, and associated slot `0x168` receives an address for a first 32-bit snapshot.
2. The candidate output word is read again and masked to its low byte.
3. F-6542ebfeed7d receives the object, extracted value, and this newly read byte value.
4. The associated-object pointer is reloaded again, and associated slot `0x168` receives an address for a second snapshot.
5. The snapshot words are compared. Equality causes F-9585ac57a521 to be called.

These slot-`0x168` calls differ from the earlier cached-interface slot-`0x094` call: each uses the associated pointer available at its own later call site. No intervening null check is shown.

The output address previously escaped to F-bd703d4ee601. Preservation of that output word across intervening dependencies is not established. Consequently, the byte controlling the immediate auxiliary sequence and the byte forwarded to F-6542ebfeed7d may differ. Likewise, snapshot interpretation depends on dependency output and preservation behavior; the visible decision compares the words present at comparison time.

Object slot `0x190` is then queried. Zero selects F-04d3f7ecf9cb with the current associated-object pointer, zero, and zero. Nonzero selects F-0edcc41cb6b7 without explicit arguments.

The second descriptor is cleaned up before the first through F-4e292a3dd745. The region exits, and the function returns one.

### Selector 5: forward search and backward attempts

The function enters the region at `0xA4C`. If the associated-object pointer is zero, it exits and returns zero.

Otherwise, F-df2422ba76c2 initializes a 20-byte collection area with zero and zero. F-2588e3ea1153 supplies an unsigned 32-bit count. F-961dd3c5e67e then receives a 16-byte comparison temporary and the embedded value at `0x074`. This copy operation occurs even for count zero.

For a nonzero count, F-e478ae4ee528 fetches element zero into the current-value temporary. A forward counter begins at one. F-d45d65316ed3 compares the copied comparison value with the current value:

- Nonzero stops forward searching.
- Zero advances only while the counter is below the count.
- Each advance fetches the element at the counter through F-e478ae4ee528, assigns it into the current temporary through F-1033f0501634, and increments the counter.

Thus a nonzero comparison at element index *k* leaves counter *k* + 1. If all *N* comparisons are zero, the final counter is *N*.

Counter one skips all backward attempts. Otherwise, the function forms counter minus two as a 32-bit index and fetches that entry before testing its signed nonnegativity.

For each backward attempt, it assigns the fetched value into the current temporary, calls object slot `0x054` with that temporary and additional arguments zero, zero, three, three, zero, then queries object slot `0x16C`.

A nonzero query ends the loop. Success requires that the current index also be nonnegative under signed 32-bit interpretation. On success, object slot `0x0E0` receives `0xFFFFFFFF`, zero, zero, zero; the collection is cleaned up through F-ccf4fcd3c34d; the region exits; and the function returns one.

After a zero query, a negative current index terminates unsuccessfully. A nonnegative index is instead decremented and fetched, allowing another attempt. Failure at index zero therefore causes a fetch and attempt at index −1. A nonzero query at that negative index still does not produce success.

All other normal collection exits clean up the collection, exit the region, and return zero. Large counts retain the unsigned forward-counter behavior and signed backward checks; ordinary nonnegative-index intuition is not sufficient to establish safe dependency behavior.

## Side effects

The explicit persistent writes are:

- Clearing the pending byte at `0x942` after successful initial eligibility.
- Writing `0x998`, then `0x994`, on the selector-2 predecessor path.

Dependency effects are not bounded by these writes. Zero-return paths can include region operations, failed candidate handling, collection attempts, and dependency mutations. No rollback is established.

The paired region operations resemble synchronization, but their locking, recursion, and failure semantics are **documented_only**. Every visible normal exit after entry invokes exit. Exceptional or nonreturning dependency behavior is not covered by a visible recovery mechanism.

## Dependencies

Opaque IDs identify dependencies without assigning unverified semantics:

| IDs | Established role |
|---|---|
| F-b7abcf60c990; F-ef61c0623fe3 | Eligibility query; selector-1 preliminary handler. |
| F-069a1d444a2d | Selector-2 predecessor-path operation. |
| F-e41650e0b8d9; F-bd703d4ee601 | Address-producing lookup; candidate operation receiving output storage. |
| F-6fa4e01d8d29; F-6542ebfeed7d; F-9585ac57a521 | Extraction; application; subsequent object operation. |
| F-76b7ec0cf6ba; F-cb37c5f052d2 | Ordered auxiliary pair. |
| F-7deb104ce23c; F-b582c09a05ae | Additional byte-controlled operations. |
| F-ba5a4c1e0746; F-6e482452607b; F-4e292a3dd745 | Descriptor initialization, query, cleanup. |
| F-04d3f7ecf9cb; F-0edcc41cb6b7 | Alternatives selected by slot `0x190`. |
| F-df2422ba76c2; F-2588e3ea1153; F-ccf4fcd3c34d | Collection initialization, count, cleanup. |
| F-961dd3c5e67e; F-e478ae4ee528 | Comparison-value copy; indexed fetch. |
| F-d45d65316ed3; F-1033f0501634 | Forward comparison; temporary assignment. |

The region operations and auxiliary operation at `0x088` have no supplied opaque IDs and remain **documented_only** dependencies. Virtual dependencies are identified by interface and byte slot. No dependency’s name or call role establishes its ownership or global-state effects.

## Acceptance examples

These are conditional deductions, not executed tests. They assume valid storage and normally returning dependencies.

1. **Eligibility precedes pending consumption.** Initial eligibility zero produces result zero without a direct pending-byte clear. Eligibility nonzero with a pending byte set clears it and returns one even for selector 127.

2. **Request forwarding retains upper bits.** Request `0x00001203` selects shared processing. If candidate processing is reached, F-bd703d4ee601 receives `0x00001283`.

3. **Cached shared interface survives pointer-field replacement.** If the associated pointer initially identifies object A and slot `0x17C` changes the field to object B, slot `0x094` still targets A’s captured interface. Later slot-`0x168` calls use the pointer reloaded at each call site. A must remain valid for its cached call.

4. **Threshold equality and wraparound differ.** Values 1000 and 4000 fail the strict comparison because 1000 + 3000 equals 4000. A mapped value `0xFFFFFFF0` instead wraps to 2984 after addition and compares below 4000.

5. **Notification data is captured, not constant.** If the external datum is D when loaded after the second slot-`0x17C` call, later mutation of that storage does not change the captured argument. On the predecessor path, F-069a1d444a2d and the notification receive D.

6. **Predecessor writes follow application.** With index two and predecessor mapping 700, slot `0x100` receives 700 before the writes of 700 to `0x998` and the first mapped value to `0x994`.

7. **The output byte is read twice.** An immediate output word `0x00000100` skips the auxiliary sequence; `0x00000102` enters it. Only if the word remains unchanged until the later read are the forwarded flags respectively zero and two. If it changes from `0x00000102` to `0x00000100` between reads, the sequence runs but the forwarded flag is zero.

8. **Candidate failure calls outward.** Candidate result zero causes slot `0x0EC`, region exit, and result zero.

9. **One element yields no backward attempt.** For count one, either comparison result leaves counter one, so the backward path does not call slot `0x054`.

10. **Exhaustion includes index −1.** If backward queries through index zero return zero, the next fetch and attempt use −1. Assuming those operations return normally, even a nonzero query there produces result zero.

## Uncertainties

- **documented_only:** Domain meanings and units of values, indices, flags, descriptors, snapshots, and notification data.
- **documented_only:** External notification-storage mutability between loads or invocations; only the load and subsequent captured-value reuse are established.
- **documented_only:** Ownership, retained addresses, global mutations, and memory preservation by unknown dependencies.
- **documented_only:** Safety and meaning of negative or unusually large collection indices, including −1.
- **documented_only:** Complete dependency-defined temporary layouts and any cleanup obligations beyond explicit calls.
- **documented_only:** Invalid-pointer handling, concurrency, exceptions, and nonreturning dependencies. No recovery contract is established.
- **Inferred intent:** Region operations provide synchronization, values represent positions or times, and backward attempts select earlier entries. None is required by the established behavior.

## Evidence

The contract derives from the supplied static representation and opaque dependency mapping. Established evidence includes offsets, storage widths, branch conditions, call arguments and order, cached versus reloaded interfaces, external-data capture, separate output-byte reads, explicit writes, cleanup placement, and normal results.

Arithmetic descriptions preserve unsigned 32-bit threshold wrapping, request masks, low-byte masks, and signed backward-index checks. Dependency semantics and safe index domains are not inferred from call shapes.

No dependency-body inspection, runtime tests, or behavioral experiments have been performed. Acceptance examples express conditional static consequences, not observed execution.
