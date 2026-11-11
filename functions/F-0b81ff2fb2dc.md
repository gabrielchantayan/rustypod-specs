<!-- Trusted provenance: F-0b81ff2fb2dc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0b81ff2fb2dc

## Purpose

The available static evidence describes an effectful sequence of indirect calls, byte writes, word writes, and computed-address reads. It does not establish a domain-level purpose.

The visible sequence:

- Writes two inherited byte values into separate records.
- Stores scaled dependency results into word-sized locations.
- Uses dependency results both as numeric values and as address components.
- Performs byte writes through addresses selected by previously read bytes.
- Ends at an explicitly marked invalid-instruction boundary rather than an established normal return.

This contract documents that sequence without interpreting it as initialization, allocation, serialization, device control, or another higher-level operation.

The evidence carries a control-flow warning. Consequently, the behavior below is a **conditional contract for the represented path**, not proof that a valid executable reaches every operation in that order.

## Interface

### Explicit interface

The recovered interface has no explicit parameters and no declared return value. That is insufficient to establish a conventional zero-argument calling convention: the body consumes inherited machine state.

The following ambient inputs are visible:

| Input | Visible use |
|---|---|
| First record base | Destination of a byte write at offset `0x0d`. |
| Second record base | Destination of a byte write at offset `0x0d`, destination of a word write at offset `0x24`, and source of a word read at offset `0x74`. |
| Inherited byte value | Written at the first record base plus `0x0d`. |
| Inherited word value | Its low byte is written at the second record base plus `0x0d`; its full word is later written to a computed address. |
| Inherited control-address value | Used to form the first indirect-call target. |
| Memory and dependency state | Supplies later reads, indirect-call behavior, and returned values. |

The first record base, second record base, inherited values, and inherited control-address value are descriptive contract names, not established public parameter names.

### Widths and representation

The representation distinguishes:

- **Byte:** 8 bits.
- **Word:** 32 bits.
- **Paired result:** 64 bits, explicitly separated into low and high 32-bit halves.

Byte stores retain only the low eight bits of their supplied value. Byte reads used as address selectors are unsigned: their values range from 0 through 255.

For clarity, arithmetic below uses a conditional 32-bit bit-vector interpretation:

- A left shift retains the low 32 bits.
- Multiplication by a power of two likewise retains the low 32 bits.
- Right shifts of explicitly unsigned values are logical.
- Address additions use the represented address width, taken here as 32 bits.

This interpretation describes the visible operations without importing signed-language overflow rules. Actual machine arithmetic, address validity, and alignment requirements remain unverified.

### Outputs and completion

There is no established normal return value.

Observable outputs consist of direct memory writes and any effects of the indirect dependencies. The final visible event is an invalid-instruction termination marker. The evidence does not establish whether an actual execution traps, is otherwise terminated, or continues through instructions absent from the representation.

### Preconditions

For the entire represented path to execute:

- Every indirect target must be executable under the actual calling convention.
- Each dependency must complete sufficiently for the next visible operation to occur.
- Every direct read and write must reference accessible memory with any required alignment.
- Returned values used to construct addresses must produce usable addresses.
- Execution must reach the represented blocks despite the control-flow warning.

No validation of these conditions is visible.

## Behavior

### Dependency notation

Indirect call sites receive opaque function IDs `F-c001` through `F-c046`, in visible execution order. These are **call-site dependency identities**, not proof of 46 distinct functions.

“Arguments” below means arguments explicitly shown in the supplied representation. An empty argument list does not establish that a dependency consumes no ambient registers or other state. An unused result means only that no explicit result is consumed by this body.

### 1. Initial dispatch and inherited-byte writes

1. Invoke `F-c001`, whose target is formed from the inherited control-address value plus a fixed displacement. Retain its word result.
2. Write the low byte of the inherited word value to the second record base plus `0x0d`.
3. Write the inherited byte value to the first record base plus `0x0d`.
4. Invoke `F-c002` with the result of `F-c001` shifted left by two bits.
5. Invoke `F-c003`, `F-c004`, and `F-c005`, in that order, with no explicit arguments.
6. Invoke `F-c006` with no explicit arguments and retain its word result.

The first dependency executes before either direct byte write. If the two byte destinations alias, the write to the first record is later and therefore determines that byte immediately after both stores.

### 2. Record update and second-base extraction

Using the result from `F-c006`:

1. Write that result shifted left by two bits to the second record base plus `0x24`.
2. Read a word from the second record base plus `0x74`. Retain it as the **derived base**.
3. Retain the result from `F-c006` shifted left by five bits as the **scaled word**.
4. Invoke `F-c007` with four explicit word arguments, in this order:
   - The derived base.
   - The derived base shifted left by one bit.
   - The scaled word.
   - The derived base masked with `0x7fffffff`, then logically shifted right by two bits.
5. Invoke `F-c008`, `F-c009`, `F-c010`, and `F-c011`, in order, with no explicit arguments.
6. Write the scaled word to the derived base plus `0x24`.
7. Read a word from the derived base plus `0x74` and pass it as the single explicit argument to `F-c012`.
8. Invoke `F-c013`, `F-c014`, and `F-c015`, in order, with no explicit arguments.

The derived base is a retained word, not a pointer freshly loaded before each later access. Dependencies may nevertheless change memory at addresses computed from it.

The read supplying `F-c012` occurs after `F-c007` through `F-c011` and after the direct store at the derived base plus `0x24`. It must not be treated as an earlier snapshot.

### 3. Paired results and indexed word stores

1. Invoke `F-c016` with no explicit arguments and obtain a paired result.
2. Separate that result into low and high words.
3. Retain the low word shifted left by two bits as the **indexed payload**.
4. Write the full inherited word value to the address formed by multiplying the high word by four and adding `0x14`.
5. Invoke `F-c017` with four explicit arguments:
   - Indexed payload.
   - Indexed payload again.
   - Indexed payload again.
   - The high word logically shifted right by three bits.
6. Invoke `F-c018` with no explicit arguments and obtain another paired result.
7. Invoke `F-c019` with:
   - The second paired result’s low word.
   - Its high word.
   - Its low word again.
   - Its high word logically shifted right by three bits.
8. Invoke `F-c020`, `F-c021`, and `F-c022`, in order, with no explicit arguments.
9. Invoke `F-c023` with no explicit arguments and retain its word result.
10. Write the indexed payload to the address formed by multiplying the result of `F-c023` by four and adding `0x14`.
11. Invoke `F-c024` with the result of `F-c023` multiplied by four.
12. Invoke `F-c025`, `F-c026`, and `F-c027`, in order, with no explicit arguments.

Both computed word-store destinations are absolute expressions in the representation. No additional table base or enclosing allocation is visible.

### 4. Two byte writes selected through the derived base

1. Invoke `F-c028` with no explicit arguments and retain its word result as the **selector word**.
2. Retain the selector word shifted left by two bits as the **shifted selector**.
3. Read an unsigned byte from the derived base plus one.
4. Add `0x11` to that byte’s numeric value and write the low byte of the shifted selector to the resulting address.
5. Invoke `F-c029` and `F-c030`, in order, with no explicit arguments.
6. Invoke `F-c031` with no explicit arguments and retain its word result.
7. Read an unsigned byte from the derived base plus one **again**.
8. Add `0x11` to this newly read byte and write the low byte of the selector word masked with `0x3fffffff` and shifted right by 20 bits.
9. Invoke `F-c032` with the result of `F-c031` shifted left by two bits.
10. Invoke `F-c033`, `F-c034`, and `F-c035`, in order, with no explicit arguments.

The second stored byte is exactly bits 20 through 27 of the selector word. Bits 28 and 29 can survive the intermediate mask and shift, but are discarded by the final byte store.

Each destination is the byte selector’s numeric value plus `0x11`, not the derived base plus that amount. Thus each destination lies between `0x11` and `0x110`, inclusive, under the stated arithmetic interpretation.

The two reads of the selector byte are distinct observable reads. Intervening dependency effects or aliasing can cause the two stores to target different addresses.

### 5. Two zero stores selected through a computed read address

1. Invoke `F-c036` with no explicit arguments and retain its word result.
2. Form a read address by multiplying the shifted selector by `0x400` and adding one.
3. Read an unsigned byte at that address.
4. Add `0x11` to the byte’s numeric value and write zero to the resulting byte address.
5. Invoke `F-c037` with:
   - The shifted selector shifted left by three bits.
   - That same shifted value again.
   - The shifted selector.
   - The result of `F-c036` shifted left by two bits.
6. Invoke `F-c038`, `F-c039`, and `F-c040`, in order, with no explicit arguments.
7. Invoke `F-c041` with no explicit arguments and retain its word result.
8. Read an unsigned byte again at the computed read address described above.
9. Add `0x11` to the newly read byte and write zero to the resulting byte address.
10. Invoke `F-c042` with the result of `F-c041` shifted left by two bits.
11. Invoke `F-c043`, `F-c044`, `F-c045`, and `F-c046`, in order, with no explicit arguments.
12. Reach the invalid-instruction boundary shown by the evidence.

The two zero-store destinations need not match. Although the numeric read address is retained by the represented computation, its contents may change between reads.

Under 32-bit modular arithmetic, the computed read address is also the selector word shifted left by 12 bits, plus one. This equivalence does not establish address validity.

## Side effects

### Direct memory effects

The visible sequence contains ten direct stores:

- Four byte stores involving inherited values or parts of the selector word.
- Four 32-bit word stores.
- Two additional byte stores of zero.

It contains six direct reads:

- Two word reads at offset `0x74`.
- Two byte reads at the derived base plus one.
- Two byte reads at the computed selector-derived address.

Dependency instruction fetches and dependency-internal memory accesses are not included in these counts.

No explicit allocation, deallocation, ownership transfer, lock operation, or resource-release operation is visible. None can be ruled out inside dependencies.

### Ordering and aliasing

Direct operations occur in the order stated. There is no visible rollback, transaction boundary, or recovery mechanism.

No non-aliasing guarantee is established. Record fields, computed word destinations, selector-read locations, and low-address byte destinations may overlap. Later reads must observe memory as it exists at their own execution points, including intervening direct and dependency-induced effects.

No atomicity, synchronization, volatility, or concurrent-access guarantee is established. If a dependency does not return, a memory access faults, or execution terminates early, preceding effects may already have occurred.

## Dependencies

All dependencies are opaque indirect calls.

| Opaque function IDs | Explicit result consumed |
|---|---|
| `F-c001`, `F-c006` | Word results used in scaled arguments or stores. |
| `F-c016`, `F-c018` | Paired results split into low and high words. |
| `F-c023` | Word result used as a scaled destination and argument. |
| `F-c028` | Word result used in selector calculations and byte stores. |
| `F-c031`, `F-c036`, `F-c041` | Word results used in later shifted arguments. |
| All remaining IDs through `F-c046` | No explicit result consumed. |

The first target depends on inherited control state. Later targets are formed from fixed code-related values and displacements. The evidence does not establish their runtime identity, validity, signatures, or whether different call sites resolve to the same implementation.

No dependency is assumed to preserve globals, preserve arbitrary memory, allocate memory, release memory, or honor an ownership convention.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests. They assume reachable control flow, valid accesses, returning dependencies, and the 32-bit interpretation stated above.

1. **Inherited-byte truncation and ordering.**  
   If the inherited word is `0x123456ab`, the second record’s byte at `0x0d` receives `0xab`. The first record’s byte write follows it. If both destinations coincide and the inherited byte is `0x34`, the resulting byte immediately after the two stores is `0x34`.

2. **Two different scalings of one result.**  
   If `F-c006` returns three, the word written at the second record plus `0x24` is 12. The scaled word supplied to `F-c007` and later written at the derived base plus `0x24` is 96.

3. **Mask before right shift.**  
   If the derived base is `0x80000008`, the fourth argument to `F-c007` is two. It is not the result of shifting the unmasked derived base.

4. **Paired-result separation.**  
   If `F-c016` returns a low word of five and a high word of eight, the indexed payload is 20. The inherited word is written at address `0x34`, and `F-c017` receives 20, 20, 20, and one.

5. **Selector truncation.**  
   If `F-c028` returns `0x12345678`, the first selector-based byte store writes `0xe0`; the second writes `0x23`. If both selector reads return nine, both destinations are `0x1a`, with intervening calls between the stores.

6. **Selector rereads are significant.**  
   If the first derived-base selector read returns two and the second returns seven, the two byte stores target `0x13` and `0x18`, respectively. Reusing the first read would violate the represented behavior.

7. **High-bit loss in the computed read address.**  
   If the selector word is `0x40000000`, its two-bit left shift becomes zero under the stated arithmetic model. The computed read address for each later zero-store selector read is therefore one. Whether that read is permitted is a separate precondition.

8. **Repeated zero writes remain distinct.**  
   If the first computed selector read returns four and the second returns six, zero is written first at `0x15` and later at `0x17`. Eliminating the second read or store would change the represented observable sequence.

## Uncertainties

The following remain **documented_only**:

- Whether the recovered control flow corresponds to valid executable instructions.
- The actual calling convention and status of inherited machine values as inputs.
- Exact machine address width, alignment rules, memory mapping, and fault behavior.
- Dependency identities, complete arguments, effects, ownership behavior, and termination.
- Whether low numeric destinations denote ordinary memory, device registers, or inaccessible locations.
- Whether computed word destinations denote valid objects.
- Whether any direct accesses alias.
- Whether execution can reach the full sequence and its terminal boundary.
- What the invalid-instruction boundary means for actual execution.

A higher-level interpretation as record manipulation or indexed storage is plausible but unestablished. It adds no contractual guarantee.

## Evidence

**Established static evidence:** the supplied representation shows inherited-state use, 46 indirect call sites, the ordered direct accesses described above, explicit byte and word operations, two paired-result extractions, masking and shifting operations, and a terminal invalid-instruction marker. It also explicitly warns of invalid instruction data during control-flow recovery.

**Inferred modeling choices:** descriptive input names, call-site dependency IDs, and the conditional 32-bit modular arithmetic model organize the evidence. They do not establish an external API, object schema, dependency implementation, or application purpose.

**Verification status:** no runtime tests have been performed. No executable behavior, dependency implementation, memory mapping, or normal completion has been verified.
