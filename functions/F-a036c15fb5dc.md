<!-- Trusted provenance: F-a036c15fb5dc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a036c15fb5dc

## Purpose

F-a036c15fb5dc performs an ordered sequence of indirect calls, one byte-sized update through a returned address, and two four-byte writes to low-address storage. It derives several call arguments from implicit machine state and from results of earlier calls. After twelve identifiable call sites, it transfers control through an unresolved indirect target.

The supplied static representation establishes this sequencing but does not establish an application-level purpose. In particular, there is insufficient evidence to describe the routine as initialization, decoding, device control, object construction, or any other domain operation.

This contract distinguishes:

- **Established static evidence:** operations, data dependencies, and ordering explicitly represented in the supplied material.
- **Inferred intent:** possible interpretations that are not acceptance requirements.
- **Documented-only unknowns:** behavior that cannot be resolved from the available representation.

No runtime tests have been performed.

## Interface

### Invocation and result

The recovered interface has no explicit parameters and no declared return value. It is not a complete calling-convention specification: the body consumes implicit register state and secondary register outputs from calls.

The relevant apparent inputs are:

| Input | Width and interpretation | Use |
|---|---|---|
| Register r5 | 32-bit signed integer in the recovered representation | Supplies the low byte for a memory write and the source of a subtract-and-shift argument. |
| Register r6 | 32-bit unsigned integer | Supplies two shifted arguments and the value written to one global region. |
| Register r3 | Opaque 32-bit value | Written to another global region. |
| Link-register state | Address-sized value, apparently 32 bits | Determines the first indirect call target by adding 0xD20. |
| Unresolved transfer target | Executable target with unknown provenance | Receives control after the other calls. |
| Memory and callee state | Unspecified | Used and potentially modified by every dependency. |

These inputs are implicit in the recovered representation, not evidence of an ordinary public API. The true entry convention, register-preservation rules, and executable-address conventions remain unknown.

The register-derived values are treated below as the values carried by the corresponding recovered expressions. Whether they are genuine entry snapshots, preserved register values, or artifacts of incomplete recovery is documented-only.

The function provides no explicit result. Results from earlier calls are consumed internally. The final indirect transfer is represented as a call followed by return, but its actual control-flow role is unresolved.

### Memory layout

Three explicit stores are visible:

| Destination | Extent | Stored value |
|---|---:|---|
| Address returned by the second call, plus byte offset 5 | 1 byte | Low eight bits of the r5-derived value |
| Address 0x45 | 4 bytes | The 32-bit r6-derived value shifted left by 22 bits |
| Address 0x41 | 4 bytes | The opaque r3-derived value |

The two four-byte regions cover 0x45–0x48 and 0x41–0x44, respectively. They are adjacent, not mutually overlapping. The source includes a warning that global symbols overlap smaller symbols at their addresses; this does not establish the identities, meanings, or extents of those aliases.

The byte store does not imply an object type or a six-byte object layout. It establishes only that byte offset 5 from the returned address is written. For ordinary memory, that requires the destination byte to be writable.

Endianness is not established. Consequently, the four-byte stores specify 32-bit values, not an independently established sequence of byte values.

## Behavior

### Dependency notation and execution conditions

The opaque IDs F-000000000001 through F-00000000000d below identify call sites for this document only. They are not externally verified function identities. Different IDs do not prove distinct implementations, and the target of a dynamically selected call need not remain constant across invocations.

The following sequence describes the normal path on which each preceding call returns and each explicit memory access completes. No local recovery path is shown for a fault, exception, nonlocal exit, or nonreturning dependency.

“No explicit arguments” means the recovered call expression supplies none. It does not establish that the callee has no implicit inputs.

### Initial calls and stores

1. **Invoke F-000000000001.**  
   The target is derived by adding 0xD20 to the incoming link-register-derived address. No explicit arguments or consumed return value are shown.

   The representation does not establish whether this address is executable, whether address arithmetic wraps, or whether an instruction-set state bit participates in target interpretation.

2. **Invoke F-000000000002.**  
   This target is derived from an embedded address and a fixed displacement. No explicit arguments are shown. Its primary result is interpreted as an address for the next store.

3. **Write the r5-derived byte.**  
   Store the low eight bits of the r5-derived value at byte offset 5 from the address returned by F-000000000002.

   No null check, bounds check, alignment check, or ownership check is visible. The function does not explicitly read the destination byte before replacing it.

4. **Write the shifted unsigned value.**  
   Form a 32-bit value by shifting the unsigned r6-derived value left by 22 bits, discarding bits shifted beyond bit 31. Store that value as four bytes beginning at address 0x45.

   Only the original low ten bits can contribute to the result.

5. **Write the opaque value.**  
   Store the r3-derived 32-bit value as four bytes beginning at address 0x41.

The explicit source order is therefore: returned-address byte store, four-byte store at 0x45, then four-byte store at 0x41. This is not a guarantee of atomicity, inter-thread visibility, or physical bus transaction order.

### Unsigned transformations and first result forwarding

6. **Invoke F-000000000003 with two explicit arguments.**

   - First argument: the r6-derived unsigned value shifted left by 22 bits in 32-bit width.
   - Second argument: that value shifted right logically by four bits.

   The left shift has the same recovered expression as the preceding store at 0x45. It is not shown as a readback from that storage.

7. **Invoke F-000000000004 with one explicit argument.**  
   Its argument is an unsigned secondary output associated with F-000000000003, shifted right logically by 17 bits.

   The recovered representation associates this output with register r1. It is distinct from an ordinary primary return value. The result after shifting lies between zero and 32,767 inclusive.

8. **Invoke F-000000000005.**  
   No explicit arguments are shown. Retain its opaque four-byte primary result for the immediately following call.

9. **Invoke F-000000000006 with two explicit arguments.**

   - First argument: the four-byte primary result of F-000000000005, forwarded without a visible transformation.
   - Second argument: the signed r5-derived value, reduced by nine and then shifted right by nine bits.

The subtraction precedes the shift. This is not equivalent to shifting first and then subtracting nine.

The recovered type makes the second expression signed. For nonnegative intermediate values, right shifting by nine produces division by 512 rounded down. For negative intermediate values, sign-extending arithmetic shift is the apparent interpretation, but the exact target instruction has not been established independently. Signed overflow in the subtraction is also unresolved; it must not be silently treated as unbounded arithmetic or assumed wrapping arithmetic.

### Signed transformations and remaining calls

10. **Invoke F-000000000007.**  
    No explicit arguments are shown.

11. **Invoke F-000000000008 with one explicit argument.**  
    The argument is a signed secondary output associated with F-000000000007, shifted right by 18 bits.

    This secondary output is represented as register-r1 state. Its signed interpretation differs from the unsigned secondary output consumed after F-000000000003.

12. **Invoke F-000000000009 with two explicit arguments.**

    - First argument: the signed 32-bit primary result of F-000000000008.
    - Second argument: that same result shifted right by 14 bits using the recovered signed interpretation.

    No intervening call or explicit memory access appears between production and consumption of this result.

13. **Invoke F-00000000000a.**  
    No explicit arguments or consumed result are shown.

14. **Invoke F-00000000000b.**  
    No explicit arguments or consumed result are shown.

15. **Invoke F-00000000000c.**  
    No explicit arguments or consumed result are shown.

16. **Transfer control to F-00000000000d.**  
    The target is unresolved. The supplied representation models the transfer as an indirect call and then shows a return, while explicitly warning that the underlying control transfer was not recovered reliably.

    Thus, normal return after this transfer is conditional on the call-like interpretation being accurate and the target returning. Neither a definite tail transfer nor a definite ordinary call can be required from this evidence alone.

## Side effects

### Explicit memory effects

The function visibly performs three stores, in the order specified above. It has no explicit reads of application data from those destinations.

This does not make the overall routine write-only. Dependencies may read or write arbitrary accessible state, and call-target resolution or execution may involve additional memory access not represented as data operations here.

The first explicit store depends on a pointer-like result from F-000000000002. No allocation, object construction, lifetime extension, release, or ownership transfer is shown. A caller must not infer that this returned storage belongs to F-a036c15fb5dc or that it remains valid after the store.

The two low-address stores may target ordinary memory, mapped hardware, aliases, or incorrectly recovered locations. Their application meaning is unknown. No volatile, synchronization, or atomic-access semantics are established.

### Callee effects and preservation

No dependency contract is available. None may be assumed to:

- Preserve the global regions written here.
- Preserve pointed-to storage.
- Preserve implicit register-derived inputs.
- Avoid allocation or release.
- Return normally.
- Avoid callbacks, exceptions, or nonlocal control transfer.

Consequently, the stored values are guaranteed only as explicit write events on the represented path, not as final post-return memory contents. Even the dependency immediately following a store may overwrite that destination.

### Failures and partial progress

There is no explicit status result, validation branch, retry, rollback, or cleanup path.

If the byte store fails, neither four-byte store has yet been reached. If the store at 0x45 fails, the byte store has already been attempted but the store at 0x41 has not been reached. If the store at 0x41 fails, the preceding two stores have already been attempted.

These statements describe source-level progress, not transactional guarantees. A target platform’s fault behavior may affect whether a particular store completed, partially completed, or caused other observable effects.

## Dependencies

| Opaque call-site ID | Explicit inputs | Outputs consumed here |
|---|---|---|
| F-000000000001 | None; target depends on link-register-derived state | None |
| F-000000000002 | None | Address-like primary result |
| F-000000000003 | Two unsigned shift results | Unsigned secondary r1 output |
| F-000000000004 | Secondary output shifted right by 17 | None |
| F-000000000005 | None | Opaque four-byte primary result |
| F-000000000006 | Forwarded four-byte result; signed subtract-and-shift result | None |
| F-000000000007 | None | Signed secondary r1 output |
| F-000000000008 | Signed secondary output shifted right by 18 | Signed 32-bit primary result |
| F-000000000009 | Primary result; that result shifted right by 14 | None |
| F-00000000000a | None | None |
| F-00000000000b | None | None |
| F-00000000000c | None | None |
| F-00000000000d | None shown; unresolved target | None established |

The intermediate targets other than the first and last are expressed through embedded address bases and fixed displacements. Their calling conventions, parameter types beyond local usage, memory effects, and ownership rules are unknown.

## Acceptance examples

These are conditional acceptance criteria derived from the static representation, not observed executions. Unless stated otherwise, assume preceding calls return, destination memory is writable, and the recovered implicit values remain applicable.

### Byte truncation and write order

Given an r5-derived value of 0x12345678 and an address returned by F-000000000002:

- Byte offset 5 receives 0x78.
- This store precedes both four-byte stores.
- No neighboring byte update is explicitly required by this function.

A dependency may independently modify neighboring bytes; such effects are outside the explicit-store contract.

### Unsigned shift truncation

Given an r6-derived value of 0x00000401:

- The left-shift result is 0x00400000 because the contribution from original bit 10 is shifted beyond 32 bits.
- Address 0x45 receives that 32-bit value.
- F-000000000003 receives 0x00400000 and 0x00000040.

This example distinguishes 32-bit truncation from unbounded left-shift arithmetic.

### Secondary unsigned output

If F-000000000003 supplies secondary output 0xFFFFFFFF, F-000000000004 receives 0x00007FFF.

This criterion concerns the secondary output, not the primary return value of F-000000000003.

### Subtraction before shifting

For the r5-derived signed values below, no overflow or negative intermediate is involved:

| Input | Subtract-nine result | Argument to F-000000000006 |
|---:|---:|---:|
| 9 | 0 | 0 |
| 520 | 511 | 0 |
| 521 | 512 | 1 |

The opaque primary result from F-000000000005 must independently be forwarded as the other argument.

### Conditional negative-shift behavior

If the target uses sign-extending right shifts:

- An r5-derived value of 8 produces −1 as the subtract-and-shift argument.
- A secondary output of −1 from F-000000000007 produces −1 as the argument to F-000000000008.
- A primary result of −16,385 from F-000000000008 produces arguments −16,385 and −2 to F-000000000009.

These negative-value expectations remain conditional until instruction semantics are established.

### Nonreturning dependency

If F-000000000005 does not return, F-000000000006 and all later call sites are not reached through the represented normal path. The three explicit stores precede that point, but their persistence is not guaranteed.

## Uncertainties

The following are **documented_only**, not implementation requirements inferred beyond the evidence:

- The complete entry calling convention and whether the apparent register inputs are true entry values.
- The validity of the recovered function boundary and embedded-address target calculations.
- The executable target selected from link-register-derived state.
- Exact machine behavior for signed subtraction overflow, especially near the minimum signed 32-bit value.
- Exact right-shift behavior for negative signed values.
- Address-overflow, alignment, and instruction-state rules.
- Endianness, memory mapping, alias definitions, and access semantics for addresses 0x41 and 0x45.
- The type, extent, lifetime, and ownership of the address returned by F-000000000002.
- All dependency preservation, mutation, allocation, release, and error behavior.
- Whether secondary register outputs have a formal ABI meaning or reflect incomplete recovery.
- The provenance and control-flow role of the final unresolved target.
- Any application-level meaning of the shifts, stored values, or call sequence.

A packing or field-extraction interpretation is plausible for the unsigned shifts, but that is inferred intent only. It does not justify naming a protocol, field layout, unit, or device operation.

## Evidence

The available evidence is a single supplied static representation of F-a036c15fb5dc, including warnings about overlapping global symbols and an unrecovered final indirect transfer.

It directly supports the three explicit store expressions, their ordering, the ordered call sites, the represented widths and signedness, and the dataflow between selected call outputs and later arguments.

It does not include dependency bodies, an authoritative ABI description, an independently verified instruction listing, a memory map, or execution observations. The dependency IDs in this document are local opaque references only. No runtime tests have been performed.
