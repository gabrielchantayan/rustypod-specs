<!-- Trusted provenance: F-6cf9efc1152a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6cf9efc1152a

## Purpose

F-6cf9efc1152a performs an ordered series of byte, halfword, and word memory accesses, followed by an unresolved indirect control transfer. No application-level purpose is established.

The sequence includes repeated writes, reads whose values become later addresses, three fixed-address byte destinations, and one fixed-address word source. Repeated or subsequently overwritten writes remain part of the behavior; final memory contents alone do not describe the access sequence.

There are twelve writes through doubled address values: ten opening word writes through doubled X, one later halfword write through doubled K, and one final word write through doubled L. These are twelve operations, not necessarily twelve distinct destinations.

This contract describes static evidence. It does not establish a complete calling convention or the behavior of the final indirect destination. No runtime tests have been performed.

## Interface

### Inputs and result

Two explicit inputs are displayed:

| Contract label | Width | Use |
|---|---:|---|
| P | 32 bits, represented as signed | Stored as a word; also used as the address base for a word read at offset `0x74`. |
| Q | 32 bits | Stored during the opening sequence; no arithmetic interpretation is required. |

Three additional values are consumed from ambient entry state:

| Contract label | Width | Use |
|---|---:|---|
| X | 32 bits, unsigned | Address base, doubled address source, stored word, low-byte source, and shift-count source. |
| Y | 32 bits, represented as signed | Address base and low-byte source. |
| Z | 32 bits, represented as signed | Address base and stored word. |

These labels are contract notation, not additional established formal parameters. The provenance and calling-convention roles of X, Y, and Z are **documented_only**.

No explicit result is displayed. If the final dependency returns normally, the displayed body returns without visibly using its result. The actual machine-level return convention remains unresolved.

### Widths and storage

A byte is 8 bits, a halfword is 16 bits, and a word is 32 bits. “Low byte” means the least significant eight bits. All offsets are byte offsets.

The following fixed-address locations have direct accesses:

| Contract label | Address | Direct access sequence |
|---|---:|---|
| G-wide | `0x000000a9` | One word read |
| G-digit | `0x0000003a` | One byte write |
| G-copy | `0x00000067` | One byte write, followed by three byte reads |
| G-mirror | `0x00000021` | One byte write |

These labels do not imply independent allocated objects, ownership, or complete layouts. The supplied material warns that global storage descriptions overlap. Dynamically addressed accesses may also overlap these fixed addresses.

A word access covers four consecutive bytes and a halfword access covers two. Byte order is not established. Offsets identify accessed fields, not a complete record structure.

### Preconditions and errors

Normal completion requires every reached address to support the specified access width and permissions, and requires the final indirect destination to support the displayed transfer.

There are no visible checks for null addresses, bounds, alignment, arithmetic overflow, access permissions, or target validity. No error return, exception handler, recovery path, or rollback is displayed.

Invalid-access behavior, integer-to-pointer interpretation, alignment rules, and the validity of low fixed addresses are **documented_only**. A numerically determined address is not necessarily an accessible address.

## Behavior

### Opening repeated writes

Let D be twice X, reduced modulo \(2^{32}\). Let x-byte and y-byte be the low bytes of X and Y.

The opening has ten complete groups. Each group contains these accesses in order:

1. A byte write of y-byte at `X + 0x11`.
2. A word write at `Y + 0x34`.
3. A word write of P at `D + 0x54`.
4. A byte write of x-byte at `Y + 0x19`.

The second access stores Q in the first group and D in the remaining nine groups. A final two-write tail stores y-byte at `X + 0x11`, then D at `Y + 0x34`.

The phase therefore contains 42 writes:

| Destination | Width | Count | Stored values |
|---|---:|---:|---|
| `X + 0x11` | Byte | 11 | y-byte |
| `Y + 0x34` | Word | 11 | Q once, then D ten times |
| `D + 0x54` | Word | 10 | P |
| `Y + 0x19` | Byte | 10 | x-byte |

These counts describe accesses even when destinations coincide or values repeat. Aliasing can make the interleaving affect subsequent reads.

### Initial word captures and byte calculation

After all opening writes, the body captures a word from G-wide as W, then a word from `P + 0x74` as A.

G-digit next receives the low eight bits of:

\[
2 \times \text{x-byte} + 0x30.
\]

This specifies a byte result, not a validated decimal digit or an established text conversion. Whether the intermediate byte representation is interpreted as signed does not change these final eight bits.

The next word captures are B from `Z + 0x14`, followed by C from `X + 0x04`.

These reads occur after the opening writes and may observe their effects if storage overlaps. A captured value is retained independently of subsequent changes to its source memory.

### Byte fields, G-copy, and the shift

The next writes occur in this order: x-byte at `Z + 0x0d`; byte `0x6c` at `X + 0x08`; y-byte at `X + 0x0d`; y-byte at `X + 0x09`; and the full word Z at `Z + 0x54`.

G-copy then receives W’s low byte. After that write, a signed word is captured from `Z + 0x44` as E. G-copy is subsequently read as a byte, and the resulting byte is written to G-mirror.

**Under ordinary memory semantics, with no external mutation or read-triggered side effect, this first G-copy read returns W’s low byte.** No intervening write is displayed between the explicit G-copy write and this read.

In particular, the earlier word write of Z at `Z + 0x54` cannot explain a different first G-copy value. Even if that word write overlaps G-copy, the later explicit G-copy byte write replaces the overlapping byte before the first read. The intervening read at `Z + 0x44` is not an ordinary-memory mutation merely because it aliases another location. Any uncertainty about the first G-copy read must instead concern external activity, device behavior, or a side effect of that intervening read.

E next supplies a low-byte write at `X + 0x10` and a full-word write at `Y + 0x54`, in that order. A byte is then written at `Z + 0x01` from E shifted right by the low eight bits of X. Only the shifted result’s low byte is stored. Finally, X replaces E at the word location `Y + 0x54`.

The E write remains part of the sequence despite being subsequently replaced. Intervening aliasing or externally observable storage can make it significant.

For nonnegative E and counts from 0 through 31, the shift has the ordinary right-shift result. A specific result for negative E or counts from 32 through 255 is not established.

### Word transfers and later G-copy reads

The body captures a word from `X + 0x40` as F. It then writes C at `Y + 0x74`, F at `X + 0x10`, and Z at `Y + 0x44`, all as words in that order.

The F write spans X-relative offsets `0x10` through `0x13`. It therefore covers the earlier E-byte destination at `0x10` and the opening y-byte destination at `0x11`.

G-copy is read for the second time, and its byte is written at `X + 0x09`. Next, x-byte is written at `Z + 0x09`, and W is written as a word at `X + 0x44`.

Only then is a signed word captured from `X + 0x54` as H. G-copy is read for the third time, and that byte is again written at `X + 0x09`.

Unlike the first G-copy read, these later reads have intervening writes that can overlap G-copy. Their values need not equal W’s low byte or each other. Each is a read from the storage contents present at its own position in the sequence.

### H-relative writes and the K capture

A’s low byte is written at `H + 0x0d`. B is then written as a word at `Z + 0x54`, replacing the earlier Z word there under ordinary memory semantics.

The body next captures a word from `X + 0x14` as J. It then writes A’s low byte at `Z + 0x11`, H’s low byte at `X + 0x09`, and H’s full word at `X + 0x44`, in that order.

The last write replaces the earlier W word at the same X-relative destination. It does not change captured W.

A signed word is then captured from `Z + 0x04` as K.

### Final captures, doubled-address writes, and transfer

A halfword with numeric value `0x0065` is written at twice K. The byte order of this halfword is unresolved.

After that write, a signed word is captured from `K + 0x64` as L. If the read overlaps the preceding halfword write, its value reflects that ordering under ordinary memory semantics.

J’s low byte is then written at `H + 0x11`, followed by a word write of decimal 100 at `H + 0x54`.

The next captures are M, a word from `K + 0x74`, and N, a signed word from `Z + 0x64`, in that order. K’s low byte is then written at `Z + 0x09`.

The last two explicit writes place L as a word at `N + 0x54`, then N as a word at twice L.

The unresolved dependency F-unknown-01 receives these four displayed arguments:

| Position | Value |
|---|---|
| First | `0x72`, decimal 114 |
| Second | L |
| Third | B |
| Fourth | M |

L, B, and M are captured 32-bit values, not fresh reads from their original source locations. No further explicit memory operation is displayed before return, assuming the dependency returns normally.

### Arithmetic boundaries

X-relative additions, doubling X, and adding `0x54` to D use unsigned 32-bit numerical arithmetic. Their results reduce modulo \(2^{32}\). This applies to every displayed X-relative access.

P, Y, Z, H, K, L, and N are represented as signed 32-bit values when used in address arithmetic. Their displayed additions, and the doublings of K and L, do not establish wraparound semantics when signed arithmetic overflows. Those cases remain **documented_only**.

The numerical rule and the validity of the resulting memory access are separate. Established unsigned wrapping does not establish pointer validity, accessibility, alignment, or permission.

## Side effects

The body directly reads and writes fixed-address storage, input-relative storage, storage reached through captured words, and doubled-address destinations.

G-wide has one direct word read and no direct fixed-address write. Another destination may nevertheless overlap its bytes.

Before the indirect transfer, no allocation, deallocation, explicit ownership transfer, reference-count operation, lock operation, or error-reporting operation is displayed. This does not establish ownership or lifetime guarantees. Storing an address-like value does not itself demonstrate retention or acquisition of a resource.

The displayed ordering distinguishes captures, temporary values, overwritten fields, and repeated accesses. It does not establish atomicity, synchronization, hardware transaction boundaries, or visibility to concurrent observers.

Ordinary memory semantics support the stated overwrite relationships and the first G-copy equality. Device storage, external mutation, or read-triggered effects may produce additional behavior not characterized by the supplied body.

No rollback is displayed. If execution stops partway through, earlier writes have no visible compensating operations. The actual fault mechanism and resulting observable state remain environment-dependent.

F-unknown-01 may affect memory, globals, resources, or control flow. No preservation or ownership guarantee is available for that dependency.

## Dependencies

### F-unknown-01

F-unknown-01 is a contract-local opaque identifier for the unresolved indirect destination. Its identity is not established.

The displayed transfer supplies `0x72`, L, B, and M, in that order. The following remain **documented_only**:

- Destination selection and validity.
- Calling convention and any additional ambient inputs.
- Memory and global preservation.
- Resource acquisition, retention, transfer, or release.
- Result values and error signaling.
- Faults, exceptions, nonlocal exits, and termination behavior.
- Whether the transfer is correctly represented as a returning invocation.

No result is visibly consumed. That observation does not establish that the dependency has no result or no effects.

No other function dependency is displayed.

## Acceptance examples

These are conditional static examples, not executed tests. Unless a boundary is being illustrated, they assume valid accesses and permitted arithmetic.

### Repetition and doubled addresses

For X equal to `0x1000`, D is `0x2000`. The opening sequence contains ten word writes of P at `0x2054`.

At `Y + 0x34`, it writes Q once and `0x2000` ten times, with the interleaving specified above. The later doubled-K and doubled-L writes bring the function’s doubled-address write count to twelve.

### Unsigned wraparound

For X equal to `0xfffffffe`:

- D is `0xfffffffc`.
- `X + 0x11` numerically becomes `0x0000000f`.
- `D + 0x54` numerically becomes `0x00000050`.

These are established unsigned numerical results, not guarantees that the corresponding accesses succeed. They imply no signed-overflow rule for Y, K, or L.

### First G-copy read despite an earlier overlapping write

Assume W has already been captured as `0x123456ab`, Z equals `0x13`, and accesses have ordinary memory semantics without external mutation.

The word write of Z at `Z + 0x54` begins at `0x67`, overlapping G-copy. Nevertheless, the subsequent explicit G-copy write stores `0xab` there.

The intervening word read at `Z + 0x44` does not modify ordinary memory. The first G-copy read therefore returns `0xab`, and G-mirror receives `0xab`, regardless of which byte the earlier overlapping word write placed at G-copy.

A different first-read value would require an additional effect, such as external mutation or device/read-triggered behavior; the earlier word write alone is insufficient.

### Later G-copy reads can differ through ordinary aliasing

Assume X equals `0x57`, so `X + 0x10` is G-copy. If F is captured as `0x22222222`, its later word write at that destination places `0x22` at G-copy, independent of byte order.

The second G-copy read then returns `0x22`, provided no intervening effect changes that byte. This does not contradict the first read returning W’s low byte: the intervening F write occurs only later.

### Byte calculation

For x-byte equal to `0x35`, G-digit receives `0x9a`.

For x-byte equal to `0xff`, G-digit receives `0x2e` after eight-bit reduction.

Neither case establishes digit validation, rejection, or text interpretation.

### Captured values remain distinct from source storage

If B is captured as `0x11223344`, later changes to `Z + 0x14` do not cause B to be reread. The later B word write and the third displayed dependency argument still use `0x11223344`, assuming execution continues without disruption of the captured execution state.

### Shift boundaries

For nonnegative E and a zero shift count, the byte written at `Z + 0x01` is E’s low byte.

For negative E or a count of at least 32, this contract does not prescribe a particular result byte.

## Uncertainties

The following remain **documented_only**:

- The true entry convention and provenance of ambient inputs.
- Complete storage layouts, ownership, and lifetimes.
- Byte order, pointer interpretation, alignment, permissions, and fault behavior.
- Whether storage is ordinary memory or has device/external effects.
- Signed address-arithmetic overflow, including signed doubling.
- Negative signed right-shift behavior and oversized shift counts.
- F-unknown-01’s identity, full interface, effects, and return behavior.
- The accuracy of the final transfer’s returning-invocation representation.

Unsigned X-derived numerical wraparound is established, not unresolved.

The first G-copy read is also not unresolved under ordinary, stable memory semantics: it returns the explicitly stored low byte of W. Its remaining uncertainty is limited to effects outside that ordinary-memory model, including the intervening read’s possible device side effects.

No initializer, serializer, container, or text-processing purpose is inferred from the constants or access patterns.

## Evidence

**Established static evidence:** the supplied body supports the displayed widths, offsets, ordering, opening repetition counts, captured-value relationships, fixed-address access classifications, unsigned X-derived calculations, signed shift expression, and four displayed indirect-transfer arguments.

The ordering specifically places the word write of Z before the explicit G-copy byte write. Between that byte write and the first G-copy read, the only displayed memory access is the word read at `Z + 0x44`; no intervening write is displayed. The ordinary-memory first-read equality follows from that order.

**Inferred intent:** none is required. The contract does not assign an unsupported domain purpose.

**Unknowns:** unresolved entry state, execution-environment semantics, signed boundary behavior, and dependency behavior are retained explicitly rather than supplied by assumption.

No runtime tests, execution traces, or independent implementation checks have been performed.
