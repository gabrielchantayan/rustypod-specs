<!-- Trusted provenance: F-5581ff5d1dec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5581ff5d1dec

## Purpose

F-5581ff5d1dec specifies an ordered sequence of memory writes and dependency calls driven by implicit entry state and ten represented 64-bit call results.

The supplied static representation establishes repeated halfword writes, word writes at derived addresses, repeated clearing of one global storage location, and a terminal bad-instruction marker. It does not establish an application-level purpose such as initialization, reset handling, device programming, or protocol processing.

This contract distinguishes:

- **Established static evidence:** widths, expressions, arguments, and operation ordering explicitly represented in the supplied material.
- **Inferred interpretation:** the use of 32-bit machine-address arithmetic to explain address calculations where the representation does not independently establish executable semantics.
- **documented_only:** unresolved behavior, including dependency effects, memory validity, and the actual terminal outcome.

All described execution sequences are conditional on preceding operations completing and dependencies returning. No runtime tests have been performed.

## Interface

### Entry and completion

The represented interface declares no explicit arguments and no returned value. It nevertheless consumes four values from implicit entry state:

| Contract label | Represented width | Use |
|---|---:|---|
| Entry address | 32 bits | Initial halfword destinations and payload |
| Context base | 32 bits | Base for two word writes at byte offset `0x14` |
| Entry word | 32 bits | Derived destinations, halfword payloads, and word payloads |
| Entry continuation | 32 bits | Basis of the initial indirect-call target |

These labels describe interface roles, not source identifiers.

Ten producer calls each expose a represented 64-bit result. For each result:

- The **low word** is bits 0–31.
- The **high word** is bits 32–63.
- High-word callback arguments undergo a represented conversion to signed 32-bit form. Their receiving interpretation is unknown.

There is no established normal return. The represented sequence ends at a bad-instruction marker.

### Widths, layout, and arithmetic

A halfword store writes 16 bits. A word store writes 32 bits. All address offsets are byte offsets.

The storage width of the global clear is unresolved. A literal zero assignment does not establish whether that store is 8, 16, 32, or another number of bits. The supplied material also warns that global storage overlaps smaller symbols at the same address; a disjoint global object layout must not be assumed.

Address calculations are described using the represented 32-bit values. A wrapping machine interpretation is useful for examples, but is not an independently established guarantee for signed-looking arithmetic, pointer conversions, or overflow. In particular, this qualification applies to doubling the entry address, adding field offsets, and interpreting the derived word destination.

Logical right shifts operate on unsigned bit patterns. The entry-word left shift used for later word payloads retains the low 32 bits. Endianness is not established.

### Preconditions and ownership

The visible sequence performs no address, alignment, range, or null validation. Reaching every stage requires all attempted stores to succeed and every preceding dependency to return.

No buffer sizes, allocation operations, releases, or ownership transfers are represented. Passing an address-like argument to a dependency does not establish ownership, lifetime, permission to dereference it, or responsibility for releasing it.

## Behavior

### Observable ordering model

The contract describes the direct writes and calls represented by this function. It does not expand dependency-internal behavior.

Repeated stores are separate represented events, even when their address and payload are identical. They must not be collapsed into one event when comparing the static trace. Whether actual execution produces distinct hardware transactions is unresolved.

No explicit destination-memory load or read-modify-write operation is represented. Payloads are derived from entry state and call results, not from the previous contents of their destinations. Dependencies may perform additional reads and writes.

### Entry phase

Before any dependency call, the represented write order is:

1. Two halfword stores at the entry address, both carrying its low 16 bits.
2. One halfword store carrying the same payload at twice the entry address.
3. One word store carrying the entry word at context base plus `0x14`.

The function then calls F-000000000001 at the target expression:

\[
\text{entry continuation} + \texttt{0x0FFE}.
\]

That call has no explicit arguments. Its result is not consumed by the represented sequence.

The first producer call follows. “No explicit arguments” describes the call expression only; it does not establish that a dependency has no implicit inputs.

### Producer calls and result stages

The ten producers occur in the following order. Each target expression is statically represented; dependency identity and behavior remain unresolved independently of these expressions.

| Result | Producer | Producer-call target expression | Pair form | Continuation token |
|---|---|---|---|---|
| 1 | F-000000000010 | `0x08910013 + 0x0FFE` | Wide | `0x08910015` |
| 2 | F-000000000011 | `0x0891007B + 0x0FFE` | Wide | `0x0891007D` |
| 3 | F-000000000012 | `0x089100E3 + 0x0FFE` | Masked | `0x089100E5` |
| 4 | F-000000000013 | `0x0891014B + 0x0FFE` | Masked | `0x0891014D` |
| 5 | F-000000000014 | `0x089101B3 + 0x0FFE` | Masked | `0x089101B5` |
| 6 | F-000000000015 | `0x0891021B + 0x0FFE` | Masked | `0x0891021D` |
| 7 | F-000000000016 | `0x08910283 + 0x0FFE` | Wide | `0x08910285` |
| 8 | F-000000000017 | `0x089102EB + 0x0FFE` | Masked | `0x089102ED` |
| 9 | F-000000000018 | `0x08910353 + 0x0FFE` | Masked | `0x08910355` |
| 10 | F-000000000019 | `0x089103BB + 0x0FFE` | Wide | `0x089103BD` |

All ten producer calls have no explicit arguments and expose a 64-bit result. Each producer after the first follows the preceding stage’s callback without an intervening direct memory operation represented in this function.

A continuation token is a numeric value used to construct a call target or argument. It is not independently established to be a valid return address, executable address, or object pointer.

### Result-pair transformations

Each result stage begins its direct memory effects with two consecutive halfword stores of the same payload to the same destination.

**Wide form**

For low word \(L\):

\[
\text{destination} = L \mathbin{\mathrm{>>}} 8.
\]

The payload is the low 16 bits of that destination. Although the represented payload extraction shifts the full 64-bit result before narrowing, the retained payload depends only on low-word bits 8–23.

The destination is between `0x00000000` and `0x00FFFFFF`.

**Masked form**

For low word \(L\):

\[
\text{destination}
= (L \mathbin{\&} \texttt{0x07FFFFFF}) \mathbin{\mathrm{>>}} 3.
\]

The payload is low-word bits 3–18, equivalently the low 16 bits of this destination.

The represented payload calculation includes a 32-bit left shift by five followed by a logical right shift by eight and narrowing to 16 bits. The intermediate width matters: bits discarded by the 32-bit left shift do not reappear.

The destination again ranges from `0x00000000` through `0x00FFFFFF`.

Neither form performs alignment correction or treats address zero as a sentinel.

### Result 1: additional writes and callback

After Result 1’s wide pair, three additional halfword stores use an address derived from the entry word:

\[
\text{derived halfword address}
= (\text{entry word} \mathbin{\&} \texttt{0x0FFFFFFF})
  \mathbin{\mathrm{>>}} 18.
\]

This address ranges from zero through `0x3FF`. The halfword payload is numerically equal to that address.

The represented intermediate narrowing does not alter the payload: after masking and shifting right by 16, at most 12 bits remain; shifting that narrowed value right by two yields the expression above.

The additional stores occur twice at the derived halfword address and then once at twice that address. A second word store of the entry word at context base plus `0x14` follows.

F-000000000020 is then called at:

\[
\texttt{0x08910015} + \texttt{0x0FFE}.
\]

Its three explicit arguments, in order, are:

1. The derived halfword address, represented as a pointer-valued argument.
2. Result 1’s high word, converted to signed 32-bit form.
3. The derived halfword address plus `0x08910015`.

The callback’s result is not consumed.

### Results 2–9: global clears and word writes

A separate word destination is derived from the entry word:

\[
\text{derived word address}
= (16 \times \text{entry word}) + \texttt{0x14}.
\]

This is distinct from context base plus `0x14`. Its machine-address interpretation is subject to the arithmetic qualifications above.

For each of Results 2–9, the order after the result pair is:

- Clear the global storage location to zero.
- Store a 32-bit payload at the derived word address.
- Call the stage’s callback.

The word payload changes at Result 7:

| Stages | Word payload |
|---|---|
| Results 2–6 | Entry word unchanged |
| Results 7–9 | Entry word shifted left by eight bits, retaining 32 bits |

The destination remains based on the unshifted entry word. Shifting the payload does not derive a new destination.

Callbacks F-000000000021 through F-000000000028 correspond to Results 2 through 9, respectively. Each callback target is that stage’s continuation token plus `0x0FFE`. Each receives, in order:

1. Zero.
2. The current result’s high word, converted to signed 32-bit form.
3. The current stage’s continuation token.

Their results are not consumed by the represented sequence.

### Result 10 and terminal sequence

After Result 10’s wide pair, the function clears the global storage location. Unlike Results 2–9, this stage contains no subsequent word store at the derived word address.

F-000000000030 is called at:

\[
\texttt{0x089103BD} + \texttt{0x07F6}.
\]

Its four explicit arguments are:

1. The 32-bit pattern `0xFFFFFF1C`.
2. Result 10’s high word converted to signed 32-bit form, then reduced by `0x264`.
3. `0x4B`.
4. Zero.

The first argument represents −228 if interpreted as signed 32-bit. The second argument’s behavior across signed overflow boundaries remains unresolved; wrapping must not be silently assumed as a language-level guarantee.

F-000000000031 follows at:

\[
\texttt{0x089104E3} + \texttt{0x07F6},
\]

with no explicit arguments.

F-000000000032, F-000000000033, and F-000000000034 then occur in that order, also with no explicit arguments. Their results are not consumed.

The representation terminates at a bad-instruction marker. This does not establish whether actual execution traps, stops, transfers elsewhere, or continues through instructions absent from the supplied representation.

## Side effects

Conditional on reaching the end of the represented sequence, its direct effects comprise:

| Effect | Count | Breakdown |
|---|---:|---|
| Halfword stores | 26 | Three entry stores, twenty result-pair stores, three additional Result 1 stores |
| Word stores | 10 | Two context-relative stores, five unshifted derived-address stores, three shifted derived-address stores |
| Global-clear stores | 9 | One after each of Results 2–10; width unresolved |
| Calls before the terminal marker | 25 | One initial call, ten producers, nine callbacks, five terminal-sequence calls |

These count events, not distinct destinations. Addresses can coincide or overlap. Halfword stores may overlap word stores or the global storage location.

There is no explicit error check, retry, rollback, or recovery branch. If an access fails or a dependency does not return, later represented operations are not guaranteed to occur.

**No explicit rollback is represented.** This is not a preservation guarantee: dependencies may overwrite or reverse earlier memory changes, including before failing to return. Later direct stores may also overwrite earlier changes through aliasing.

Likewise, a global clear establishes a zero-write event at that point, not a lasting zero value. No dependency is assumed to preserve that storage.

## Dependencies

The opaque IDs below label call sites for this contract. They do not establish implementation identity, distinctness, or provenance.

| Dependency IDs | Visible interface |
|---|---|
| F-000000000001 | Initial indirect call; target depends on entry continuation; no explicit arguments |
| F-000000000010–F-000000000019 | Ten producer calls at the explicit target expressions listed above; represented 64-bit results |
| F-000000000020 | Three-argument callback using the derived halfword address, Result 1 high word, and an address-like sum |
| F-000000000021–F-000000000028 | Eight three-argument callbacks using zero, the corresponding high word, and a continuation token |
| F-000000000030 | Four-argument terminal-stage call |
| F-000000000031 | Subsequent indirect call without explicit arguments |
| F-000000000032–F-000000000034 | Three final calls in fixed order |

Dependency calling conventions, implicit inputs, register preservation, memory effects, exceptions, termination, allocation, release, and ownership behavior are **documented_only**. The visible target expressions remain established even though these behavioral properties are unknown.

## Acceptance examples

These are conditional static examples, not executed tests. They assume successful accesses, returning dependencies, and, where needed, the stated 32-bit machine interpretation.

### Entry-address truncation and repetition

For entry address `0x00001234`, the first three halfword stores carry `0x1234`: twice at `0x00001234`, then once at `0x00002468`.

A trace containing only one store at the first address does not match the represented sequence.

### Entry-word-derived destinations

For entry word `0x12345678`:

- The derived halfword address is `0x8D`.
- Result 1’s additional halfword payload is `0x008D`, stored twice at `0x8D` and once at `0x11A`.
- Under 32-bit truncation, the derived word address is `0x23456794`.
- Results 2–6 store `0x12345678` there.
- Results 7–9 store `0x34567800` at that same address.

The odd halfword address `0x8D` is not rounded or rejected by visible logic. Whether the platform accepts it is unresolved.

### Wide and masked results

For a wide stage with low word `0x12345678`, both halfword stores target `0x00123456` and carry `0x3456`.

For a masked stage with low word `0xF1234568`, both stores target `0x002468AD` and carry `0x68AD`. Changing only bits 27–31 of that low word does not change the destination or payload.

Changing only a result’s high word leaves its pair unchanged, but changes the corresponding high-word argument calculation.

### Producer targets remain specified

Result 3 is produced by F-000000000012 at `0x089100E3 + 0x0FFE`. Its following callback uses the different base `0x089100E5 + 0x0FFE`.

An unspecified producer target, or substitution of the callback base for the producer base, does not match the static contract.

### Zero destination and final-stage distinction

A wide-stage low word below `0x100` produces address zero and payload zero. The function attempts two halfword stores there rather than taking a visible error branch.

After Result 10’s pair, the accepted direct sequence contains the global clear followed by F-000000000030, without another derived-address word store.

### Dependency overwrite

If a dependency overwrites an earlier destination and then fails to return, this contract does not require the earlier value to survive. Only the direct operations reached before that call, and their represented order, are established.

## Uncertainties

The following remain **documented_only**:

- Accuracy of the supplied instruction and control-flow reconstruction, particularly around the reported bad instruction.
- Actual entry calling convention and preservation of implicit state across calls.
- Pointer width beyond the representation’s 32-bit casts and arithmetic.
- Machine semantics for signed-looking overflow and conversions.
- Global-clear width, byte layout, and overlap with other storage.
- Memory mapping, access permissions, alignment requirements, endianness, and hardware side effects.
- Dependency behavior, including whether dependencies reverse earlier changes or do not return.
- Whether the apparent 64-bit producer results correspond to actual dependency return types.
- The terminal execution outcome and existence of any executable normal-return path.

Repeated stores could be intentional transactions or reconstruction artifacts. Neither interpretation is established.

## Evidence

The sole evidence is the supplied static representation. It supports the explicit producer-target expressions, callback-target expressions, masks, shifts, store widths, argument ordering, and represented operation order.

The event counts and numeric examples are static deductions from those expressions. They are not runtime observations.

The supplied warnings limit confidence in executable control flow and global-storage interpretation. No independent instruction inspection, dependency inspection, implementation access, or runtime testing has been performed.
