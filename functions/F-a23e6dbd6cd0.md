<!-- Trusted provenance: F-a23e6dbd6cd0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a23e6dbd6cd0

## Purpose

F-a23e6dbd6cd0 coordinates a byte-sized state change with an externally obtained object and two indirect dispatches through the input object.

On its ordinary continuation path, it:

- Derives a pending state byte from two reads of a related object.
- Obtains another object through F-fdd8589eca2d.
- Writes the value one to a byte within that returned object.
- Writes the pending state byte through the input object’s then-current related-object pointer.
- Performs two ordered indirect invocations using the dispatch slot at byte offset `0x58`.

The state calculation resembles a Boolean toggle with normalization of values greater than one. That interpretation is **inferred intent**, not an established description of the surrounding subsystem. The source does not establish whether the bytes represent selection, visibility, activation, or another property.

## Interface

### Input

The function receives one object address, referred to here as the **input object**. The supplied representation treats this address as pointing to an array of 32-bit words.

The following layout is used:

| Location | Access | Meaning established by this function |
|---|---|---|
| Input object, byte offset `0x000` | Address-sized read | Address of a dispatch table |
| Input object, byte offset `0x378` | Address-sized read | Address of a related object |
| Related object, byte offset `0x08F` | Unsigned 8-bit reads; 8-bit write | State byte used to derive and publish the pending state |
| Object returned by F-fdd8589eca2d, byte offset `0x0CD` | 8-bit write | Byte set to one |
| Dispatch table, byte offset `0x058` | Callable-address read | Target used for each indirect invocation |

The `0x378` offset follows from word index `0xDE` with four-byte words: `0xDE × 4 = 0x378`. These offsets describe the supplied 32-bit representation. They do not establish a portable layout for other pointer widths.

### Output

There is no explicit return value. Observable results consist of memory writes and dependency invocations.

A normal return is possible only if the dependency calls and both indirect invocations complete normally. If the zero-result failure path is taken, normal return is not established.

### Preconditions

For the ordinary path to have defined behavior:

- The input object must remain accessible wherever this function dereferences it.
- Its related-object address must identify readable storage for both initial byte reads.
- At the later write, the related-object address then stored in the input object must identify writable storage at offset `0x08F`.
- A nonzero result from F-fdd8589eca2d must identify writable storage at offset `0x0CD`.
- Each dispatch-table address and slot used for an invocation must identify a valid callable target under the applicable calling convention.
- Memory and object lifetimes must remain valid across dependency calls.

No local null checks, bounds checks, object-type checks, or synchronization establish these conditions. A nonzero returned address is not independently validated.

## Behavior

### Pending-state derivation

Before calling F-fdd8589eca2d, the function derives a pending byte from the related object’s state.

There are two distinct state-byte reads:

1. The first supplies an unsigned byte used to calculate one minus that value.
2. The second supplies an unsigned byte compared with one. If this second value is greater than one, the pending byte is replaced with zero.

The pending result occupies eight bits. For a first-read value \(a\) and a second-read value \(b\), its value is:

- Zero if \(b > 1\).
- Otherwise, the low eight bits of \(1-a\).

The subtraction is not an unrestricted-width state value retained until publication; the pending storage is byte-sized. If the reads differ, a first-read value greater than one can therefore produce a wrapped byte when the second read is zero or one. For example, first-read value two and second-read value zero produce byte value `255`.

When the state is stable across both reads, the result is:

| State read both times | Pending state |
|---|---:|
| `0` | `1` |
| `1` | `0` |
| `2` through `255` | `0` |

The two reads must not be treated as a single atomic snapshot. Each accesses the related object through the input object’s pointer field. The function provides no synchronization against changes to either that field or the byte.

The exact numeric interpretation of a wrapped byte as a signed character in another consumer is outside this function’s contract. The published storage is eight bits.

### Obtaining the secondary object

After deriving the pending state, the function invokes F-fdd8589eca2d. The supplied call has no explicit arguments.

Its result is used as a 32-bit address-like value:

- A zero result selects the failure dependency F-a817d8d58c10.
- A nonzero result selects the ordinary write path.

The failure dependency is marked as not returning in the supplied representation. Consequently, the represented zero-result path does not reach either local byte write or either indirect invocation.

The implementation of F-a817d8d58c10 is unavailable. The function does not establish whether it terminates execution, transfers control elsewhere, raises an exception, or performs another nonlocal action. No ordinary recovery behavior should be specified.

If that dependency were to return despite its annotation, the subsequent code contains no separate recovery branch. Such behavior is outside the established normal contract.

### Ordered byte writes

After a nonzero result from F-fdd8589eca2d, the function performs two explicit writes in this order:

1. It writes the single byte `1` at offset `0x0CD` of the returned object.
2. It writes the previously computed pending byte at offset `0x08F` of the related object reached through the input object at that time.

The destination pointer for the second write is read after F-fdd8589eca2d returns. The pending value, however, was calculated before that call. Therefore, a dependency-induced pointer change can cause a value derived from one related object to be written into another.

The function does not re-read the state to recompute the pending value after obtaining the secondary object.

No non-aliasing guarantee is established. If the two byte destinations coincide, the second write determines the byte left by these explicit writes. For example, an initial write of one can be overwritten immediately by a pending zero. Indirect invocations may subsequently change that byte again.

### Ordered indirect invocations

After both byte writes, the function performs two indirect invocations, in sequence, through dispatch-table offset `0x58`.

For this description, three externally stored 32-bit values are designated **K1**, **K2**, and **K3**. These are opaque labels, not asserted meanings or known numeric constants.

Before the first invocation, the function captures K1 for later use. The first represented invocation includes:

- The input object.
- A value read from K1 for that invocation.
- A value read from K2.
- A callable-address value read through the dispatch slot.
- Four additional machine-state values whose application-level meanings are not established.

The fourth represented argument is obtained from the same dispatch-slot expression as the call target. This establishes neither a self-reference convention nor a requirement that it equal the eventual target if the underlying storage changes between reads.

After the first invocation returns, the second invocation includes:

- The input object.
- The previously captured K1 value.
- A value read from K3.

The second invocation resolves the dispatch-table slot again. It is not required to use the same target as the first invocation. The first invocation may change the input object’s dispatch-table address, the slot contents, or other relevant state.

The captured K1 value survives as the second invocation’s represented argument even if the first invocation changes K1’s storage. Conversely, the function does not establish preservation of K2, K3, the input object, or global state across either invocation.

The exact ordering of separate operand reads within an invocation is not fully established by the supplied representation. The contract therefore does not impose a total ordering among all dispatch-slot, table-pointer, and argument-value reads associated with one call.

## Side effects

The function itself explicitly writes two bytes on the nonzero-result path. It does not explicitly allocate, free, retain, or release an object.

That observation does **not** establish ownership semantics:

- F-fdd8589eca2d might return borrowed, shared, newly created, or otherwise managed storage.
- F-a817d8d58c10 may perform arbitrary failure-related effects.
- Either indirect target may mutate memory, modify global values, change dispatch targets, or affect object lifetimes.

No rollback is present. If execution stops after the first byte write, the second write and indirect invocations have not thereby occurred. If the first indirect invocation fails to return, the second invocation does not occur.

Both explicit byte writes precede the first indirect invocation in the represented control flow. Either invocation can subsequently overwrite their results; the function does not guarantee the final memory contents at return independently of those callees.

## Dependencies

| Opaque dependency | Established interaction | Unestablished properties |
|---|---|---|
| F-fdd8589eca2d | Called once after pending-state derivation, with no explicit arguments; its result is compared with zero and otherwise used as an address | Allocation, ownership, lifetime, global mutations, failure mechanism |
| F-a817d8d58c10 | Called with no explicit arguments when the preceding result is zero; marked non-returning | Concrete termination or transfer mechanism, memory effects |
| Unresolved indirect targets | Reached twice through dispatch-table offset `0x58`, subject to normal return from preceding stages | Identities, signatures, ownership effects, state preservation, return behavior |

No opaque function IDs are available for the indirect targets. Assigning them invented identities or assuming both calls reach one dependency would exceed the evidence.

The first indirect invocation’s extended argument list is not sufficient to establish a reliable public signature. In particular, the four additional machine-state values must not be assigned invented domain meanings.

## Acceptance examples

These examples are conditional static acceptance criteria, not reports of executed tests. Unless otherwise stated, assume valid memory, a nonzero secondary-object result, no intervening changes to the relevant fields, and normally returning indirect targets.

### Stable zero state

Given a related-state byte of zero for both initial reads:

- The pending byte is one.
- The secondary-object byte at `0x0CD` is written to one first.
- The related-state byte at `0x08F` is then written to one.
- The first indirect invocation precedes the second.

### Stable one state

Given a related-state byte of one for both reads, the pending state and later related-state write are zero. The secondary-object byte is still written to one before that zero is published.

### Stable non-Boolean state

Given any stable initial byte from two through `255`, the pending byte is zero. The behavior does not preserve that non-Boolean value or map it to one.

### Different values across reads

If the first state-byte read observes two and the second observes zero, the pending byte is `255`.

If the first observes zero and the second observes two, the pending byte is zero because the second read’s comparison overrides the first calculation.

These cases distinguish the established two-read behavior from a single-read Boolean toggle.

### Related-object replacement during the dependency call

Suppose both initial reads use one related object, but F-fdd8589eca2d replaces the input object’s related-object pointer before returning a valid secondary object.

The later state write uses the replacement pointer. Its value remains the pending byte derived before the call.

### Aliased write destinations

If the returned object’s `0x0CD` byte and the current related object’s `0x08F` byte are the same location, that location receives one and then the pending byte. With a pending zero, the two explicit writes leave zero before indirect dispatch begins.

### Dispatch changes during the first invocation

If the first indirect target changes the dispatch-table address or slot and returns normally, the second invocation resolves the slot again and can reach a different target.

If the first invocation changes K1’s storage, the second invocation still receives the K1 value captured before the first invocation, subject to valid execution and the represented calling convention.

### Zero dependency result

If F-fdd8589eca2d returns zero, F-a817d8d58c10 is invoked. Under its supplied non-returning annotation, neither explicit byte write nor either indirect invocation follows.

## Uncertainties

The following items are **documented_only**:

- Domain meaning of either byte and the input object.
- Exact values and semantics of K1, K2, and K3.
- Ownership and lifetime rules for every referenced object.
- Whether the repeated reads can encounter concurrent or externally driven changes.
- Atomicity, synchronization, and memory-ordering guarantees.
- Concrete effects of the failure dependency.
- Reliable application-level signature of the first indirect invocation.
- Instruction-level ordering of operand reads within each indirect call.
- Validity of any execution in which an unknown callee invalidates subsequently accessed storage.

The toggle-like purpose is inferred intent. The offsets, byte-write order, branch condition, and represented call sequence are static observations. No assumption that unknown callees preserve globals or release—or do not release—memory is part of this contract.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping.

The representation supports the two initial unsigned-byte reads, byte-sized pending state, zero-result branch, ordered byte writes, capture of K1, and two ordered indirect invocations through slot `0x58`.

Its non-returning annotation supports treating the zero-result dependency as terminating that represented path, but does not identify the termination mechanism. Its incomplete indirect-control-flow recovery limits confidence in the exact calling convention and extended argument list.

No dependency implementations, runtime traces, or runtime tests were available. No runtime tests have been performed.
