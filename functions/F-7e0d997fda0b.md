<!-- Trusted provenance: F-7e0d997fda0b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7e0d997fda0b

## Purpose

F-7e0d997fda0b describes a partially recovered operation that interleaves four byte writes with twenty-four indirect dependency calls. Three writes select their destination through the same byte-sized selector location. The fourth selects its destination through a location derived from an earlier dependency result.

The established behavior is limited to the supplied static representation. That representation contains warnings about invalid instruction data and ends at a truncated control-flow boundary. It therefore supports a contract for the visible sequence, not a claim that the sequence is a complete, correctly recovered executable function.

No application-level purpose is established. The repeated byte writes and call groups may reflect a protocol or device interaction, but identifying a device, command format, or transaction would be inferred intent rather than evidence.

## Interface

**Explicit input**

- One value, designated **P**, represented as an unsigned 32-bit integer.
- The visible sequence uses both the complete value and selected portions of it.

**Implicit inputs**

- An incoming register-derived value, designated **R**, represented as a signed integer. The visible interface does not establish how a caller supplies it or whether it is a legitimate hidden argument.
- An incoming link-register-derived address, designated **L**, used to resolve the first indirect dependency.
- Memory contents at the selector locations described below.
- Dependency behavior, including returned values, memory changes, control transfer, and possible failures.

**Visible output**

- Four explicit one-byte memory writes, if execution reaches them.
- Ordered dependency invocations and their explicit arguments.
- No explicit return value.

The representation declares a void result but does not show an ordinary return. Its visible endpoint is an invalid-instruction or unrecovered-control-flow boundary.

**Width and addressing model**

The representation identifies bytes as 8-bit values and unsigned integer values as 32-bit values. The descriptions of masking, truncation, and logical right shifts below use those widths.

The pointer width and complete execution ABI are not independently established. Where a 32-bit address-arithmetic interpretation is used, it is identified as conditional rather than asserted for an unknown execution environment.

**Preconditions for reaching the visible endpoint**

- Every reached selector read must be permitted.
- Every reached byte write must be permitted.
- Every reached indirect call target must be executable under the actual instruction-set and calling conventions.
- Each dependency must return normally for subsequent visible operations to occur.
- Arithmetic used in address formation and signed left shifts must have a defined interpretation in the actual execution environment.

The visible sequence contains no validation that establishes these preconditions.

## Behavior

### 1. Address selection and retained values

The operation first computes an integer-derived base, designated **B**, from R multiplied by 524,288, or 2¹⁹.

This computation is represented using a signed integer. The supplied material does not establish whether overflowing products wrap, trap, or are artifacts of recovery. Consequently, behavior for overflowing R values is documented_only.

For the first three writes, the selector location is B + 1. At each applicable point, the operation reads one unsigned byte from that location. A selector value **S** identifies the byte-write destination S + 17.

Thus, when ordinary integer-to-address interpretation applies, a selector can identify only addresses 17 through 272 inclusive. The selector is not a pointer-sized value and is not sign-extended.

B is computed once. The selector byte at B + 1 is read again before each of the first three writes; it is not cached across their intervening dependencies.

### 2. First write and first dependency group

The first observable memory operation is a one-byte read at B + 1. The operation then writes the low eight bits of P to the address selected by that read.

The following calls occur in order:

| Order | Dependency | Explicit argument | Consumed result |
|---|---|---|---|
| 1 | F-d10000000001 | P, as a 32-bit unsigned value | None |
| 2 | F-d10000000002 | No explicit argument | One 32-bit payload, A |
| 3 | F-d10000000003 | A | None |
| 4 | F-d10000000004 | No explicit argument | None |
| 5 | F-d10000000005 | No explicit argument | None |
| 6 | F-d10000000006 | No explicit argument | None |
| 7 | F-d10000000007 | No explicit argument | An integer result, X |

“No explicit argument” describes only the visible representation. It does not establish that the dependency has no hidden inputs, reads no registers, or accesses no shared state.

The first dependency is resolved relative to L, using an offset of 4,080. Its identity may therefore depend on the incoming execution context. The other dependencies are reached through fixed-address-derived indirect targets in the supplied representation.

### 3. Second write and second dependency group

After F-d10000000007 returns, the operation reads a fresh selector byte at B + 1.

It writes the low byte of P logically shifted right by 22 to the newly selected destination. Equivalently, the stored byte consists of P bits 22 through 29. Bits 30 and 31 do not survive the byte truncation.

It then performs these calls:

| Order | Dependency | Explicit argument | Consumed result |
|---|---|---|---|
| 8 | F-d10000000008 | X shifted left by two bits | None |
| 9 | F-d10000000009 | No explicit argument | None |
| 10 | F-d1000000000a | No explicit argument | None |
| 11 | F-d1000000000b | No explicit argument | None |
| 12 | F-d1000000000c | No explicit argument | Unsigned 32-bit result U |

X is represented as signed. The contract does not silently substitute unsigned modular multiplication for this left shift. Negative X values and values whose shift exceeds the signed range require the actual machine semantics to determine the argument supplied to F-d10000000008.

### 4. Third write and third dependency group

After U is obtained, the operation reads another fresh selector byte at B + 1. Designate this particular byte **S3**.

It writes a byte equal to four times the low byte of U, reduced to eight bits, to S3 + 17. Equivalently, the stored byte is:

**4 × (U modulo 256), modulo 256.**

Only U bits 0 through 5 affect this stored byte. Its lowest two bits are zero.

The operation then invokes:

| Order | Dependency | Explicit argument | Consumed result |
|---|---|---|---|
| 13 | F-d1000000000d | S3, zero-extended to an unsigned 32-bit value | None |
| 14 | F-d1000000000e | No explicit argument | One 32-bit payload, C |
| 15 | F-d1000000000f | C | None |
| 16 | F-d10000000010 | No explicit argument | None |
| 17 | F-d10000000011 | No explicit argument | None |
| 18 | F-d10000000012 | No explicit argument | None |
| 19 | F-d10000000013 | No explicit argument | An integer result, Y |

F-d1000000000d receives the selector value used for the immediately preceding write. It does not receive U, the written byte, or the destination address.

C is forwarded as an uninterpreted 32-bit payload. Nothing establishes that A or C is a pointer, resource handle, status, or ownership-bearing object.

### 5. Fourth write and final dependency group

The fourth selector read uses a different location. Its address is derived from U multiplied by eight, then increased by one.

Under the represented 32-bit unsigned arithmetic, that selector address is:

**((8 × U) + 1) modulo 2³².**

This address calculation can wrap. It is not a bounds-checked array access. U bits 29 through 31 do not affect the wrapped selector address.

The operation reads one unsigned byte from that location and writes to the address obtained by adding 17 to that byte.

The stored value is the low byte of U after masking to its low 30 bits and logically shifting right by 20. The intermediate shifted value can be as large as 1,023, but the write retains only eight bits. Consequently, the final stored byte is exactly U bits 20 through 27; bits 28 and 29 are discarded by byte truncation, while bits 30 and 31 were removed by the mask.

The remaining calls occur in order:

| Order | Dependency | Explicit argument | Consumed result |
|---|---|---|---|
| 20 | F-d10000000014 | Y shifted left by two bits | None |
| 21 | F-d10000000015 | No explicit argument | None |
| 22 | F-d10000000016 | No explicit argument | None |
| 23 | F-d10000000017 | No explicit argument | None |
| 24 | F-d10000000018 | No explicit argument | None |

The same signed-shift uncertainty described for X applies to Y.

After the final call, the supplied representation reaches a bad-instruction boundary. No successful return, cleanup phase, or later state transition is established.

## Side effects

### Explicit memory effects

The visible sequence performs:

- Four one-byte selector reads.
- Four one-byte writes.
- No explicit wider data-memory write.
- No explicit allocation or release.

These statements concern only operations visible in F-7e0d997fda0b. Dependencies may perform additional reads, writes, allocation, release, synchronization, or control transfer.

The write destinations may coincide. They may also alias selector locations. There is no requirement that successive writes retain distinct values or affect distinct objects.

In particular, a write or dependency can change the selector used by a later read. Implementations of this contract must not replace the three B + 1 reads with a single cached read unless an external guarantee makes that transformation behaviorally equivalent.

### Ordering and failure visibility

Each explicit write precedes its following call group. Every later stage depends on prior calls returning.

There is no visible transaction, rollback, retry, error conversion, or compensating write. If a call does not return or a memory access fails, earlier effects may remain observable. The representation does not specify the resulting exception, signal, fault code, or external error report.

### Ownership and shared state

No ownership transfer is established for P, A, C, X, Y, U, or the selector bytes.

Passing a payload to a dependency does not prove that the dependency consumes, retains, or releases a resource. Likewise, no dependency is assumed to preserve global state or selector memory. Repeated reads are evaluated against the state present when they occur.

## Dependencies

The dependency identifiers in this document are opaque, document-local labels for the twenty-four visible call sites. Their numbering denotes invocation order, not a known relationship among implementations.

- **F-d10000000001:** Context-relative indirect target; receives P.
- **F-d10000000002 and F-d10000000003:** Produce and forward payload A.
- **F-d10000000004 through F-d10000000006:** Calls with no explicit inputs or consumed outputs.
- **F-d10000000007 and F-d10000000008:** Produce X and later receive its left-shifted form, with the second byte write occurring between them.
- **F-d10000000009 through F-d1000000000b:** Calls with no explicit inputs or consumed outputs.
- **F-d1000000000c:** Produces U, which controls the third write value, fourth selector address, and fourth write value.
- **F-d1000000000d:** Receives the zero-extended third selector.
- **F-d1000000000e and F-d1000000000f:** Produce and forward payload C.
- **F-d10000000010 through F-d10000000012:** Calls with no explicit inputs or consumed outputs.
- **F-d10000000013 and F-d10000000014:** Produce Y and later receive its left-shifted form, with the fourth byte write occurring between them.
- **F-d10000000015 through F-d10000000018:** Final calls with no consumed results.

No dependency’s internal behavior, error contract, reentrancy, memory ownership, or preservation of global state is established.

## Acceptance examples

These are conditional acceptance cases derived from static evidence. They are not runtime test results. Each assumes valid reached memory accesses and normally returning dependencies.

### Stable selector and small shift operands

Suppose:

- P is hexadecimal 12C000A5.
- Each of the three reads at B + 1 returns hexadecimal 20.
- F-d10000000007 returns X = 3.
- F-d1000000000c returns U = hexadecimal 12345678.
- F-d10000000013 returns Y = 5.
- The fourth selector read returns hexadecimal 40.

Then the explicit writes, in order, are:

1. Hexadecimal A5 to address hexadecimal 31.
2. Hexadecimal 4B to address hexadecimal 31.
3. Hexadecimal E0 to address hexadecimal 31.
4. Hexadecimal 23 to address hexadecimal 51.

F-d10000000008 receives 12, F-d1000000000d receives 32, and F-d10000000014 receives 20.

The fourth selector location is hexadecimal 91A2B3C1 under 32-bit unsigned address arithmetic.

### Selector changes between groups

Suppose the first selector read returns 5, and an intervening dependency changes the selector memory so the second read returns 9.

The first write targets address 22; the second targets address 26. Reusing the first selector for the second write would violate the visible sequence.

### Byte truncation of the third write

If U is 64, the third stored byte is zero, not 256. The destination still depends on S3, and F-d1000000000d still receives S3.

### Wrapped fourth selector address

If U is hexadecimal 20000000, the fourth selector address is 1 under 32-bit unsigned arithmetic. The fourth stored byte is zero.

This example does not establish that address 1 is readable or that execution reaches the write.

### Non-returning dependency

If F-d10000000007 does not return, the first write and the first seven call invocations may have occurred. The second, third, and fourth writes do not occur along that execution.

## Uncertainties

The following remain documented_only:

- **Recovery validity:** Invalid-instruction warnings may indicate incorrect instruction decoding, an incorrect function boundary, or genuinely invalid executable data.
- **Complete control flow:** No valid continuation or ordinary return is established after the visible endpoint.
- **ABI:** The roles and widths of incoming register-derived state are not independently verified.
- **Signed arithmetic:** Overflow in R × 2¹⁹ and signed left shifts of X or Y are not assigned invented semantics.
- **Address interpretation:** Pointer conversion, accessibility of low addresses, address wrapping outside explicitly unsigned calculations, and memory-mapped behavior are unknown.
- **Dependency identity:** The first target depends on L. Actual executable identity and validity of every indirect target are unverified.
- **Shared state and ownership:** Dependency side effects and resource contracts are unknown.
- **Concurrency and hardware behavior:** No volatility, atomicity, inter-thread ordering, or device-register semantics are established.
- **Intent:** There is insufficient evidence to name a protocol, data structure, or business operation.

## Evidence

Established static evidence consists of the supplied representation’s explicit argument, arithmetic, byte accesses, indirect calls, result forwarding, call order, and terminal invalid-instruction marker.

The four write-value descriptions and conditional acceptance values are arithmetic consequences of the represented widths, masks, and truncations. The 32-bit wrapping interpretation of the fourth selector calculation follows its unsigned operand; actual pointer realization remains environment-dependent.

The representation also reports overlapping global symbols. That warning supplies no additional reliable object layout or ownership information.

No runtime tests, executable inspection, dependency inspection, or implementation validation have been performed. This contract separates the visible static sequence from inferred intent and leaves unsupported execution details documented_only.
