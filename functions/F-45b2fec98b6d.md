<!-- Trusted provenance: F-45b2fec98b6d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-45b2fec98b6d

## Purpose

F-45b2fec98b6d describes **ordered indirect-call occurrences with unresolved target identities**, selected result transformations, and two explicit single-byte stores. The supplied static representation does not establish an application-level purpose.

Its visible responsibilities are to:

- Execute a fixed sequence of indirect calls.
- Pass selected results onward after shifting them left by two bits.
- At two points, use one result both to derive a byte-store address and to form two equal, fourteen-bit-shifted arguments.
- Forward one four-byte result unchanged to one argument position, and another unchanged to two argument positions.
- Continue to a boundary marked as undecodable, without a demonstrated normal return.

Call occurrences are separate events, not necessarily separate destinations or implementations. In particular, the first destination depends on unconstrained incoming control-address state and may equal a later destination.

This contract distinguishes **established static evidence**, **inferred intent**, and **documented_only unknowns**. Established evidence concerns the supplied representation; decoding warnings limit its authority as a description of executable behavior. No runtime tests have been performed.

## Interface

### Explicit inputs and outputs

The represented entry point declares no explicit parameters and no returned value. This does not establish independence from incoming machine state, nor does it prove a valid no-argument calling convention.

No caller-provided buffer, length, aggregate record, status-output address, or ownership-bearing handle is identified.

The body does not establish a successful return value or a normal-return postcondition. Its terminal boundary is unresolved.

### Implicit inputs

Two incoming values participate directly in the represented behavior:

| Input | Represented use | Established limitations |
|---|---|---|
| Incoming control-address value | Adding 4,094 forms the first indirect-call destination | No validation, range restriction, or distinctness from later destinations is shown |
| Incoming byte value | Supplies the data for both explicit byte stores | No initialization within the body is shown; physical preservation across calls is unresolved |

Selected dependency results supply the remaining explicit dataflow inputs: integer-like values for shifts and address formation, and uninterpreted four-byte values for unchanged forwarding.

These are represented data relationships. They do not independently establish which physical registers or storage locations carry the values, or how dependencies preserve them.

### Widths and arithmetic

The representation distinguishes:

- Single-byte values and single-byte stores.
- Four-byte uninterpreted values used by two forwarding transitions.
- Integer-typed results used by shifts.
- Address expressions used for indirect calls and stores.

Under an eight-bit-byte model, the first two categories correspond to eight-bit and 32-bit data. The supplied material does not independently establish the integer width, pointer width, complete target data model, or signed-shift execution rules.

The established shift counts are:

- Two bits for ordinary scaled-result forwarding.
- Three bits for byte-store address formation.
- Fourteen bits for duplicated arguments associated with each store.

For suitable nonnegative values without overflow, these shifts correspond to multiplication by 4, 8, and 16,384, respectively. That arithmetic interpretation is not a promise of wraparound behavior for large or negative results.

The four-byte forwarding operations preserve the represented value without a visible shift, narrowing operation, dereference, or interpretation. This does not establish an external ABI representation.

### Preconditions for continued execution

Progress through the represented sequence requires:

- Reached call destinations to support executable transfers under the actual execution model.
- Dependencies to return control sufficiently normally for subsequent operations to occur.
- Each direct store destination to permit a single-byte write.
- Arithmetic and conversion to addresses to support the represented operations.

These are environmental requirements, not validations performed by F-45b2fec98b6d. No corresponding guard, error result, or recovery path is visible.

## Behavior

### Occurrence identifiers and ordering

**F-U001 through F-U135** identify the 135 indirect-call occurrences in their represented order. They are opaque occurrence references, not resolved dependency identities. Different identifiers may resolve to the same destination or implementation.

All sequencing below is conditional on preceding operations completing. The supplied body shows no conditional branch or loop. Repeated shapes describe a fixed call sequence, not data-dependent iteration.

A call described as having “no explicit arguments shown” may still consume implicit machine state or shared state. Its return value being unused does not make the call unobservable.

### Scaled-result forwarding

A scaled-result group contains preliminary calls followed by a producer and a recipient. The recipient receives one explicitly shown argument: its producer’s result shifted left by two bits.

The first four groups occur before the first byte store:

| Group | Preliminary occurrences, no explicit arguments shown | Producer | Recipient |
|---|---|---|---|
| 1 | F-U001–F-U004 | F-U005 | F-U006 |
| 2 | F-U007–F-U009 | F-U010 | F-U011 |
| 3 | F-U012–F-U014 | F-U015 | F-U016 |
| 4 | F-U017–F-U019 | F-U020 | F-U021 |

Within each row, preliminary occurrences execute in numerical order, then the producer, then the recipient. Rows execute in order.

The enclosing body does not explicitly consume the preliminary calls’ results. Their effects may nevertheless influence later execution. No accumulation of results across groups is visible.

### First byte-store transition

F-U022 through F-U024 precede producer F-U025. Its result participates in two subsequent effects:

- The incoming byte is stored at the address formed by shifting that result left by three bits.
- F-U026 is invoked with two explicitly shown arguments, each formed by shifting the same result left by fourteen bits.

The store occurs after F-U025 returns and before F-U026 is called. There is no intervening call between the producer and these uses.

The destination is a result-derived address, not an established field or offset within a known allocation. No bounds, permission, ownership, or null-address check precedes the store.

The two argument values are equal at the represented call boundary. Their equality does not establish their meaning or how F-U026 uses them.

### First unchanged-result transition

After F-U026, F-U027 through F-U029 execute before producer F-U030.

F-U031 receives the four-byte result of F-U030 as its one explicitly shown argument. The enclosing body does not visibly transform or dereference that result.

### Middle scaled-result groups

The following nine groups use the same two-bit forwarding relationship:

| Group | Preliminary occurrences, no explicit arguments shown | Producer | Recipient |
|---|---|---|---|
| 5 | F-U032–F-U034 | F-U035 | F-U036 |
| 6 | F-U037–F-U039 | F-U040 | F-U041 |
| 7 | F-U042–F-U044 | F-U045 | F-U046 |
| 8 | F-U047–F-U049 | F-U050 | F-U051 |
| 9 | F-U052–F-U054 | F-U055 | F-U056 |
| 10 | F-U057–F-U059 | F-U060 | F-U061 |
| 11 | F-U062–F-U064 | F-U065 | F-U066 |
| 12 | F-U067–F-U069 | F-U070 | F-U071 |
| 13 | F-U072–F-U074 | F-U075 | F-U076 |

Each recipient uses its own row’s producer result. These results are not visibly combined with the earlier store-related result or with other groups’ results.

### Second byte-store transition

F-U077 through F-U079 precede producer F-U080.

After that producer returns:

- The represented incoming byte is stored at the address obtained by shifting its result left by three bits.
- F-U081 receives two equal explicitly shown arguments, each obtained by shifting that same result left by fourteen bits.

The store precedes F-U081 and follows every occurrence through F-U080.

The two store destinations are not established to be different. The representation uses the same incoming byte value for both writes, but does not resolve its physical preservation across dependencies.

### Second unchanged-result transition

F-U082 through F-U084 precede producer F-U085.

F-U086 receives the four-byte result of F-U085 in both explicitly shown argument positions, without a visible transformation. This differs from F-U031, which receives only one explicitly shown argument.

No ordering between the two argument evaluations is established or needed by the visible dataflow: both refer to the same produced value, without an intervening represented effect.

### Final scaled-result groups and tail

Nine further groups follow:

| Group | Preliminary occurrences, no explicit arguments shown | Producer | Recipient |
|---|---|---|---|
| 14 | F-U087–F-U089 | F-U090 | F-U091 |
| 15 | F-U092–F-U094 | F-U095 | F-U096 |
| 16 | F-U097–F-U099 | F-U100 | F-U101 |
| 17 | F-U102–F-U104 | F-U105 | F-U106 |
| 18 | F-U107–F-U109 | F-U110 | F-U111 |
| 19 | F-U112–F-U114 | F-U115 | F-U116 |
| 20 | F-U117–F-U119 | F-U120 | F-U121 |
| 21 | F-U122–F-U124 | F-U125 | F-U126 |
| 22 | F-U127–F-U129 | F-U130 | F-U131 |

After F-U131, **four** additional indirect-call occurrences execute: F-U132, F-U133, F-U134, and F-U135. Each shows no explicit arguments.

The supplied representation then reaches an invalid-instruction boundary. That boundary does not establish a normal return, intentional shutdown, particular exception, or portable halt behavior.

## Side effects

### Direct memory effects and layout

Exactly two explicit data-memory stores appear outside dependency bodies. Each writes one byte.

No explicit data-memory load, bulk copy, allocation, release, or traversal appears in the enclosing body. This inventory does not include operations performed by dependencies.

For each store:

- The address depends on the immediately preceding designated producer.
- The stored data is the represented incoming byte.
- No read-modify-write operation is shown.
- No surrounding object layout or accessible extent is established.
- No synchronization or atomicity guarantee is established.

Under ordinary non-overflowing arithmetic, shifting an integer left by three produces a multiple of eight. This does not prove an eight-byte object, array stride, field alignment requirement, or allocation size.

If the stores address the same byte, the second direct write occurs later. Dependencies between or after the stores may read, overwrite, release, or otherwise affect the relevant memory. Consequently, the final memory state cannot be determined from the two direct stores alone.

### Dependency effects and ownership

Every reached dependency call is observable even when its result is unused. No dependency is established to be pure, to preserve globals, or to preserve relevant memory.

There is no established ownership transfer:

- A shifted argument is not proven to identify an allocation.
- A four-byte forwarded value is not proven to be a handle or pointer.
- Receiving an argument does not establish acquisition, retention, or release.
- No explicit allocation or deallocation by this body establishes a lifetime boundary.

The contract does not assume unknown dependencies preserve incoming machine-state values. The representation’s continued use of an incoming byte is evidence of represented dataflow, not proof of a compatible executable preservation mechanism.

### Errors and partial execution

No explicit error check, returned error status, retry, cleanup, rollback, or exception handler is visible.

Producer results are consumed without visible failure discrimination. A value conventionally used as a sentinel is not specially handled by the enclosing body.

If a call fails to return, or an arithmetic or memory operation cannot complete, later operations are not guaranteed to occur. Already completed calls and stores have no visible compensating action.

## Dependencies

F-U001 has a destination derived from incoming control-address state plus 4,094. Its destination may coincide with a destination reached later.

F-U002 through F-U135 use represented local anchors or fixed address values with the same displacement. These expressions do not resolve target identity, ABI, purpose, or executable-memory layout.

The explicit argument relationships comprise:

| Relationship | Number of occurrences |
|---|---:|
| One argument containing a producer result shifted left by two | 22 |
| Two equal arguments containing a producer result shifted left by fourteen | 2 |
| One unchanged four-byte argument | 1 |
| Two equal unchanged four-byte arguments | 1 |

All other call occurrences show no explicit arguments.

Occurrence numbering imposes order, not distinctness. Even if two destinations are equal, both invocations remain required events in the represented sequence. They cannot be merged or omitted merely because target identity or argument values coincide.

## Acceptance examples

These are conditional review examples, not executed tests. They assume the supplied representation describes the reached behavior and that all preceding operations complete.

### Small scaled result

If F-U005 returns 3 under an integer model supporting the shift, F-U006 receives 12 in its explicitly shown argument position.

F-U001 through F-U005 precede that invocation. Forwarding 3 unchanged would violate the represented relationship.

### Store followed by duplicated arguments

If F-U025 returns 2, the incoming byte is 165, and the address model permits the operations:

- One byte containing 165 is written at address 16.
- The write precedes F-U026.
- Both explicitly shown arguments to F-U026 are 32,768.

Address 16 must be writable for execution to progress through the store. The function does not establish that condition.

### Zero producer result

If F-U080 returns zero under a compatible model, the second store targets address zero, and both arguments to F-U081 are zero.

There is no visible null-address bypass. If that store cannot complete, invocation of F-U081 is not guaranteed.

### Unchanged four-byte value

Under an eight-bit-byte model, if F-U030 produces the four-byte pattern 0x89ABCDEF, F-U031 receives that same pattern in its one explicitly shown argument position.

If F-U085 produces that pattern, F-U086 receives it in both shown positions. No signed interpretation or pointer access is required by these forwarding relationships.

### Conditional truncation

If the execution model uses 32-bit left shifts that discard high bits, shifting 0x40000001 left by two produces 0x00000004.

That result is conditional on the stated model. It is not an unconditional requirement for an unresolved signed-integer execution model.

### Aliased call destinations

If incoming control-address state makes F-U001 resolve to the same destination as F-U002, the represented order still contains two invocations: F-U001 followed by F-U002.

Destination equality does not authorize deduplication. The first invocation may change state observed by the second.

### Interrupted execution and terminal tail

If F-U040 does not return, F-U041 and subsequent operations are not reached through this sequence. Earlier effects, including the first direct store if completed, have no visible rollback.

If execution instead progresses beyond F-U131, all four tail occurrences—F-U132 through F-U135—precede the undecodable boundary. Stopping after F-U134 would omit a represented call.

## Uncertainties

The following remain **documented_only**:

- Actual dependency identities and aliasing among call destinations.
- Validity of the recovered entry boundary and apparent parameterless interface.
- Calling convention, implicit arguments, physical value preservation, and execution-state address conventions.
- Integer width, pointer width, signed-shift semantics, overflow behavior, and integer-to-address conversion.
- Meaning and admissible range of the incoming control-address value.
- Origin and physical preservation of the incoming byte.
- Accessibility, ownership, volatility, and lifetime of store destinations.
- Dependency allocation, release, retention, global-state changes, and control-flow effects.
- Runtime behavior at the undecodable terminal boundary.

**Inferred intent:** repetition may reflect a generated or encoded operation sequence. No application purpose—such as initialization, serialization, dispatch, or device access—is established. That interpretation is not an acceptance requirement.

## Evidence

The supplied static body supports this structural inventory:

- **135 indirect-call occurrences**, including four after the final scaled-result recipient.
- Twenty-two two-bit-shifted result-forwarding relationships.
- Two single-byte stores at addresses derived by three-bit shifts.
- Two calls receiving duplicated fourteen-bit-shifted results.
- Two unchanged four-byte forwarding transitions: one with one shown argument, one with two.
- Fixed visible ordering without a represented conditional branch or loop.
- A first call destination dependent on incoming state, without a distinctness constraint.
- Decoding warnings and an undecodable terminal boundary.

These findings derive solely from inspection of the supplied static representation. Dependency resolution, executable verification, and runtime testing have not been performed.
