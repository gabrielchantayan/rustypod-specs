<!-- Trusted provenance: F-3aac2c52e74a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3aac2c52e74a

## Purpose

The available static evidence describes an execution prefix that coordinates dependency calls and performs five explicit memory writes: one 32-bit payload write, two derived byte writes, and two zero-byte writes.

No application-level purpose is established. The evidence does not identify a complete object initializer, serializer, allocator, buffer manager, or normally returning routine. Such interpretations must not become acceptance requirements.

The supplied representation includes an invalid-instruction warning and terminates at a decoding boundary. This contract describes only the represented prefix, conditional on valid entry state, valid instruction interpretation, successful memory accesses, and dependency calls returning as represented. It does not establish execution beyond that boundary.

## Interface

### Explicit inputs

Two explicit inputs are represented, each 32 bits wide.

| Input | Established use |
|---|---|
| First input | No explicit use within the represented prefix. |
| Second input | Passed as the sole explicitly represented argument to **F-U01**. |

Lack of an explicit use does not establish that the first input’s incoming machine state is invisible to dependencies. The complete calling convention is unavailable.

### Implicit entry state

Three additional incoming values are used without being supplied through the two explicit inputs:

- A control value from which the target of **F-U01** is derived.
- A base value used for two byte reads at base plus `0x01`.
- A 32-bit payload used by the first explicit memory write.

These are entry-state requirements, not established public parameters. Their origins, lifetimes, and calling-convention roles are **documented_only** unknowns.

Execution requires readable lookup locations, writable destinations, executable dependency targets, and any additional conditions imposed by the dependencies. The representation supplies no validation of those requirements.

### Outputs and completion

No ordinary return value is established. There is no represented normal return before the decoding boundary.

The observable prefix includes:

- Thirty dependency invocations.
- Arguments explicitly forwarded to selected dependencies.
- Four unsigned byte reads.
- One 32-bit write and four byte writes.
- Any additional effects performed by dependencies.

The writes do not establish an output structure or a contiguous memory layout. Their destinations arise from separate calculations and may alias one another or other relevant memory.

### Widths and arithmetic

Explicit inputs, the payload, and most consumed dependency results are 32 bits wide. **F-U02** supplies a 64-bit result, consumed as low and high 32-bit halves.

Each unsigned byte read yields a value from 0 through 255. Adding `0x11` to such a value produces an address in the inclusive range `0x11`–`0x110`; that addition is not truncated to eight bits. Each byte write retains the low eight bits of its supplied value.

The unsigned shifts have definite bit-level meaning. Other calculations involve represented signed 32-bit intermediates, including multiplication and left shifts. Exact target behavior for signed overflow, negative left-shift operands, conversion of large unsigned values to signed intermediates, and address conversion is not fully established by this representation.

Consequently:

- Bit-extraction requirements below are distinguished from address-arithmetic assumptions.
- A 32-bit wrapping interpretation is stated explicitly wherever an example needs it.
- Examples using a derived lookup address without that qualification use arithmetic within the signed 32-bit range.
- An example about an earlier prefix does not establish that all later arithmetic is overflow-free.

## Behavior

### Dependency identity and ordering

**F-U01** through **F-U30** are opaque invocation-site identifiers. They do not establish recovered names, distinct implementations, or independently verified function boundaries.

The target of **F-U01** depends on incoming control state. The other targets are derived from fixed values in the representation. Target validity and callee semantics are unknown.

The following phases describe the observable sequence. Every later event is conditional on earlier events completing and control continuing as represented. Retained results describe the represented data flow; they do not imply that unknown callees preserve arbitrary globals or undocumented machine state.

### Initial forwarding

The prefix first invokes **F-U01** with the second explicit input, then invokes **F-U02**.

The 64-bit result from **F-U02** supplies four 32-bit arguments to **F-U03**, in this order:

1. Its low half.
2. Its high half.
3. Its low half again.
4. Its high half logically shifted right by three bits.

The right shift is unsigned; it does not propagate the high half’s sign bit.

After **F-U03**, the prefix invokes **F-U04**, **F-U05**, **F-U06**, and **F-U07**, in that order.

### Payload write

The 32-bit result of **F-U07** is multiplied by four. The incoming implicit payload is written as a 32-bit quantity at the resulting address plus `0x14`.

Only after that write does the prefix invoke **F-U08**, with four times the **F-U07** result as its sole explicit argument. The expression is represented at both uses; the evidence does not establish a separate public address or buffer object.

The prefix then invokes **F-U09**, **F-U10**, **F-U11**, and **F-U12**, in that order.

The payload’s byte order, destination alignment requirements, and write atomicity are not established.

### Two writes derived from one retained result

The unsigned 32-bit result of **F-U12** is retained for both derived-byte writes. It is also shifted left by two bits to obtain a retained scaled word.

At the unsigned-shift stage, bits shifted beyond bit 31 are discarded. The scaled word is subsequently held in a represented signed 32-bit intermediate; uses that depend on interpreting an out-of-range unsigned result as signed remain subject to the arithmetic limitations above.

For the first derived-byte write:

- Read an unsigned byte at the incoming base plus `0x01`.
- Use that byte’s value plus `0x11` as the write address.
- Write the low byte of the retained scaled word.

The written byte is the original **F-U12** result’s low six bits shifted left by two. Its bottom two bits are zero.

The destination is derived directly from the loaded byte. The evidence does not describe loading a full-width pointer or adding the byte to an object base.

Next, invoke **F-U13**, **F-U14**, and **F-U15**, in order, retaining the 32-bit result of **F-U15**.

The second derived-byte write performs a fresh unsigned byte read at the incoming base plus `0x01`. Its destination is that newly read value plus `0x11`.

Its value is obtained from the retained **F-U12** result by masking with `0x3fffffff`, logically shifting right by 20 bits, and retaining the low byte. The resulting byte is exactly bits 20–27 of the original result. Bits 28–29 are discarded by the byte write; bits 30–31 are removed by the mask.

The second byte’s source is not replaced by the **F-U15** result. That result instead supplies the argument to **F-U16**, after a left shift by two bits.

### Two lookup-derived zero writes

After **F-U16**, the prefix invokes **F-U17**, **F-U18**, **F-U19**, and **F-U20**, in order, retaining the 32-bit result of **F-U20**.

A lookup address is calculated by multiplying the retained scaled word from **F-U12** by `0x400`, then adding `0x01`. One unsigned byte is read there. A zero byte is written at the loaded byte’s value plus `0x11`.

The prefix then invokes **F-U21** with four explicit arguments, in order:

1. The retained scaled word shifted left by three bits.
2. The same shifted value again.
3. The retained scaled word without that additional shift.
4. The **F-U20** result shifted left by two bits.

Next, **F-U22**, **F-U23**, **F-U24**, and **F-U25** are invoked in order. The 32-bit result of **F-U25** is retained.

A second unsigned byte read is performed using the same scaled lookup-address expression. Another zero byte is written at this newly loaded value plus `0x11`.

The prefix then invokes **F-U26**, passing the **F-U25** result shifted left by two bits. Finally, it invokes **F-U27**, **F-U28**, **F-U29**, and **F-U30**, in order, and reaches the decoding boundary.

No normal completion behavior is established after **F-U30**.

### Repeated reads and arithmetic limits

Neither repeated lookup may be replaced by a cached earlier value:

- **F-U13** through **F-U15** intervene between the two base-relative reads.
- **F-U21** through **F-U25** intervene between the two scaled-address reads.
- Earlier explicit writes may themselves change later lookup bytes if addresses alias.

The source establishes neither non-aliasing nor stable lookup contents.

Under an explicitly assumed 32-bit wrapping interpretation, the scaled lookup address is the original **F-U12** result shifted left by 12 bits, retaining 32 bits, then increased by one. Likewise, the first two **F-U21** arguments correspond to that original result shifted left by five bits, retaining 32 bits.

These identities do not establish defined signed arithmetic for every input. In particular, an initial unsigned left shift may be well-defined while a later multiplication of its retained signed intermediate exceeds the signed 32-bit range.

## Side effects

### Explicit memory accesses

The explicit writes occur in this order:

| Order | Width | Destination | Value |
|---|---:|---|---|
| 1 | 32 bits | Four times the **F-U07** result, plus `0x14` | Incoming implicit payload |
| 2 | 8 bits | First base-relative lookup byte, plus `0x11` | Low byte of the retained scaled **F-U12** word |
| 3 | 8 bits | Fresh base-relative lookup byte, plus `0x11` | Bits 20–27 of the **F-U12** result |
| 4 | 8 bits | First scaled-address lookup byte, plus `0x11` | Zero |
| 5 | 8 bits | Fresh scaled-address lookup byte, plus `0x11` | Zero |

Each lookup-derived write follows its corresponding byte read. No dependency invocation is represented between that read and its associated write.

The four byte-write destinations are low numeric addresses between `0x11` and `0x110`. Whether those addresses are valid ordinary memory, mapped devices, or artifacts of an incorrect instruction interpretation is **documented_only**.

### Ownership and dependency effects

No explicit allocation, release, ownership transfer, or lifetime management with known semantics is established.

An argument that resembles an address is not evidence that the recipient owns, retains, or releases memory. No dependency is known to preserve global memory, avoid allocation, avoid release, return normally, or leave lookup bytes unchanged.

The contract specifies only the explicit accesses and represented data flow of this prefix. It does not bound effects performed inside dependencies.

### Errors and interruption

No explicit bounds check, null check, error-code branch, retry, rollback, or cleanup path appears.

If a dependency fails to return, or an access prevents continued execution, later events are not reached through this sequence. Earlier effects have no represented undo mechanism.

The decoding boundary does not establish a particular runtime exception, trap, halt instruction, or process-termination behavior.

## Dependencies

“No explicit arguments” describes only the visible representation. It does not prove a zero-argument machine interface. Likewise, an unconsumed result does not establish a void return type.

| Opaque IDs | Explicit interface visible in this prefix |
|---|---|
| **F-U01** | One 32-bit argument: second explicit input |
| **F-U02** | No explicit arguments; 64-bit result consumed |
| **F-U03** | Four 32-bit arguments derived from **F-U02** |
| **F-U04–F-U06** | No explicit arguments; no result consumed |
| **F-U07** | No explicit arguments; 32-bit result consumed |
| **F-U08** | One argument: four times the **F-U07** result |
| **F-U09–F-U11** | No explicit arguments; no result consumed |
| **F-U12** | No explicit arguments; unsigned 32-bit result consumed |
| **F-U13–F-U14** | No explicit arguments; no result consumed |
| **F-U15** | No explicit arguments; 32-bit result consumed |
| **F-U16** | One argument: **F-U15** result shifted left by two |
| **F-U17–F-U19** | No explicit arguments; no result consumed |
| **F-U20** | No explicit arguments; 32-bit result consumed |
| **F-U21** | Four arguments derived from retained **F-U12** and **F-U20** results |
| **F-U22–F-U24** | No explicit arguments; no result consumed |
| **F-U25** | No explicit arguments; 32-bit result consumed |
| **F-U26** | One argument: **F-U25** result shifted left by two |
| **F-U27–F-U30** | No explicit arguments; no result consumed |

Dependency implementations, target validity, shared identities, hidden inputs, side effects, and ownership contracts remain **documented_only**.

## Acceptance examples

These are conditional static expectations, not executed tests. Each assumes compatible entry state, valid relevant accesses, and dependencies returning through the described prefix. Unspecified dependency results must permit any intervening arithmetic relevant to the example.

### Split-result forwarding

If **F-U02** supplies low half `0x11223344` and high half `0x80000008`, **F-U03** receives:

1. `0x11223344`
2. `0x80000008`
3. `0x11223344`
4. `0x10000001`

A sign-extending right shift would violate the established forwarding behavior.

### Payload placement

If **F-U07** returns 3, the payload write starts at address `0x20` and spans four bytes. **F-U08** subsequently receives 12.

The payload write precedes **F-U08**. Its value comes from the implicit payload, not an established relationship to the first explicit input.

### Independently selected derived-byte destinations

If **F-U12** returns `0x00100003`, the retained scaled word is `0x0040000c`. The two derived byte values are `0x0c` and `0x01`.

If the first base-relative read yields `0x20`, the first byte is written to `0x31`. If the second read yields `0x40`, the second byte is written to `0x51`.

This example establishes those earlier writes only. It does not assert overflow-free execution of the later scaled-address multiplication.

### Repeated zero writes without overflow

For this example, **F-U12** returns 3. The retained scaled word is 12, and the later lookup address is:

12 multiplied by `0x400`, plus one, giving `0x3001`.

This calculation is within the signed 32-bit range.

If the first read at `0x3001` yields `0x10`, the first zero write targets `0x21`. If intervening dependencies change that byte to `0x11`, the second zero write targets `0x22`.

The second read must observe the later contents rather than reuse `0x10`.

### Larger result under an explicit wrapping assumption

If **F-U12** instead returns `0x00100003`, multiplying its scaled word `0x0040000c` by `0x400` produces the mathematical value `0x100003000`.

That value exceeds both the signed 32-bit maximum and the unsigned 32-bit maximum. The lookup address `0x3001` follows only under the explicitly assumed 32-bit wrapping interpretation: retain `0x3000`, then add one.

Without that assumption or additional target evidence, this input’s later lookup address remains **documented_only**.

### Lookup-byte boundary

A destination-selection byte of `0xff` produces address `0x110`, not `0x10`. The byte is loaded unsigned, and the subsequent address addition is not an eight-bit addition.

### Interrupted prefix

If the payload write prevents further execution, **F-U08** and subsequent events are not reached through the represented sequence. No alternate recovery path is established.

## Uncertainties

The following remain **documented_only**:

- Whether the representation begins at a genuine callable entry and correctly decodes the entire prefix.
- The actual calling convention and completeness of the apparent two-input interface.
- Origins and validity of implicit entry state.
- Dependency identities, hidden arguments, memory effects, and return behavior.
- Target address width, address wrapping, signed-overflow behavior, and large unsigned-to-signed conversions.
- Endianness, alignment, atomicity, and ordinary-memory versus device-memory interpretation.
- Preservation of machine state required for the represented continuation.
- Ownership, lifetime, synchronization, allocation, and release requirements.
- Any normal completion or behavior beyond the decoding boundary.

An inferred intent is coordination of lower-level operations while emitting compact values. This is not an established semantic purpose, ownership model, or data-format requirement.

## Evidence

Established static evidence consists of the supplied representation’s two explicit 32-bit inputs, thirty ordered invocation sites, one consumed 64-bit result, consumed 32-bit results, four unsigned byte reads, one 32-bit write, and four byte writes.

The represented data flow retains results across intervening calls, performs fresh repeated lookups, and contains no explicit normal return, recovery branch, or rollback before the invalid-instruction boundary.

The overflow distinction follows from the represented widths and arithmetic: `0x0040000c` multiplied by `0x400` is `0x100003000`, whereas 12 multiplied by `0x400` is `0x3000`.

No dependency implementations, authoritative ABI, memory map, or runtime observations were supplied. No runtime tests have been performed.
