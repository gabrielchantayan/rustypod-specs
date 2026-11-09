<!-- Trusted provenance: F-d9a1b83bbdc8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d9a1b83bbdc8

## Purpose

**Established static behavior:** F-d9a1b83bbdc8 issues an ordered sequence of **26 four-byte stores**, interleaved with nine dependency invocations. After the last store, it enters an unconditional infinite loop. The supplied body contains no normal return, explicit error result, or recovery path.

Inputs and dependency results serve both as data and as numerical components of memory addresses. Several required values come from incoming machine state outside the four declared arguments.

**Inferred intent:** The repeated scaled values and fixed offsets may relate to initialization or state construction. The evidence does not establish an object model, buffer layout, or application-level purpose. This contract therefore specifies observable accesses and control flow without assigning such meanings.

All behavior below is conditional on execution reaching the relevant operation. A faulting access, nonreturning dependency, or external control transfer can prevent later operations.

## Interface

### Explicit inputs

The supplied representation exposes four arguments:

| Contract name | Width and representation | Uses |
|---|---|---|
| **token** | Opaque 32-bit word | Argument to the first dependency |
| **base-one** | Signed 32-bit integer | Address component for the first store |
| **base-two** | Signed 32-bit integer | Address component, stored payload, and dependency argument |
| **quantity** | Unsigned 32-bit integer | Stored payload, address component, and source of logical shifts |

These names are contract notation, not recovered source identifiers.

### Additional incoming state

The body also consumes values not visibly initialized within it:

| Contract name | Visible role |
|---|---|
| **saved-word** | Incoming opaque 32-bit payload for the first store |
| **auxiliary-base** | Incoming signed integer used as an address component |
| **shared-pointer** | Incoming pointer-shaped value used as both a destination and a stored payload |
| **incoming-link** | Incoming address-like value used to construct the first invocation target |
| **stack-word** | Opaque 32-bit argument obtained from an incoming stack location represented at offset `0x168` |

The complete calling convention, stack-offset reference point, and means of supplying this additional state are **documented_only**.

The stack-word is consumed at the direct dependency invocation. The representation does not establish a separate entry-time load or guarantee that earlier callees cannot affect the underlying stack storage.

### Widths and notation

The following derived values simplify the contract:

- **quarter**: quantity logically shifted right by two.
- **result**: the unsigned 32-bit result of D-04.
- **result-quarter**: result logically shifted right by two.
- **result-sixteenth**: result logically shifted right by four.
- **handle**: the signed 32-bit result of D-07.
- **final-base**: the signed 32-bit result of D-08.

Logical right shifts discard low bits and insert zero bits. Thus quarter is the integer quotient of quantity divided by four, rounded down.

All 26 stores are represented as four-byte writes, including stores carrying shared-pointer. A 32-bit pointer/address model is indicated by the casts and pointer-sized writes. Byte order and actual memory-access alignment requirements are not supplied.

Offsets are in bytes. Expressions such as result-quarter + base-two do not imply an additional element-size multiplier.

The arithmetic has distinct width constraints:

- Operations represented as unsigned 32-bit arithmetic, including quantity scaling and `4 × result + 0x14`, retain only the low 32 bits.
- Unsigned right shifts have the logical meaning stated above.
- Signed expressions, including `2 × base-two`, handle shifted left by two, and subtraction from the signed low word of D-02’s result, do not establish a general overflow rule from this representation alone.
- Integer-to-address conversion, out-of-range signed arithmetic, and invalid pointer arithmetic remain **documented_only** where the underlying execution semantics are unavailable.

The address formulas below describe the represented accesses; they do not certify that those addresses are valid.

### Output and preconditions

There is no visible returned value. If all dependencies return and all accesses succeed, the terminal behavior is nontermination in the final loop.

No validation is performed. Progress requires every reached invocation target to be callable and every reached memory access to be permitted by the execution environment. No disjointness, allocation ownership, minimum extent, or alignment guarantee is established for the inputs.

## Behavior

The following phases preserve the visible order of top-level accesses and calls. They do not prescribe operand-evaluation order within a single expression or the internal behavior of dependencies.

### Initial store and split dependency result

The first operation stores saved-word at **base-one + quantity**. This precedes every dependency invocation.

The routine derives quarter, invokes D-01 with token, and then invokes D-02 without explicit arguments. D-01’s result is not visibly consumed. D-02 supplies a represented 64-bit result, divided into its low and high 32-bit words.

After D-02 returns, the following six stores occur in this order:

| Destination | Four-byte payload |
|---|---|
| auxiliary-base + quarter | Low word of D-02’s result |
| quantity + base-two | Low word of D-02’s result minus `0x16` |
| 2 × quarter | quarter |
| 2 × quantity | Original quantity |
| shared-pointer + quarter | High word of D-02’s result |
| 2 × base-two | base-two |

Together with the initial store, this phase contains **seven stores**.

The subtraction affects only the extracted low word. It is not a subtraction from the full 64-bit value, and there is no borrow adjustment to the high word. Behavior outside the representable range of the shown signed subtraction is not resolved by the supplied evidence.

The destinations 2 × quarter and 2 × quantity are numerical addresses, not offsets from an established allocation.

### Result-driven store sequence

D-03 is invoked with two arguments, both equal to base-two. Its result is not visibly consumed. D-04 is then invoked without explicit arguments and supplies result.

The next **ten stores** occur in the following order:

| Order within phase | Destination | Four-byte payload |
|---|---|---|
| 1 | 2 × base-two | Original quantity |
| 2 | quarter + base-two | result |
| 3 | base-two + `0x40` | quarter |
| 4 | 2 × base-two | Original quantity |
| 5 | quarter + `0x4c` | quarter |
| 6 | base-two + `0x4c` | quarter |
| 7 | 2 × base-two | base-two |
| 8 | 4 × result + `0x14` | result-quarter |
| 9 | result + base-two | result |
| 10 | 4 × result + `0x14` | result-quarter |

The original quantity is used for the first and fourth stores in this phase. Later reduction of a working value to quarter does not retroactively change those payloads.

Repeated stores are separate observable operations. In particular, the two stores at 4 × result + `0x14` surround a store at result + base-two. Aliasing can make the intermediate value significant even when the first and last payloads match.

No dependency invocation appears between these ten stores.

### Direct dependency and remaining indirect calls

The routine next reads a four-byte word from **base-two + `0x3c`** and supplies it as the first argument to F-c512cbdbd236. The second argument is stack-word. The dependency’s return value is not visibly used.

This memory read occurs after all preceding calls and stores. Its value is not an entry-time snapshot: earlier stores can alias the location, and earlier dependencies can change it.

After F-c512cbdbd236 returns, the remaining calls and **six stores** are interleaved as follows:

1. Store shared-pointer at **result-sixteenth + base-two**.
2. Invoke D-05 with **result** and **result-quarter**. Retain its opaque 32-bit result.
3. Store shared-pointer at **result-quarter + base-two**.
4. Invoke D-06 with the D-05 result and **result-quarter**. Its return value is not visibly consumed.
5. Invoke D-07 with **result-quarter** as both arguments, obtaining handle.
6. Store base-two at **shared-pointer + base-two**.
7. Store handle at **handle + `0x18`**.
8. Store shared-pointer at **result-sixteenth + base-two** again.
9. Store result-sixteenth at **result-sixteenth + `0x18`**.
10. Invoke D-08, obtaining final-base.

D-08 receives four arguments in this exact order:

- result-sixteenth;
- handle shifted left by two;
- result-quarter;
- result-sixteenth.

The repeated write at result-sixteenth + base-two occurs before and after intervening dependencies. It does not imply that those dependencies preserve the destination.

The later reduction of result-quarter by another two bits yields result-sixteenth. No rounding other than discarding the low bits is visible.

### Final stores and terminal loop

After D-08 returns, three stores remain:

| Order within phase | Destination | Four-byte payload |
|---|---|---|
| 1 | shared-pointer, offset zero | result-quarter |
| 2 | final-base + `0x3c` | result-sixteenth |
| 3 | Value of the global at `0x0000003c`, plus `0x18` | Value of that same global |

The last assignment uses the global once as an address-forming value and once as a payload. The supplied representation does not establish whether those uses share one load, use separate loads, or have a particular load order.

If the global remains stable during that assignment, its value is stored at its value plus `0x18`. If it can change between accesses, the address-forming value and payload need not match.

The global is used after D-08 and the two preceding final stores. Neither entry-time stability nor preservation by callees is assumed. Earlier stores may themselves alias the global.

After the global-dependent store, execution enters an unconditional infinite loop. No further memory operation, dependency invocation, return, or exit condition is visible in the loop.

## Side effects

### Memory mutation and aliasing

The body contains exactly **26 four-byte stores**:

| Phase | Store count |
|---|---:|
| Initial store and split dependency result | 7 |
| Result-driven store sequence | 10 |
| Direct dependency and remaining indirect calls | 6 |
| Final stores | 3 |
| **Total** | **26** |

No explicit byte store, halfword store, bulk copy, atomic operation, or memory barrier is shown.

No separation between destinations is established. Addresses can coincide or overlap partially. Stores can affect later reads, dependency-visible memory, or the low-address global. Pointer-shaped payloads can overwrite locations otherwise treated as integers.

The ordered stores cannot be replaced by a final-memory snapshot. Repeated values, partial overlap, memory-mapped devices, or external observers can make intermediate writes observable. Whether such observers or devices actually exist is unknown.

### Ownership and lifetime

The body contains no explicit allocation, deallocation, retain, release, or ownership-transfer operation.

This does not establish an absence of such effects inside dependencies. Passing a value to a dependency does not establish whether it is borrowed, retained, consumed, or released. Writing shared-pointer into memory does not establish ownership or extend the lifetime of its referent.

### Failure and partial effects

There is no visible bounds check, null check, alignment check, status inspection, rollback, cleanup, or alternative error path.

A fault or nonreturning dependency can leave a prefix of the stores observable. The body supplies no transactional or all-or-nothing guarantee. What exception, trap, or process-level result follows an invalid access is **documented_only**.

## Dependencies

D-01 through D-08 are opaque, document-local dependency IDs for invocation sites whose implementations are unavailable. F-c512cbdbd236 is the supplied opaque ID for the identified direct dependency.

| ID | Target evidence | Explicit arguments | Result use |
|---|---|---|---|
| D-01 | incoming-link + `0xff8` | token | Not visibly consumed |
| D-02 | Computed address `0x089121c9` | None | Represented 64-bit result split into two words |
| D-03 | Computed address `0x089121cb` | base-two, base-two | Not visibly consumed |
| D-04 | Embedded code anchor + `0xffe` | None | Unsigned 32-bit result |
| F-c512cbdbd236 | Direct identified dependency | Word read at base-two + `0x3c`, stack-word | Not visibly consumed |
| D-05 | Computed address `0x089118b3` | result, result-quarter | Opaque 32-bit input to D-06 |
| D-06 | Computed address `0x08911647` | D-05 result, result-quarter | Not visibly consumed |
| D-07 | Separate embedded code anchor + `0x448` | result-quarter, result-quarter | Signed 32-bit handle |
| D-08 | Computed address `0x08911517` | result-sixteenth, shifted handle, result-quarter, result-sixteenth | Signed 32-bit final-base |

The fixed addresses are arithmetic results of the visible target expressions, not independently verified executable entry points. The invocation notation does not establish an additional function-pointer-table read.

No dependency is assumed to return, preserve globals, preserve pointed-to storage, avoid callbacks, or follow any particular ownership convention.

## Acceptance examples

These examples are conditional static consequences, not executed tests. Each assumes execution reaches the described operations and that the relevant arithmetic and accesses are valid.

### Split 64-bit result

If D-02 supplies `0x1122334455667788`, its associated stores carry:

- `0x55667788` at auxiliary-base + quarter;
- `0x55667772` at quantity + base-two;
- `0x11223344` at shared-pointer + quarter.

These are individual write payloads. Aliasing or later writes can change the eventual contents.

### Logical truncation and numerical addresses

For quantity equal to `11`, quarter is `2`, not `3`.

The body consequently writes:

- `2` at numerical address `4`;
- `11` at numerical address `22`;
- `2` at base-two + `0x40`;
- `2` at numerical address `78`, from quarter + `0x4c`.

This does not imply those low or unaligned addresses are accessible.

### Unsigned multiplication width

For quantity equal to `0x80000000`, the unsigned 32-bit destination calculation 2 × quantity has low word zero. The corresponding store therefore targets numerical address zero under the indicated 32-bit address model and carries `0x80000000`.

This is a width consequence, not a guarantee that execution can reach or complete the store.

### Repeated destination

With no additional aliasing or intervening dependency writes to the location, the explicit writes at 2 × base-two carry, in order:

1. base-two;
2. original quantity;
3. original quantity;
4. base-two.

Reporting only the final value would omit three visible stores.

### Later arguments and stores

If D-04 returns `0x40`, result-quarter is `0x10` and result-sixteenth is `0x04`.

D-07 receives `(0x10, 0x10)`. If it returns handle equal to `3`, D-08 receives `(0x04, 0x0c, 0x10, 0x04)`.

After D-08 returns, the store through shared-pointer carries `0x10`, and the store at final-base + `0x3c` carries `0x04`.

### Final global and nontermination

If the global at `0x0000003c` is stable at `0x1000` during the final assignment, that assignment writes `0x1000` at address `0x1018`.

If every dependency returns and every access succeeds, the routine then remains in the unconditional loop. A normal return under those conditions contradicts the visible control flow.

## Uncertainties

The following remain **documented_only**:

- Complete ABI, incoming machine-state requirements, and stack-location interpretation.
- Byte order, actual address-space mapping, alignment restrictions, and memory extents.
- Exact treatment of signed overflow, signed left shift, integer-to-pointer conversion, and invalid pointer arithmetic.
- Implementation and memory effects of every dependency.
- Whether ignored return values have meaning outside this body.
- Exact access multiplicity and operand-evaluation order for the final global-dependent assignment.
- Effects of concurrency, asynchronous handlers, or device memory.
- Whether computed targets and low-address accesses represent executable behavior faithfully in the supplied reconstruction.
- Application-level intent and ownership relationships.

These unknowns do not justify adding validation, suppressing accesses, assuming callee preservation, or replacing nontermination with a return.

## Evidence

The evidence is the supplied static body and opaque dependency mapping. It establishes four explicit arguments, additional incoming state, nine invocation sites, a represented 64-bit dependency result, unsigned shifts, **26 four-byte stores**, a final global-dependent assignment, and an unconditional terminal loop.

The store total follows from seven stores in the initial phase, ten in the result-driven phase, six around the later dependency calls, and three after D-08.

Ordering claims concern visible top-level operations. They do not establish dependency internals or instruction-level evaluation order within expressions.

No runtime tests have been performed. No caller implementation, dependency implementation, memory map, independent instruction listing, ABI documentation, or execution trace was available.
