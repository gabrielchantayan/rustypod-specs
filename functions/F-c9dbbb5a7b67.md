<!-- Trusted provenance: F-c9dbbb5a7b67; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c9dbbb5a7b67

## Purpose

F-c9dbbb5a7b67 clears an outer-object flag, invokes a fixed sequence of dependencies, modifies one table record, and selects between two completion paths using a byte read through another table-base observation.

The zero-byte path invokes a dependency, follows a pointer chain, and conditionally transfers to a destination read from a global function-pointer slot. The nonzero-byte path repeatedly invokes a dependency until it returns zero, then conditionally transfers through a different global function-pointer slot.

Two distinct reads of the same second-table base global serve different purposes: the first supplies the retained base for repeated byte reads; the second supplies the record address used for branch selection and the zero-byte path. These observations are not established to have equal values.

This contract describes statically established behavior. Terms such as “context,” “record,” and “completion predicate” identify structural roles, not application-level meaning.

## Interface

### Explicit arguments

| Position | Represented width | Visible use |
|---|---:|---|
| First | 32 bits | Address of an outer object that is read and modified. |
| Second | 32 bits | No explicit use in the visible body. |
| Third | 32 bits | No explicit use in the visible body. |
| Fourth | 32 bits | Forwarded unchanged on every invocation of F-0ea4820fe774. |

Four additional 32-bit values appear in calls to F-0ea4820fe774 as incoming register or control-state contents. Their provenance and public-interface status are unresolved. They are not established as four additional public arguments to F-c9dbbb5a7b67.

### Output and completion

No caller-visible return value is established. Dependency results determine record selection, repetition, and conditional transfers.

Normal completion requires the executed dependencies to return normally and all executed memory accesses to be valid. The repeated-processing path has no visible iteration limit. No exception handling, cancellation, or fault recovery is shown.

The two indirect sites are represented as calls followed by return. Their exact ABI form, including whether either is a tail transfer, remains unresolved.

### Storage and widths

All offsets below are byte offsets. Addresses, table-base values, function-pointer slots, and word accesses are represented as 32 bits. Byte accesses are 8 bits.

**Outer object**

| Offset | Access | Role |
|---|---|---|
| +0x08 | Two separate 32-bit reads | Context address before and after the first dependency call. |
| +0x0C | 8-bit write | Cleared before any explicit dependency call. |

**Retained context**

| Offset | Access | Role |
|---|---|---|
| +0x00 | 32-bit read | Pointer examined after the repeated-processing loop terminates. |
| +0x34 | Two separate 32-bit reads | Values supplied to F-6518403de3d0 and F-ddf4a1f94e38. |

**First-table record**

The record stride is 0x18, or 24 bytes.

| Offset | Access | Assigned value |
|---|---|---:|
| +0x13 | 8-bit write | 0 |
| +0x14 | 8-bit write | 0 |
| +0x08 | 32-bit write | 0 |
| +0x10 | 8-bit write | 0xFF |

**Records addressed through the second-table global**

One selector result is combined separately with two observations of the same table-base global, using a 24-byte stride.

| Base observation | Record access | Role |
|---|---|---|
| First | Byte at +0x00 | Read for every invocation of F-0ea4820fe774. |
| Second | Byte at +0x11 | Selects the zero-byte or nonzero-byte path. |
| Second | 32-bit pointer at +0x04 | Starts the pointer chain on the zero-byte path. |

The loop-byte record and branch/pointer record coincide only if the two base observations produce the same effective address. The function performs no comparison between them.

**Global slots**

The visible body reads:

- One first-table base slot, after the first selector call.
- One second-table base slot twice, after the second selector call.
- A function-pointer slot specific to indirect site A, only when that site is reached.
- A separate function-pointer slot specific to indirect site B, only when that site is reached.

These slots are not established to be immutable.

### Preconditions

The visible body performs no null, bounds, alignment, or lifetime checks except the final context-pointer guard. Every address used on the executed path must support the corresponding access for normal completion.

No ownership relationship follows from these layouts. Tables may alias each other, the context, or other accessed storage; disjointness is not established.

## Behavior

### Initial state change and context acquisition

The first explicit operation clears the outer object’s byte at +0x0C.

The function then reads the outer object’s 32-bit field at +0x08 and invokes F-494f99f352ec with that value and the constant 0x83.

After the dependency returns, the function reads the outer field at +0x08 again. This second observation supplies the context address retained for subsequent operations.

Consequently, a change to the outer context field during F-494f99f352ec can change the retained context. A later change to that outer field does not itself replace the retained address.

The function reads the retained context’s word at +0x34 and supplies it to F-6518403de3d0.

### First record selection and modification

After F-6518403de3d0 returns, F-36a285714182 is called with the retained context address and the constant 2.

The 32-bit selector result determines a 24-byte displacement from the first-table base. That base is read from its global slot after the selector returns.

At the resulting record address, the function performs these writes in order:

1. Clear byte +0x13.
2. Clear byte +0x14.
3. Clear the 32-bit field at +0x08.
4. Set byte +0x10 to 0xFF.

The address used for these writes is retained; the table-base global is not reread between them. Atomicity or transactional visibility is not established.

F-338d16f9ce72 is then invoked with no explicit arguments.

### Second context-field read and record selection

After F-338d16f9ce72 returns, the retained context’s word at +0x34 is read again. F-ddf4a1f94e38 receives this newly observed value and the constant 2.

The earlier +0x34 value is not reused. Dependencies or aliasing writes before this second read can therefore affect its value.

F-36a285714182 is then called again with the retained context and 2. Its result independently selects the displacement for subsequent second-table accesses. It need not equal the first selector result.

After this second selector call, two separate reads of the second-table base global occur, in this order:

1. The first observed base is retained for loop-byte accesses.
2. The second observed base is combined with the selector result to form the branch/pointer record address.

No explicit dependency call occurs between these two reads. Nevertheless, no stability or synchronization guarantee establishes equal observations. The reads must not be collapsed into one observation in this contract.

The byte at +0x11 of the branch/pointer record selects the path:

- Zero selects the zero-byte path.
- Any nonzero byte selects the repeated-processing path.

Signed interpretation of the byte does not change this zero-versus-nonzero decision.

### Zero-byte path

F-1333331b2da1 is called with the retained context.

After it returns, the function follows a pointer chain from the retained branch/pointer record address:

- Read the 32-bit pointer at record offset +0x04.
- Read the 32-bit word addressed by that pointer.

The resulting word is supplied to F-9813035c7ce7 as its second argument; its first argument is 1.

The branch/pointer record address is retained across F-1333331b2da1. Its +0x04 field and the pointed-to word are read afterward. Changes to those locations during the dependency can therefore affect the supplied value. A change only to the second-table base global does not redirect these accesses.

If F-9813035c7ce7 returns a value other than the 32-bit −1 representation, 0xFFFFFFFF, the function returns without reaching either indirect site.

If the result is 0xFFFFFFFF, the previously supplied second-argument value is used as an address. The function reads a 32-bit word at that address plus 0x04 and supplies the word as the explicit argument at indirect site A.

The destination at site A is obtained from that site’s own global function-pointer slot when the transfer is reached, after F-9813035c7ce7 has returned. It is not a destination captured before that dependency. A dependency that changes the slot can therefore affect the destination invoked.

Both the argument-word access and destination-slot read belong to this post-sentinel path. Their relative read order is not established by the supplied representation.

This path does not execute F-0ea4820fe774 or the final context-pointer predicate.

### Nonzero-byte path

F-0ea4820fe774 is invoked at least once. Each invocation receives, in represented order:

1. The retained context address.
2. The byte at the selected displacement from the **first** second-table base observation.
3. The constant 1.
4. The fourth explicit argument to F-c9dbbb5a7b67.
5. Four incoming 32-bit register/control-state values whose ABI meaning is unresolved.

The byte is reread for every invocation. It is not captured once before repetition. Its storage may differ from the record whose +0x11 byte selected this path.

The first base observation and the second selector result remain retained throughout repetition. The function does not rerun the selector or reread the table-base global on later iterations. Changes to the byte at the retained address can affect later arguments; changing only the global base cannot redirect them.

Any nonzero result from F-0ea4820fe774 causes another invocation. A zero result ends repetition. No visible retry bound, delay, or alternative exit exists.

After the zero result, the function reads the pointer at the retained context’s +0x00:

- A null pointer causes return without a dereference.
- A nonnull pointer is dereferenced for a 32-bit word.
- A zero word causes return.
- A nonzero word causes indirect site B to be reached.

At site B, the destination is read from its own global function-pointer slot, after the loop has terminated and the final predicate has passed. No explicit arguments are shown. The predicate word is not shown as the transfer destination.

A change to this slot during a preceding dependency, including a loop invocation, can affect the destination invoked. The destination is not captured before the loop.

### Arithmetic boundaries

Each selector result is 32 bits, and each record displacement uses multiplication by 24 before combination with a represented 32-bit base.

No index-range check, negative-index rejection, multiplication-overflow check, or address-overflow check is visible. The representation does not establish portable signed-overflow behavior or checked arithmetic. Normal completion requires valid effective addresses for all executed accesses; a nonnegative selector result alone is insufficient to establish validity.

## Side effects

The directly established writes are one cleared byte in the outer object and three byte assignments plus one word assignment in the first-table record.

There is no visible allocation, deallocation, reference-count operation, or explicit ownership transfer. Clearing the first record’s +0x08 word does not establish release of any object previously referenced there.

No dependency is assumed to preserve globals, object contents, pointer targets, or allocation lifetimes. In particular, dependencies may affect:

- The context address observed after F-494f99f352ec.
- Either later read of the retained context’s +0x34 field.
- Table-base values observed after selector calls.
- The zero-path pointer chain or repeated-path byte.
- The final context pointer and its target.
- Either global indirect-destination slot before it is read.

The function retains selected addresses rather than continuously following global replacements. If a dependency invalidates retained storage, later accesses have no visible recovery mechanism.

Aliasing can make the explicit record writes affect other subsequently read fields. There is no established nonaliasing guarantee to exclude this.

No rollback is shown if a later dependency fails to return, a memory access is invalid, or repetition never terminates.

## Dependencies

| Opaque dependency or site | Visible contract within this function |
|---|---|
| F-494f99f352ec | Receives the first outer context-field observation and 0x83. Result unused. |
| F-6518403de3d0 | Receives the first retained-context +0x34 observation. Result unused. |
| F-36a285714182 | Called twice with retained context and 2. Each 32-bit result independently determines a record displacement. |
| F-338d16f9ce72 | Called after the four first-record writes. No explicit arguments. |
| F-ddf4a1f94e38 | Receives the later retained-context +0x34 observation and 2. Result unused. |
| F-1333331b2da1 | Called only on the zero-byte path, before the pointer-chain reads. Receives retained context. |
| F-9813035c7ce7 | Receives 1 and the pointer-chain result. Only 0xFFFFFFFF selects site A. |
| F-0ea4820fe774 | Invoked repeatedly on the nonzero-byte path. Zero terminates repetition; any nonzero result continues it. |
| Indirect site A | Reads its destination from its own global function-pointer slot after the sentinel result. Receives the word read at the retained address plus +0x04. |
| Indirect site B | Reads its destination from a separate global function-pointer slot after the final predicate passes. No explicit arguments. |

“A” and “B” identify transfer sites, not resolved function identities. Distinct slots do not establish distinct destination values.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless stated otherwise, all accessed storage is valid and dependencies return normally.

1. **Context replacement.**  
   The outer object initially identifies context A. F-494f99f352ec changes that field to context B. Subsequent explicit context accesses and selector calls use B.

2. **Independent selector results.**  
   The first selector returns 3 and the second returns 7. The ordered writes affect first-table record 3. Both second-table base observations use displacement 7 × 24, but they need not identify the same record as each other.

3. **Two different second-table observations.**  
   The first read of the second-table global produces base L; the second produces base R. The branch byte comes from R + 7 × 24 + 0x11. If nonzero, repeated byte arguments come from L + 7 × 24, not necessarily from the branch record. If zero, the pointer chain starts at R + 7 × 24 + 0x04.

4. **Changed context field.**  
   The first +0x34 read produces X. An intervening operation changes the field to Y. F-6518403de3d0 receives X, while F-ddf4a1f94e38 receives Y.

5. **Zero path without indirect transfer.**  
   The branch byte is zero. F-1333331b2da1 runs, then the pointer chain is read. F-9813035c7ce7 returns zero. Neither indirect site is reached.

6. **Late destination change at site A.**  
   F-9813035c7ce7 changes site A’s global destination slot and returns 0xFFFFFFFF. The transfer uses the destination read from that slot afterward. Its explicit argument comes from the retained pointer-chain value interpreted as an address, plus +0x04.

7. **Loop observes changed byte, not changed base.**  
   The first loop call receives byte 4, changes that same byte to 9, and returns nonzero. The next call receives 9. Changing only the second-table global does not redirect the next read.

8. **Guarded final dereference.**  
   After loop termination, a null context +0x00 pointer causes return without dereference. A nonnull pointer to a zero word also causes return, without reading site B’s destination slot.

9. **Late destination change at site B.**  
   The final loop invocation changes site B’s destination slot and returns zero. If the subsequent context pointer and pointed-to word are both nonzero, the transfer uses the destination read from that slot afterward.

10. **Unbounded repetition.**  
    If every F-0ea4820fe774 result is nonzero, repetition continues. The final context-pointer read and site B’s destination read are never reached.

## Uncertainties

The following remain **documented_only**:

- Application-level meaning of fields, constants, selectors, and predicates.
- Table capacities, valid selector ranges, and overflow behavior outside valid-address cases.
- Whether the two second-table base observations can differ in the actual execution environment.
- Volatility, atomicity, synchronization, concurrency, and reentrancy guarantees.
- Aliasing relationships and allocation lifetimes.
- Dependency-specific ownership, allocation, release, and global-preservation behavior.
- Exact calling conventions and argument-extension rules.
- Meaning of the four additional incoming values supplied to F-0ea4820fe774.
- Indirect destination identities, validity guarantees, ABI form, and return behavior.
- Relative evaluation order of site A’s argument-word read and destination-slot read.
- Fault behavior for invalid pointers, unsuitable alignment, or invalid indirect destinations.

## Evidence

Established static evidence consists of the represented access widths and offsets, constants, explicit call order, ordered writes, two independent selector calls, two separate second-table global reads with different uses, branch conditions, repeated byte reads, the 0xFFFFFFFF sentinel, the guarded final dereference, and site-local reads of two global indirect-destination slots.

“Record selection,” “context,” and “completion predicate” are structural interpretations. Application intent is not established.

No runtime tests, dependency executions, memory observations, or ABI validation have been performed.
