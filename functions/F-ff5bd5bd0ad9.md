<!-- Trusted provenance: F-ff5bd5bd0ad9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ff5bd5bd0ad9

## Purpose

F-ff5bd5bd0ad9 performs an ordered sequence of mixed-width memory accesses, seven global-value handoffs, arithmetic-derived accesses, activation-storage assignments, and dependency calls. Its represented endpoint is nonreturning.

No application-level purpose is established. Repeated offsets suggest structured state propagation, but do not establish initialization, allocation, cleanup, serialization, or a particular data structure.

This contract describes the supplied static representation. It does not establish that the recovered interface, arithmetic-derived addresses, or terminal mechanism faithfully describe a callable machine-level routine. No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface has seven argument positions and no returned value.

| Position | Width | Established use |
|---|---:|---|
| First | 32 bits | Base for repeated field reads and a byte write; source of shifted values assigned to activation storage. |
| Second | 32 bits | Initial stored value and address component for an early byte read. |
| Third | 32 bits | Incoming value is not explicitly consumed. |
| Fourth | 8 bits | Payload of the initial byte store. |
| Fifth | 32 bits | Its storage address is passed to F-unknown-indirect-1. |
| Sixth | 32 bits | Incoming value is not explicitly consumed. |
| Seventh | 32 bits | Its represented storage is assigned zero after F-unknown-indirect-1 returns. |

An absent explicit read does not establish inaccessibility through aliasing or through the address supplied to a dependency.

### Ambient inputs

Additional required state consists of:

- Two incoming 32-bit address-like values, called **ambient A** and **ambient B**.
- An incoming byte.
- An unresolved indirect call target.
- Seven distinct 32-bit global storage locations, numbered by consumption order.
- A separate global value used after the indirect call.
- Activation storage whose relationship to the represented arguments is unresolved.

The numbered globals are not established as an array or object family. Their contents are read during execution, not captured together at entry.

### Preconditions

There is no represented validation. Every reached access requires suitable addressability, permissions, and any required alignment. This includes addresses constructed from narrow signed values, promoted bytes, sums, and products.

No ordinary return is represented. Faults or nonreturning dependencies can prevent later effects.

## Behavior

### Widths, arithmetic, and ordering

Offsets are in bytes. Memory operations distinguish 8-, 16-, and 32-bit widths. A narrow store does not replace the surrounding 32-bit field.

Signed 16-bit reads contribute signed values to address calculations. Plain-byte signedness is unresolved: a byte containing all one bits may contribute either 255 or −1 when promoted. Widening that byte to 16 bits consequently need not produce the same stored pattern under both interpretations.

The representation uses 32-bit integer-like values. Addition, multiplication, doubling, and left shifts must not be treated as unbounded arithmetic. Portable modulo arithmetic is not established for signed overflow or invalid shifts. Byte order, alignment, and atomicity are undocumented.

The ordering below is material because addresses may alias. A retained value is distinguished from a later read of the same location. “Low byte” and “16-bit form” identify the represented narrow payload; out-of-range signed conversion semantics remain dependent on the execution model.

### Entry and first handoff inputs

The first effect is a 32-bit store of the second argument at ambient A + 20. A byte store of the fourth argument at ambient B + 17 follows.

A signed 16-bit read at ambient B + ambient A supplies the address component for a byte read at that signed value plus the second argument. The incoming ambient byte is then stored at ambient A + 29, before the byte just read is stored in 16-bit form at the signed value plus ambient A.

A 32-bit read at ambient A + 68 supplies the first **transfer word**. The early byte-derived value supplies the first **link destination**.

### Seven handoffs

Every numbered handoff has the following ordered obligations:

1. Store the continuing base itself as 32 bits at continuing base + 84.
2. Read and retain the value of the corresponding numbered global.
3. Store a value sourced again from that same global as 32 bits at link destination + 68.
4. Store the retained transfer word as 32 bits at retained global value + 20.
5. Store the specified byte payload at continuing base + 17.

The offset-84 self-store precedes consumption of the numbered global. If those locations alias, that store can change the global value subsequently consumed. The retained global value and the later global source expression are distinct represented uses; the representation does not establish their exact machine-load count.

The continuing base and final byte payload are:

| Handoff | Continuing base | Payload at continuing base + 17 |
|---|---|---|
| 1 | Ambient B | Low byte of doubled ambient B |
| 2 | Ambient B | Low byte of doubled ambient B |
| 3 | Ambient B | Low byte of the second read of first argument + 100 |
| 4 | Middle base, defined below | Low byte of the third read of first argument + 100 |
| 5 | Middle base | Low byte of doubled middle base |
| 6 | Next base, defined below | Low byte of the fourth read of first argument + 100 |
| 7 | Final base, defined below | Low byte of the fifth read of first argument + 100 |

The entry supplies handoff 1’s transfer word and link destination. The following stages supply those for handoffs 2–7.

### After handoff 1

A signed 16-bit read uses ambient B plus the first retained global value. A dependent byte read uses that signed value plus handoff 1’s transfer word.

The first read of first argument + 100 supplies a 32-bit store at signed value + 84. A byte store at newly byte-derived value + 13 then writes the low byte of handoff 1’s transfer word.

Only afterward is a 32-bit value read from first retained global value + 100. This **saved hundred-field value** remains a later payload. A 16-bit store at signed value plus first retained global value uses the byte-derived value.

A subsequent 32-bit read at first retained global value + 68 supplies handoff 2’s transfer word. The newly byte-derived value is handoff 2’s link destination.

### After handoff 2

The second read of first argument + 100 occurs before a signed 16-bit read at ambient B plus the second retained global value. A dependent byte read uses that signed value plus handoff 2’s transfer word.

The intervening writes occur in this order:

- First argument + 1 receives the low byte of the second retained global value.
- Second retained global value + 13 receives the low byte of handoff 2’s transfer word.
- Second retained global value + 68 receives the saved hundred-field value as 32 bits.
- Newly byte-derived value + 13 receives the low byte of the second retained global value.
- Signed value plus second retained global value receives the byte-derived value in 16-bit form.

A separate 32-bit read at second retained global value + 68 supplies handoff 3’s transfer word. It is not interchangeable with the earlier store payload under aliasing. The new byte-derived value supplies handoff 3’s link destination.

### After handoff 3

The third read of first argument + 100 is followed by a 32-bit read at ambient B + 68, defining the **third-stage field base**. A signed 16-bit read uses ambient B plus the third retained global value.

A 32-bit read at handoff 3’s transfer word + 116 supplies a derived base. Then ambient B + 20 is read as 32 bits, defining the **middle base**.

Third-stage field base + 13 receives the low byte of the derived base. Signed value plus derived base receives the third-stage field base in 16-bit form. A 32-bit read at derived base + 68 supplies handoff 4’s transfer word; the third-stage field base supplies its link destination.

Handoff 4 uses the middle base for both its offset-84 self-store and its offset-17 byte store.

### After handoff 4

A 32-bit read at middle base + 68 defines the **fourth-stage field base**. A signed 16-bit read uses middle base plus the fourth retained global value.

Handoff 4’s transfer word + 21 receives the low byte of the fourth retained global value. Fourth retained global value + 68 then receives that transfer word as 32 bits.

A subsequent 32-bit read at fourth-stage field base + 116 supplies another derived base. Signed value plus that derived base receives the fourth-stage field base in 16-bit form.

A 32-bit read at derived base + 68 supplies handoff 5’s transfer word. Its link destination is the fourth-stage field base.

### After handoff 5

The fourth read of first argument + 100 precedes a 32-bit read at middle base + 68, defining the **fifth-stage field base**. A signed 16-bit read uses middle base plus the fifth retained global value.

A 32-bit read directly at signed value + 20 defines the **next base**. No intervening pointer-field dereference is implied.

Fifth-stage field base + 84 receives that field base itself as 32 bits. A 32-bit store at fifth retained global value plus handoff 5’s transfer word writes the saved hundred-field value. Fifth retained global value + 20 then receives the fifth-stage field base as 32 bits.

A fresh 32-bit read at saved hundred-field value + 68 supplies the byte stored at fifth retained global value + 13. Next base + 13 receives the low byte of the fifth-stage field base. Signed value plus fifth retained global value receives that field base in 16-bit form.

A 32-bit read at fifth retained global value + 68 supplies handoff 6’s transfer word. Its link destination is the fifth-stage field base.

### After handoff 6

The fifth read of first argument + 100 supplies a byte retained through handoff 7. A 32-bit read at next base + 68 defines the **sixth-stage field base**. A signed 16-bit read uses next base plus the sixth retained global value.

Before the next signed-value-based lookup:

- Handoff 6’s transfer word + 84 receives the next base as 32 bits.
- Next base + 17 receives the low byte of the fifth hundred-field read.
- Sixth-stage field base + 68 receives the sixth retained global value as 32 bits.

A 32-bit read at signed value + 20 defines the **final base**. Sixth-stage field base + 84 then receives that field base itself.

A 32-bit store at sixth retained global value plus handoff 6’s transfer word writes the saved hundred-field value. Sixth retained global value + 20 receives the sixth-stage field base. A fresh 32-bit read at saved hundred-field value + 68 supplies the byte stored at sixth retained global value + 13.

Final base + 13 receives the low byte of the sixth-stage field base. Signed value plus sixth retained global value receives that field base in 16-bit form.

A separate 32-bit read at sixth retained global value + 68 supplies handoff 7’s transfer word. Its link destination is the sixth-stage field base.

### Product-derived phase

After handoff 7, a 32-bit read at final base + 68 supplies a multiplier operand. A signed 16-bit read at final base plus the seventh retained global value supplies the **signed factor**.

The following pair occurs twice, in alternating order: a 32-bit store of the signed factor at signed factor + 84, then a byte store of the final base at seventh retained global value + 17.

The product of the signed factor and multiplier operand supplies an offset-68 read. That result supplies an offset-116 read, whose result supplies an offset-20 read. At the resulting destination base, the existing offset-20 value is saved before replacement by doubled final base as 32 bits.

The saved old value supplies another offset-20 read. Its result, multiplied by the signed factor, defines the **first product**.

Another signed-factor self-store at offset 84 precedes a byte store of the saved old value at seventh retained global value + 17. A 32-bit read at first product + 116 supplies the **retained product field**, assigned to activation offset 264.

Multiplying the first product by the signed factor defines the **second product**. Another signed-factor self-store precedes a byte store of the retained product field at seventh retained global value + 17.

The remaining ordered effects are:

- Read 32 bits at second product + 68, then at first argument + 116.
- Store the retained product field as 32 bits at seventh retained global value + 100.
- Store the low byte of the preceding product-derived offset-68 read at seventh retained global value + 9.
- Read second product + 68 again, using its low byte for a store at second product + 17.
- Repeat the signed-factor self-store, then store the low byte of the retained product field at doubled second product + 17.
- Read 32 bits at signed factor multiplied by second product, plus 68; then read 32 bits at first argument + 20.
- Store the low byte of the saved first-argument offset-116 read at retained product field + 1.
- At the base read from first argument + 20, store the low byte of the final product-derived read at offset 17, then the low byte of doubled second product at offset 9.

The two reads at second product + 68 remain separate.

### Activation storage and calls

Activation offsets describe the representation, not a verified ABI layout.

Before the indirect call, assignments occur in this order:

| Offsets | Assigned values |
|---|---|
| 16, then 52 | First argument shifted left by four, then eight bits |
| 228, 96, 168, 440 | Zero, in that order |
| 36, then 76 | Previously prepared four-bit and eight-bit shifted values |
| 296 | Retained product field |
| 316 | Prepared eight-bit shifted value |
| 380 | Retained product field |
| 392 | Prepared eight-bit shifted value |

Offset 264 was assigned earlier, during the product-derived phase. These are assignment obligations, not a guarantee that aliasing cannot affect the resulting storage.

F-unknown-indirect-1 receives the address of the fifth argument’s storage and two zeros. No returned value is explicitly consumed.

If it returns, the seventh argument’s represented storage is assigned zero. Activation offset 412 receives 0x08400e1c; offsets 256, 288, and 320 then receive zero, in that order.

The separate global argument is consumed only afterward. F-cdec09f5d432 receives that value and three zeros.

If it returns, F-unknown-terminal-mechanism-1 receives zero and 0x08401154. Its represented result supplies an indirect transfer target. This endpoint is marked nonreturning; it is not an ordinary success return or proof of a particular exception.

## Side effects

Writes reach argument-derived, ambient-derived, global-derived, narrow-integer-derived, product-derived, and activation-storage addresses.

No non-aliasing guarantee is established. A completed write may change a later global read, repeated field read, address component, or only part of a wider value. Repeated stores remain represented effects even when their payloads match.

No synchronization, atomicity, rollback, structured error result, logging, allocation, or release is explicitly identified. A fault can leave a completed prefix of mutations.

Passing the fifth argument’s storage address establishes access, not ownership transfer, release authority, retention permission, or lifetime extension. Dependency effects remain unresolved.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-unknown-indirect-1 | Incoming unresolved target; receives the fifth argument’s storage address and two zeros. |
| F-cdec09f5d432 | Receives a global value consumed after the first call and subsequent assignments, followed by three zeros. |
| F-unknown-terminal-mechanism-1 | Receives zero and a fixed location-like operand; supplies the represented final transfer target. |
| F-unknown-terminal-target-1 | Target of that transfer; no independent argument or return contract is established. |

These IDs identify roles, not resolved implementations. Unknown callees are not assumed to preserve globals or memory, return normally, or refrain from allocating or releasing storage.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Entry order:** The second argument’s 32-bit store at ambient A + 20 precedes the fourth argument’s byte store at ambient B + 17.

2. **Global aliasing:** If a handoff’s continuing-base offset-84 destination aliases its numbered global, the retained global value reflects the preceding self-store. An entry-time global snapshot is not equivalent.

3. **Handoff byte payloads:** Handoffs 1–2 use doubled ambient B; handoffs 3, 4, 6, and 7 use the second, third, fourth, and fifth hundred-field reads respectively; handoff 5 uses doubled middle base.

4. **Second-stage mixed widths:** Suppose the second retained global value is 8192, handoff 2’s transfer word is 16384, and the signed read yields −2. The dependent byte read uses address 16382. After the intervening writes, its 16-bit-form store uses address 8190, provided all accesses and arithmetic are valid.

5. **Byte interpretation:** An all-one-bits byte can widen to 255 or −1 under different plain-byte interpretations. Acceptance must identify the applicable model.

6. **Repeated hundred-field reads:** If an intervening write changes first argument + 100, the next of its five reads observes the changed contents. Caching the first read is not generally equivalent.

7. **Product-phase repetition:** Both initial signed-factor self-stores and both alternating seventh-global offset-17 stores occur. The later second-product offset-68 read is likewise not replaced by its earlier result.

8. **Post-call global consumption:** If F-unknown-indirect-1 changes the separate global and returns, F-cdec09f5d432 receives the subsequently consumed value.

9. **Partial completion:** A fault does not require restoration of completed writes. A nonreturning first dependency prevents all represented post-call assignments and calls.

## Uncertainties

The following remain **documented_only**:

- Application purpose, field meanings, function-boundary accuracy, and calling convention.
- Ambient-state origins and activation-storage origin, extent, and lifetime.
- Concrete global contents and memory mappings.
- Plain-byte signedness, byte order, alignment, signed narrowing, overflow, and shift behavior.
- Whether arithmetic-derived accesses accurately represent executable behavior.
- Exact machine-load counts for repeated global source expressions.
- Dependency mutations, preservation, ownership, retention, and return behavior.
- Meaning of shifted activation values and fixed operands.
- Runtime meaning of the terminal mechanism and behavior omitted from the representation.

These unknowns do not justify repairing unusual addresses or assuming conventional objects.

## Evidence

**Established static evidence:** The supplied body represents seven numbered global handoffs, each with an offset-84 self-store before global consumption and an offset-17 byte store afterward. It includes the second-stage dependent signed-16-bit read, byte read, intervening writes, and 16-bit store; five reads at first argument + 100; chained products; activation assignments; two ordinary calls; and a terminal indirect transfer.

**Inferred intent:** Repeated offsets and propagated values may indicate structured runtime-state manipulation. This is not an acceptance requirement.

**Evidence limits:** The supplied material reports omitted unreachable regions and annotates the endpoint as nonreturning. Neither annotation independently proves runtime behavior. No execution trace, dependency implementation, ABI documentation, or runtime test result was available.
