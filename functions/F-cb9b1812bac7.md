<!-- Trusted provenance: F-cb9b1812bac7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cb9b1812bac7

## Purpose

F-cb9b1812bac7 conditionally modifies a record, obtains an address-like result through an opaque dependency, processes an indexed destination, and returns a downstream status.

These are descriptive roles, not established domain semantics. The supplied static evidence establishes memory accesses, dependency calls, arithmetic, and return behavior; it does not establish that the dependencies allocate a container, copy a record, commit changes, or release ownership.

An initial unsigned index outside 0–15 causes an immediate zero return without dependency calls or direct writes. For accepted indices, zero can also mean that acquisition returned zero or that downstream status evaluation returned zero. The return value alone does not distinguish these cases.

## Interface

The function accepts four arguments and returns a 32-bit integer.

| Argument | Representation | Established use |
|---|---|---|
| Context | Address of storage accessed as 32-bit words | Supplies the index, record address, and acquisition arguments; receives the acquisition result. |
| Flag selector | 32-bit integer | Any nonzero value enables a record-byte read-modify-write. |
| Conversion selector | 32-bit integer | Any nonzero value enables a working-storage dependency call followed by two record-halfword stores. |
| Value | Opaque 32-bit value | Initializes private working storage on the accepted-index path. |

### Context layout

| Byte offset | Access width | Role |
|---|---:|---|
| `+0` | 32 bits | First argument to F-04b7df4e1a09. |
| `+4` | 32 bits | Record address used for optional edits and subsequently passed to F-4a58994bf609. |
| `+12` | 32 bits | Second argument to F-04b7df4e1a09. |
| `+16` | 32 bits | Initial unsigned index, retained for the gate and destination calculation. |
| `+24` | 32 bits | Acquisition result, written after F-04b7df4e1a09 and freshly read for F-c7bbf7146083. |

No direct use of other context fields is established.

### Record layout

| Byte offset | Access width | Conditional direct effect |
|---|---:|---|
| `+11` | 8 bits | Read the byte, set bit `0x20`, and write the resulting byte. |
| `+22` | 16 bits | Store the two-byte portion selected at byte offset 2 within the working storage, using its contents at this access. |
| `+24` | 16 bits | Store the numeric low 16 bits of the working value read for this later store. |

The first halfword source is identified by its **storage position**, not unconditionally by numeric significance. With a conventional little-endian interpretation, bytes 2–3 represent bits 31–16. With a conventional big-endian interpretation, those bytes represent bits 15–0. Target endianness is not supplied.

The second halfword source is different: narrowing the working value establishes the numeric low 16 bits, independently of their position in memory.

The context field at `+24` and record field at `+24` have different layout roles. Their physical storage is not guaranteed to be disjoint. Neither is the record guaranteed to be disjoint from the address-exposed working storage.

### Preconditions and validity

For a rejected index, only the initial 32-bit context read at `+16` is exercised.

For an accepted index, each memory access must be valid when it occurs:

- Context fields must support their actual reads and writes.
- A nonzero flag selector requires a valid byte access at the record address then used, plus 11.
- A nonzero conversion selector requires valid halfword destinations for the two stores.
- Nonzero acquisition results must satisfy the downstream dependencies’ actual requirements.
- Any aliasing or dependency effects must leave subsequent accesses valid.

The visible body performs no explicit pointer validation, alignment repair, address-boundary check, or exception translation. These are validity requirements, not implemented checks.

## Behavior

### Initial index gate

The first visible memory operation reads the 32-bit context word at `+16` as an unsigned index.

If the index is at least 16, the function returns zero. It does not initialize the working storage, call a dependency, edit the record, or write the context result field.

Unsigned interpretation matters: `0xffffffff` is rejected rather than treated as an accepted negative index.

If the index is 0–15, its initial value is retained. Later changes to the context index do not change the destination displacement.

### Working storage and first dependency

On the accepted-index path, the fourth argument initializes private 32-bit working storage. F-4c6d30ac2311 is then called without explicit arguments.

No return value from that dependency is consumed. Its placement before the edits is established; a locking, interrupt-control, or transaction role is not.

### Optional byte edit

After F-4c6d30ac2311 returns, a nonzero flag selector enables the byte operation.

The function obtains the record address through the context, reads the record byte at `+11`, combines that byte with `0x20` using bitwise OR, and writes the result to the corresponding byte destination.

This is a read-modify-write. Bits other than `0x20` retain the values observed by the read. The operation is not an unconditional assignment of `0x20`, and atomicity is not established.

A zero flag selector skips this direct edit. It does not guarantee that dependencies leave that byte unchanged.

### Optional working-value conversion and ordered halfword stores

A nonzero conversion selector causes F-193eb1bbf5fe to receive the address of the private 32-bit working storage. The dependency’s return value is not consumed; the subsequent stores use the storage contents.

After the dependency returns, two separate stores occur in order:

1. The first reads the two-byte portion at working-storage byte offset 2 and stores it into the record halfword at `+22`.
2. The second reads the working value for a narrowing conversion and stores its numeric low 16 bits into the record halfword at `+24`.

The source of each store is determined at that store’s access. There is no established single snapshot supplying both stores.

This distinction matters when storage aliases. F-193eb1bbf5fe has received the working-storage address and may have effects that make a record destination overlap that storage. If the first store overlaps the working storage, it can change the value read by the second store. A stable record address does not exclude this possibility.

The first store can also overlap context storage, including the record-address field. The later store obtains its record address through the context rather than relying on an established immutable record pointer. Consequently, both the second source value and its destination can be affected by earlier aliasing writes.

The cross-statement order is established. A more detailed relative order between source reads and destination-address reads within either individual store is not established by the supplied representation.

A zero conversion selector skips F-193eb1bbf5fe and both halfword stores.

### Second no-argument dependency and acquisition

After the optional edits, F-cc60a532ba6c is called without explicit arguments. Thus, both no-argument dependencies run on every accepted-index path, even when both selectors are zero.

Next, F-04b7df4e1a09 receives the 32-bit context values at `+0` and `+12`. These reads occur after the preceding calls and edits. Their relative argument-evaluation order is not established.

The returned 32-bit result is written to context offset `+24`, including when it is zero. The function also retains this acquisition result for the subsequent branch and downstream uses.

If acquisition returns zero, the function returns zero without calling F-4a58994bf609, F-c7bbf7146083, or F-2b1ba6d8de9d. Earlier writes are not rolled back by the visible body.

### Destination processing and final return

For a nonzero acquisition result, the first argument to F-4a58994bf609 is the retained acquisition result plus a fixed displacement of 24 bytes and a displacement of 32 bytes per retained index.

The combined displacement is 24–504 bytes:

- Index 0 selects displacement 24.
- Index 15 selects displacement 504.

The index multiplication and combined displacement fit within 32 bits. Adding that displacement to the acquisition result may cross a representable or valid address boundary. No explicit guard is present; portable overflow behavior and valid wraparound addresses are not established.

F-4a58994bf609 receives this calculated destination and the record address read through the context at that stage. Its return value is not consumed. A 32-byte stride does not establish its access extent or prove that it copies 32 bytes.

After this call, the function freshly reads context offset `+24` and passes that value to F-c7bbf7146083. This argument need not equal the retained acquisition result: the preceding dependency or an aliasing effect may have changed the field.

The resulting 32-bit status is retained. F-2b1ba6d8de9d then receives:

- The original nonzero acquisition result.
- An integer indicator equal to 1 if the retained status is zero, otherwise 0.

The function returns the retained status unchanged after that call. Any return value from F-2b1ba6d8de9d is ignored. A negative nonzero status therefore produces indicator 0 and is returned as that same negative status.

## Side effects

Established direct writes are:

- Initialization of private working storage on accepted indices.
- The optional record-byte write at `+11`.
- The optional, ordered record-halfword writes at `+22` and `+24`.
- The acquisition-result write to context `+24`.

These are logical destinations, not guaranteed disjoint physical regions. A record write can also modify context fields or working storage if the addresses overlap. In particular, the first halfword store can affect what the second store reads.

All dependencies may have additional effects not described by the supplied body. No preservation of globals, context contents, record contents, or previously exposed storage is assumed.

There is no visible rollback. Acquisition failure does not reverse earlier edits. The context result field remains written unless another effect changes it.

No allocation, ownership transfer, resource release, or object-lifetime transition is directly established. An address-like acquisition result and a later call receiving that result are insufficient to prove such behavior.

The working-storage address is exposed to F-193eb1bbf5fe. The storage exists during this invocation. No evidence authorizes its use after the containing invocation returns or establishes a callee obligation to retain, discard, or release that address.

## Dependencies

| Opaque function ID | Established call boundary |
|---|---|
| F-4c6d30ac2311 | Called after working-storage initialization on every accepted-index path, before optional edits. No explicit arguments or consumed return value. |
| F-193eb1bbf5fe | Called only for a nonzero conversion selector. Receives the working-storage address. Subsequent stores read that storage separately. |
| F-cc60a532ba6c | Called after optional edits on every accepted-index path. No explicit arguments or consumed return value. |
| F-04b7df4e1a09 | Receives context words at `+0` and `+12`. Its 32-bit result is stored in the context and gates downstream processing. |
| F-4a58994bf609 | Receives the calculated destination and the then-read record address following nonzero acquisition. Operation and memory extent are unknown. |
| F-c7bbf7146083 | Receives a fresh read of context `+24`. Its 32-bit result becomes the retained final status. |
| F-2b1ba6d8de9d | Receives the original acquisition result and the zero-status indicator before the retained status is returned. |

These contracts describe visible invocation behavior only. They impose no undocumented ownership or global-preservation guarantees.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless stated otherwise, accesses are valid, dependencies return normally, and no additional dependency or concurrent effect changes the values relevant to the example.

1. **Rejected index.** An initial index of 16 or `0xffffffff` produces zero, no dependency calls, and no direct writes.

2. **Accepted index boundaries.** With nonzero acquisition result A, index 0 supplies A plus 24 bytes to F-4a58994bf609; index 15 supplies A plus 504 bytes, conditional on valid address arithmetic.

3. **Flag preservation.** With a nonzero flag selector, an observed record byte of `0x85` produces a byte write of `0xa5`. An observed `0xa5` still produces a write of `0xa5`.

4. **Selector interpretation.** Selector values 1 and −1 both enable their respective operations. The tests are against zero, not against a particular selector bit.

5. **Halfword values without aliasing.** Suppose F-193eb1bbf5fe leaves the working value numerically equal to `0x1234abcd`; the record address remains stable; neither record halfword destination overlaps the working storage or relevant context fields; and nothing changes the working storage between its two reads. On a conventional little-endian target, the stores are `0x1234` at record `+22`, then `0xabcd` at record `+24`. On a conventional big-endian target, they are `0xabcd`, then `0xabcd`. Without an established endianness, the first store is specified only as the selected two-byte portion at working offset 2; the second is the numeric low 16 bits.

6. **First store changes the second source.** Assume a conventional little-endian target, a post-conversion working value of `0x1234abcd`, and a stable record address whose `+22` destination is the first two bytes of the working storage. Also assume no other relevant overlap or interference. The first store reads `0x1234` from working bytes 2–3 and replaces working bytes 0–1 with that value. The working value consequently becomes `0x12341234`. The second store then reads and stores low half `0x1234`, not the original `0xabcd`. Stable record addressing alone is therefore insufficient for example 5.

7. **Acquisition failure after edits.** If edits are requested and acquisition returns zero, context `+24` receives zero, the final three dependencies are skipped, and the return is zero. No visible rollback restores the edited fields.

8. **Zero downstream status.** With nonzero acquisition result A and status zero, F-2b1ba6d8de9d receives A and 1. The function then returns zero.

9. **Negative downstream status.** With status −7, F-2b1ba6d8de9d receives the original acquisition result and 0. The function returns −7.

10. **Fresh result-field read.** If acquisition returns A and F-4a58994bf609 changes context `+24` to B, F-c7bbf7146083 receives B. F-2b1ba6d8de9d still receives A.

11. **Retained index.** If a dependency changes the context index after the initial accepted read, the destination displacement still uses the initial index.

## Uncertainties

The following remain **documented_only**:

- Domain meanings and complete layouts of the context, record, and destination.
- Target endianness, alignment requirements, and address-space validity rules.
- The transformation and other effects of F-193eb1bbf5fe.
- Whether dependencies create aliases involving the exposed working storage.
- Relative operand-evaluation order within individual expressions where the supplied representation does not establish it.
- F-4a58994bf609’s operation and access extent.
- Whether acquisition allocates, locates, reserves, or otherwise obtains anything.
- Whether the final dependency releases resources or changes ownership.
- Status meanings beyond their visible zero/nonzero effects.
- Concurrency, atomicity, and dependency effects on globals and reachable memory.
- Behavior following invalid memory access or a dependency that does not return normally.
- Defined arithmetic behavior at address or integer boundaries.

Preparation, transfer, and finalization are inferred intent only. They do not add behavioral or ownership guarantees.

## Evidence

This contract is based on the supplied static function body and opaque dependency mapping.

Established static evidence includes the unsigned index gate; access widths and offsets; selector conditions; dependency call order; exposure of working storage; the first halfword’s byte-offset selection; the second halfword’s numeric narrowing; the separation and order of those stores; the acquisition-result write and reread; destination displacement; status indicator; and return paths.

Aliasing examples are conditional consequences of those accesses, not claims that such aliases occur in an observed execution. Endianness-specific numeric examples likewise depend on explicitly stated target assumptions.

No dependency implementations, external ownership contracts, runtime traces, or executable observations were supplied. No runtime tests have been performed.
