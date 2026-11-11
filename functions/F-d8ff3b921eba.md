<!-- Trusted provenance: F-d8ff3b921eba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d8ff3b921eba

## Purpose

F-d8ff3b921eba performs an ordered sequence of indirect calls, four-byte memory writes, byte-address lookups, and one-byte memory writes. Later operations derive addresses and payloads from an ambient unsigned value and from dependency results.

No application-level purpose is established. The supplied representation does not establish initialization of a particular structure, serialization, device control, or a specific protocol.

The visible path terminates at an invalid-instruction boundary rather than an ordinary return. This contract describes that visible execution prefix, conditional on each preceding call and memory access completing.

## Interface

### Explicit input and output

The function has one explicit four-byte input. Its bits are forwarded to F-D01. No direct interpretation, validation, or later direct use of that input is visible.

The apparent interface has no return value. No normal return is visible, so successful return to the caller is not an established outcome.

### Ambient inputs

The representation also uses three entry-state quantities:

- **R:** an unsigned 32-bit value used in logical shifts and subsequent address calculations.
- **S:** an opaque four-byte value used as the payload of the first direct store.
- A return-link-derived value used to construct the first indirect-call destination by adding `0xFFE`.

R and S are specification labels, not additional explicit parameters. Their origin and the machine-level convention supplying them are unresolved.

Other indirect-call destinations are constructed from code-location-derived anchors. The represented displacement is `0xFF0` for F-D21, F-D33, and F-D45, and `0xFFE` for the remaining occurrences. These calculations do not establish ordinary public interfaces for the destinations.

### Preconditions for reaching the endpoint

Reaching the visible endpoint requires:

- Every indirect destination to be executable and callable under the actual machine convention.
- Every dependency to return to the next represented operation.
- Every directly accessed address to permit the indicated access width.
- Address construction, integer-to-address conversion, and alignment to be supported by the execution environment.

The visible body does not validate these conditions.

No object extent, allocation, ownership, or lifetime is established for any accessed address. A dependency result used in arithmetic is not thereby established to be a pointer or an allocation handle.

## Behavior

### Notation and arithmetic widths

F-D01 through F-D56 identify indirect-call **occurrences in execution order**, not necessarily distinct implementations. Different occurrences may resolve to the same destination.

The following labels identify retained values:

| Label | Meaning |
|---|---|
| A, B, C, D, E | Signed integer results used in multiplication-based address or argument calculations |
| H, J | Signed integer results used in left-shifted call arguments |
| U, V | Unsigned 32-bit results used for byte payloads and, for V, a lookup address |
| T1, T2, T3 | Opaque four-byte results forwarded to subsequent calls |
| P | The retained address component derived from `(R >> 3) × 0x80000` |
| b1, b2, b3, b4 | Independently read unsigned bytes |
| low8(x) | The least significant eight bits of x |

Four-byte stores carry 32-bit payloads. Byte reads produce values from 0 through 255. One-byte stores retain only eight payload bits.

Right shifts of R, U, and V are logical unsigned shifts. The unsigned calculations `U << 2`, `(R >> 3) × 0x80000`, and `8 × V + 1` have 32-bit value semantics, including reduction modulo 2³² where necessary.

The product defining P is subsequently retained in a signed integer quantity in the representation. Its unsigned product bits are established; the conversion of values outside the signed range and their interpretation as addresses remain environment-dependent.

Signed arithmetic requires separate treatment:

- A, B, C, D, and E participate in multiplication and address-offset addition.
- The visible signed left shifts use **H and J**, for F-D40 and F-D52 respectively.
- E supplies the multiplication-based address and argument calculations surrounding F-D28; E is not used in either signed left shift.

Portable behavior is not established for signed overflow, negative signed left operands, or signed shifts whose results are not representable. The acceptance examples avoid those cases.

The expressions below identify calculated addresses, not offsets within a proven object. Nothing establishes a common structure base.

### Initial calls and four-byte stores

The first twenty-one calls and first four direct stores occur in this order:

| Position | Observable behavior |
|---|---|
| F-D01 | Receives the explicit four-byte input. |
| F-D02–F-D04 | Execute in increasing ID order; no explicit arguments are shown. |
| F-D05 | Produces A. |
| Between F-D05 and F-D06 | Store S as four bytes at address `4 × A + 0x14`. |
| F-D06 | Receives `4 × A`. |
| F-D07–F-D09 | Execute in increasing ID order. |
| F-D10 | Produces B; the value `4 × B` is retained. |
| Between F-D10 and F-D11 | Store `4 × B` as four bytes at address `16 × B + 0x14`. |
| F-D11 | Receives `4 × B`. |
| F-D12–F-D14 | Execute in increasing ID order. |
| F-D15 | Produces C. |
| Between F-D15 and F-D16 | Store the retained `4 × B` as four bytes at address `4 × C + 0x14`. |
| F-D16 | Receives `4 × C`. |
| F-D17–F-D19 | Execute in increasing ID order. |
| F-D20 | Produces D. |
| Between F-D20 and F-D21 | Store `4 × D` as four bytes at address `16 × D + 0x14`. |
| F-D21 | Receives two arguments, in order: `4 × D`, then `16 × D`. |

The third store retains its B-derived payload even though its address depends on C.

The four destinations need not be distinct or non-overlapping. No bounds or capacity checks are visible. Calls whose results are not consumed remain part of the required sequence; their effects are not known.

### Forwarding and the R-derived four-byte store

F-D22 produces T1. F-D23 receives T1 followed by `R >> 6`.

F-D24, F-D25, and F-D26 then execute in that order. F-D27 produces E. After F-D27 returns and before F-D28 begins, the function stores `R >> 3` as four bytes at address `4 × E + 0x14`.

F-D28 receives `4 × E`. F-D29, F-D30, and F-D31 follow in order. F-D32 then produces U.

This is the fifth direct four-byte store. The visible body does not directly read any of the five stored words back.

### First P-derived lookup and byte store

After F-D32, the function calculates and retains P from `(R >> 3) × 0x80000`.

It reads one unsigned byte from address `P + 1`, obtaining b1. It then stores `low8(U << 2)` at address `b1 + 0x11`.

F-D33 follows the byte store and receives the full unsigned 32-bit value `U << 2`, not merely the stored byte.

The byte-derived destination ranges from `0x11` through `0x110`, inclusive. It is not `P + 0x11`.

### Second lookup using retained P

F-D34 produces T2, which F-D35 receives. F-D36, F-D37, and F-D38 execute in order, followed by F-D39, which produces H.

The function then performs a fresh unsigned-byte read from `P + 1`, obtaining b2. It stores

`low8((U & 0x3FFFFFFF) >> 20)`

at address `b2 + 0x11`.

F-D40 follows and receives `H << 2`.

Before conversion to a byte, the masked and shifted U expression can contain ten significant bits. The stored byte contains exactly bits 20 through 27 of U; bits 28 and 29 are discarded.

H determines the argument to F-D40. H does not determine the preceding byte payload or its lookup address.

The second lookup uses the retained P but reads memory again. Reusing b1 would require a memory-invariance guarantee not present in the supplied evidence.

### Third lookup using retained P

F-D41, F-D42, and F-D43 execute in order. F-D44 produces V.

A third unsigned-byte read from `P + 1` obtains b3. The function stores `low8(4 × low8(V))` at address `b3 + 0x11`. The resulting byte is equivalently the low eight bits of `V << 2`.

F-D45 receives b3 widened to an unsigned 32-bit value. It does not receive V, the stored payload, or the destination address.

The three P-derived lookups are separate reads. Intervening calls or overlapping writes may affect the values returned by later reads.

### V-derived lookup and terminal calls

F-D46 produces T3, which F-D47 receives. F-D48, F-D49, and F-D50 execute in order. F-D51 produces J.

The function reads an unsigned byte from address `8 × V + 1`, obtaining b4. This address expression uses unsigned 32-bit arithmetic.

It then stores

`low8((V & 0x3FFFFFFF) >> 20)`

at address `b4 + 0x11`. The stored byte contains bits 20 through 27 of V.

F-D52 follows and receives `J << 2`. J controls this call argument, not the final lookup address or byte payload.

F-D53, F-D54, F-D55, and F-D56 then execute in order. The represented path reaches an invalid-instruction boundary immediately afterward. No normal return or continuation beyond that boundary is established.

## Side effects

The visible body contains:

- Fifty-six indirect-call occurrences.
- Five direct four-byte writes.
- Four direct unsigned-byte reads.
- Four direct one-byte writes.
- No explicit allocation, release, ownership transfer, validation, rollback, or normal return.

These counts exclude all dependency-internal behavior.

Each direct store precedes the next listed dependency call. That dependency may observe or overwrite the stored data. Dependencies preceding a lookup may change the storage subsequently read.

The direct accesses are not proven disjoint. A byte destination may coincide with another byte destination, overlap a four-byte destination, or overlap lookup storage. An earlier write can therefore affect a later read. The contract does not permit replacing fresh reads with cached values.

The stored four-byte values are established, but their byte order is not. No layout beyond the specified access widths and calculated addresses is supported.

There is no visible error-reporting protocol. A faulting access or a dependency that does not return prevents the remaining visible suffix from being reached through the shown path. No recovery or rollback is shown.

The externally observed response to the final invalid instruction depends on the execution environment. Process termination, handler behavior, or recovery must not be inferred from the boundary alone.

## Dependencies

| Occurrences | Established argument/result relationship |
|---|---|
| F-D01 | Receives the explicit four-byte input. |
| F-D05, F-D10, F-D15, F-D20 | Supply A, B, C, and D respectively. |
| F-D06, F-D11, F-D16 | Receive `4 × A`, `4 × B`, and `4 × C` respectively. |
| F-D21 | Receives `4 × D`, then `16 × D`. |
| F-D22 → F-D23 | T1 is forwarded; F-D23 also receives `R >> 6`. |
| F-D27 → F-D28 | F-D27 supplies E; F-D28 receives `4 × E` after the R-derived word store. |
| F-D32 → F-D33 | F-D32 supplies U; F-D33 receives the unsigned 32-bit value `U << 2`. |
| F-D34 → F-D35 | T2 is forwarded. |
| F-D39 → F-D40 | F-D39 supplies H; F-D40 receives the signed expression `H << 2`. |
| F-D44 | Supplies V. |
| F-D45 | Receives b3 widened to an unsigned 32-bit value. |
| F-D46 → F-D47 | T3 is forwarded. |
| F-D51 → F-D52 | F-D51 supplies J; F-D52 receives the signed expression `J << 2`. |
| Remaining occurrences | No explicit arguments are shown, and no result is directly consumed. |

“No explicit arguments are shown” does not establish a verified zero-argument machine interface. Dependencies may consume ambient state.

No dependency is established to preserve globals, preserve lookup storage, own memory, allocate memory, or release memory. Forwarding an opaque four-byte result does not establish pointer semantics or an ownership transfer.

The retained-value relationships describe the supplied representation. The complete machine convention and its state-preservation guarantees remain unresolved.

## Acceptance examples

These are conditional static expectations, not executed tests. They assume that preceding dependencies return, accesses succeed, and signed arithmetic remains within defined cases.

### Initial word-store relationships

For A = 3, B = 5, C = 7, and D = 9, the first four direct stores are:

| Order | Address | Four-byte payload |
|---|---|---|
| 1 | `0x20` | S |
| 2 | `0x64` | 20 |
| 3 | `0x30` | 20 |
| 4 | `0xA4` | 36 |

The third payload remains B-derived. F-D21 subsequently receives 36 and 144, in that order.

### E multiplication versus H and J shifts

If E = 6 and R = `0x40`, the fifth four-byte store writes 8 at address `0x2C`, and F-D28 receives 24.

Separately, if H = 3, F-D40 receives 12. If J = 5, F-D52 receives 20. These latter arguments arise from signed left shifts of H and J; they do not use E.

### Full-width argument versus byte truncation

If U = `0x00000123` and b1 = `0x20`, the first byte store writes `0x8C` at address `0x31`.

F-D33 receives `0x0000048C`, preserving bits that the byte store discards.

### Upper-field extraction

If U = `0x3AB00000`, the second byte store writes `0xAB` at the destination selected by b2.

The masked and shifted intermediate value is `0x3AB`; conversion to a one-byte payload discards its upper two bits.

### Fresh lookup after an intervening effect

If the first read from `P + 1` yields `0x20`, and an intervening dependency changes that location so the second read yields `0x40`, the first two byte destinations are `0x31` and `0x51`.

Reusing the first lookup result for the second store would violate the represented behavior.

### Unsigned lookup-byte widening

If b3 = `0xFF`, the third byte destination is `0x110`, and F-D45 receives 255, not −1.

If V is also `0x000000FF`, that third store writes `0xFC`.

### Final V-derived operation

If V = `0x00100003`, the final lookup reads address `0x00800019`. If that read returns `0x22`, the final byte store writes `0x01` at address `0x33`.

For the same V, the earlier third byte store has payload `0x0C`, but its destination comes from the independent P-derived lookup.

### Interrupted execution

If F-D34 does not return, the first U-derived byte store and F-D33 have already occurred. The remaining direct reads and writes are not reached through the visible path. No rollback is specified.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and the meaning of the addressed storage.
- Complete calling convention, ambient-input origins, and machine-state preservation.
- Dependency identities, aliasing, signatures, and internal effects.
- Address width, integer-to-address mapping, and alignment requirements.
- Conversion of the unsigned P product into the represented signed storage when outside the signed range.
- Machine behavior for signed overflow and invalid signed shifts.
- Four-byte byte order.
- Memory ownership, lifetime, permissions, volatility, and hardware significance.
- Concurrency, synchronization, and memory-ordering requirements.
- Completeness and accuracy of the recovered control flow.
- Execution or recovery beyond the terminal invalid-instruction boundary.

The supplied warnings identify invalid instruction data and overlapping storage symbols. They limit confidence in a complete control-flow account and in assigning independent meanings to overlapping symbols. They do not establish additional operations or a hidden recovery path.

## Evidence

**Established static evidence:** The supplied representation supports the call order, explicit forwarding relationships, consumed results, access widths, arithmetic expressions, separate byte reads, store ordering, and terminal invalid-instruction boundary described here. In particular, E participates in multiplication-based calculations surrounding F-D28, while H and J are the operands of the signed left shifts used for F-D40 and F-D52.

**Inferred intent:** Repeated scaling and byte extraction suggest a coordinated state-update or encoding operation. No particular application-level interpretation is sufficiently supported to become a requirement.

**Unknown:** Dependency internals, runtime mappings, actual machine-state preservation, successful end-to-end execution, and behavior beyond the visible endpoint.

No runtime tests have been performed. The acceptance examples are deductions from static evidence, not observed execution results.
