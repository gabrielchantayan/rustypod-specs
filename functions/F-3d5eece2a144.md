<!-- Trusted provenance: F-3d5eece2a144; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3d5eece2a144

## Purpose

F-3d5eece2a144 performs an ordered sequence of indirect calls, byte updates, 32-bit stores, and scaled-address accesses. The supplied representation does not establish an application-level purpose, object format, allocation protocol, or serialization scheme.

This contract covers the **visible execution prefix**, conditional on each dependency returning and each memory access completing. No explicit conditional branch, validation path, ordinary return, or status result is visible. The representation ends at a truncated control-flow boundary associated with invalid instruction data; it does not establish the actual runtime behavior beyond that boundary.

## Interface

### Explicit inputs

| Argument position | Represented width | Established visible use |
|---|---:|---|
| First | 32 bits | No explicit use |
| Second | 32 bits | No explicit use |
| Third | 32 bits, represented as signed | Shifted left by three and supplied twice to the first call |
| Fourth | Unsigned 32 bits | Shifted logically right by three; the reduced value participates in three displayed argument positions |

Unused displayed parameters are not proven behaviorally irrelevant: the true calling convention and dependencies’ access to incoming machine state remain unknown.

### Implicit inputs and outputs

Execution also depends on an incoming link-derived value, an incoming register-derived address, auxiliary register results, memory contents, and unresolved dependency behavior.

No explicit return value is established. Observable outputs are direct memory writes, dependency interactions, and any subsequent behavior outside the recovered prefix. No allocation, release, ownership transfer, error-result convention, rollback, or exception policy is established.

### Width and arithmetic model

Bytes are unsigned 8-bit values. Words and ordinary captured results are represented as 32-bit values. Two results are represented as 64 bits and explicitly separated into low and high 32-bit halves.

Byte stores retain the low eight bits; word stores write 32 bits. Unsigned right shifts are logical. A captured byte used as an address component is widened unsigned.

Arithmetic deductions and examples below use **32-bit modular address and shift arithmetic**, with wraparound modulo \(2^{32}\). This is a conditional interpretation consistent with the displayed widths, not independently verified machine semantics. Signed-looking shifts and products do not establish a source-language overflow contract.

Successful execution requires every reached call target to be callable and every reached access to be permitted under the actual machine’s mapping, alignment, and access rules. No checks establish these conditions.

## Behavior

### Reading the occurrence ledger

All dependency IDs below are opaque, document-local **call-occurrence IDs**. Different IDs do not prove different callees; repeated call shapes do not prove a shared callee.

A range such as **F-0102–F-0105** means four separate calls, in ascending ID order, each with an empty displayed argument list. It never means an optional or collapsed call. When a result is assigned to a run, it is the last occurrence’s represented result. Empty displayed lists do not establish argument-free signatures.

Events within each subsection occur in the stated order, and subsections concatenate. There are no omitted intervening call occurrences in this ledger. Reads used as call arguments occur before that call. Captured values remain the represented operands of later expressions; memory reads are fresh whenever another read is specified. No dependency is assumed to preserve memory, globals, ownership, or external state.

For compactness:

- A **destination base** obtained from a call is four times its 32-bit result.
- A **high-byte extraction** of a payload means its value masked with `0x3fffffff`, then shifted logically right by 20. The result can require ten bits before byte-store truncation.
- A **byte-selected write** reads an unsigned byte at the specified source address, then writes at the absolute address equal to that byte plus `0x11`. It is not relative to the source address.

### 1. Initial reduction and first byte clear

The **reduced fourth argument** is the fourth argument shifted logically right by three, ranging from zero through `0x1fffffff`.

**F-0101** displays three arguments: the third argument shifted left by three, that same shifted value, and the reduced fourth argument. Its target is derived from the incoming link value plus `0xffe`.

**F-0102–F-0105** follow; the last supplies the **initial result**. The routine then reads a byte at the incoming register-derived address plus `0x01` and clears the byte at the resulting selector plus `0x11`.

**F-0106** receives the initial result shifted left by two. **F-0107–F-0110** follow; their last result determines the **primary base**, four times that result.

### 2. Two word-to-byte updates

The routine reads the word at primary base plus `0x74` and stores its low byte at primary base plus `0x01`. It reads the word at primary base plus `0x74` again. There is no intervening dependency call between these two reads.

**F-0201** receives the second word read, the reduced fourth argument, and the reduced fourth argument.

**F-0202** has an empty displayed argument list and supplies a result passed to **F-0203**. **F-0204–F-0207** follow; their last result determines the **secondary base**, four times that result.

The word at secondary base plus `0x74` is read, and its low byte is stored at secondary base plus `0x01`. **F-0208** receives the secondary base. **F-0209–F-0212** follow.

An auxiliary register-derived value available at this point is called the **early auxiliary value**. Its precise production convention is unresolved.

### 3. High-bit-derived stores

The **top-seven value** is the primary base shifted logically right by 25. The **high stride** is the top-seven value shifted left by 20. Their respective ranges are zero through 127 and zero through `0x07f00000`.

The following interactions occur in order:

1. The word at four times the early auxiliary value plus `0x14` receives the top-seven value shifted left by 19. **F-0301** receives the early auxiliary value.
2. **F-0302** supplies a represented 64-bit result. **F-0303** receives its low half, high half, and low half again.
3. **F-0304–F-0307** obtain a destination base. Its word at offset `0x14` receives the early auxiliary value shifted left by three. **F-0308** receives that destination base.
4. **F-0309–F-0312** occur. The word at high stride multiplied by `0x200000`, plus `0x14`, receives high stride multiplied by `0x80000`. **F-0313** receives the stored product in all three displayed positions.
5. **F-0314–F-0317** obtain another destination base. Its word at offset `0x14` receives the high stride. **F-0318** receives that destination base.
6. **F-0319–F-0322** supply the **first payload**.

Under the stated modular model, both large products in item 4 are zero, because high stride contains a factor of \(2^{20}\). That event therefore writes a zero word at absolute address `0x14`.

### 4. Byte selection and retention

The **first scaled payload** is the first payload shifted left by two.

A byte-selected write reads high stride plus `0x01` and stores the low byte of the first scaled payload. **F-0401** receives the captured byte shifted right by three, the first scaled payload, and high stride shifted left by two.

**F-0402–F-0405** supply a separate **first update result**. The byte at high stride plus `0x01` is read again and retained as the **selector**. A byte at selector plus `0x11` receives the high-byte extraction of the first payload.

**F-0406** receives the first update result shifted left by two. **F-0407–F-0410** follow.

The word at four times selector plus `0x14` receives the first scaled payload. **F-0411** receives selector three times, followed by selector shifted right by one.

**F-0412–F-0415** obtain the next destination base. Selector remains the earlier captured unsigned byte, even if its source memory subsequently changes.

### 5. Scaled links and destination-base interactions

The next ordered interactions are:

1. The destination obtained by F-0415 receives the first scaled payload at offset `0x14`. **F-0501** receives that destination base.
2. **F-0502–F-0505** supply a result whose four-times value is **link one**. Link one is stored at sixteen times that result plus `0x14`. **F-0506** receives link one.
3. **F-0507–F-0510** obtain a destination base. Its word at offset `0x14` receives link one. **F-0511** receives that destination base.
4. **F-0512–F-0515** supply a result whose four-times value is **link two**. Link two is stored at sixteen times that result plus `0x14`. **F-0516** receives link two.
5. **F-0517** supplies a result forwarded to **F-0518**. **F-0519–F-0522** obtain a destination base. Its word at offset `0x14` receives link two masked with `0x3fffffff` and shifted logically right by one. **F-0523** receives that destination base.
6. **F-0524–F-0527** supply a result whose four-times value is **link three**. Link three is stored at sixteen times that result plus `0x14`.
7. The word at link three plus `0x14` is freshly read and shifted logically right by three for **F-0528**. Its address is not generally the address of the preceding store.
8. **F-0529–F-0532** obtain a destination base. Its word at offset `0x14` receives link three. **F-0533** receives that destination base.
9. **F-0534–F-0537** supply the **second payload**.

Each destination-base call is a separate interaction after its corresponding store; it is not optional bookkeeping.

### 6. Selector-derived stores and fresh-byte clearing

The **selector group** is selector shifted right by three, ranging from zero through 31. The **record base** is sixteen times the second payload. The **half payload** is the second payload masked with `0x3fffffff`, then shifted logically right by one.

The word at selector group multiplied by `0x80000`, plus `0x24`, receives selector. **F-0601** receives record base and second payload shifted left by two.

**F-0602–F-0605** occur. The word at record base plus `0x24` receives selector. The word at record base plus `0x74` is then read for **F-0606**.

**F-0607–F-0610** supply the retained **shift source**. A byte-selected write reads record base plus `0x01` and stores the high-byte extraction of the second payload. **F-0611** receives shift source shifted left by two.

**F-0612** supplies a result forwarded to **F-0613**. **F-0614–F-0617** supply a distinct **post-clear argument result**.

A fresh byte-selected write reads record base plus `0x01` again and stores zero. **F-0618** then receives the post-clear argument result shifted left by two. **F-0619–F-0622** occur before the next direct clear.

The two record-base-selected byte destinations may differ. The F-0618 operand is not the second payload or the retained shift source.

### 7. Independently addressed clears

The byte at half payload plus `0x0f` is cleared. The **clear base** is half payload multiplied by `0x400`. A byte-selected write reads clear base plus `0x01` and stores zero.

**F-0701** receives shift source shifted left by 23 in its first three displayed positions, and shift source shifted left by 25 in its fourth.

**F-0702** supplies a result forwarded to **F-0703**. **F-0704–F-0707** supply a separate **clear-update result**.

A fresh read of clear base plus `0x01` captures the **clear selector**. The byte at clear selector plus `0x11` is cleared. **F-0708** receives the clear-update result shifted left by two. **F-0709–F-0712** follow.

The routine then clears, in order:

- The word at the retained clear selector plus `0x44`.
- The word at absolute address `0x24`.

The first word address ranges from `0x44` through `0x143`; byte selection does not guarantee word alignment.

### 8. Repeated word clears and auxiliary interactions

**F-0801** displays zero, `0x98`, clear base, and `0x98`.

**F-0802** supplies a result forwarded to **F-0803**. **F-0804** supplies another result forwarded to **F-0805**. **F-0806–F-0809** obtain a destination base.

The word at that base plus `0x44` is cleared, followed by absolute word address `0x24`. **F-0810** receives the destination base.

**F-0811–F-0814** supply a represented 64-bit result. **F-0815** receives its low half shifted left by two. The routine then clears the word at twice its high half plus `0x44`, followed by absolute word address `0x24`.

**F-0816** receives zero, a newly exposed **late auxiliary value**, and that auxiliary value shifted left by two. Its precise register-result convention remains unresolved.

**F-0817** occurs. Define the **masked half** as half payload masked with `0x3fffff`. **F-0818** receives masked half shifted left by seven.

**F-0819–F-0822** obtain a destination base. The word at that base plus `0x44` is cleared, followed by absolute word address `0x24`. **F-0823** receives that destination base.

**F-0824–F-0827** supply the **pre-zero result**.

Across stages 7 and 8, the four direct clears of absolute word address `0x24` are distinct ordered events.

### 9. Further clearing and links

**F-0901** receives pre-zero result shifted left by two and returns the **zero-link result**. The word at four times that result plus `0x14` is cleared.

**F-0902** receives zero-link result three times, followed by masked half shifted left by seven.

The remaining interactions are:

1. **F-0903–F-0906** obtain a destination base. Its word at offset `0x14` is cleared. **F-0907** receives that destination base.
2. **F-0908–F-0911** supply a result whose four-times value is **link four**. Link four is stored at sixteen times that result plus `0x14`. **F-0912** receives link four.
3. **F-0913–F-0916** obtain a destination base. Its word at offset `0x14` receives link four. **F-0917** receives that destination base.
4. **F-0918–F-0921** supply another result. Its four-times value is stored at sixteen times that result plus `0x14`. **F-0922** receives its four-times and sixteen-times values.
5. **F-0923** supplies a result forwarded to **F-0924**, whose second displayed argument is masked half shifted left by four.
6. **F-0925–F-0928** obtain a destination base. Its word at offset `0x14` receives masked half multiplied by `0x80`. **F-0929** receives that destination base.
7. **F-0930–F-0933** supply the **third payload**.

The masked multiplication does not overflow 32 bits: its maximum is `0x1fffff80`.

### 10. Final byte updates

The **final base** is half payload multiplied by `0x4000000`.

A byte-selected write reads final base plus `0x01` and stores the low byte of third payload shifted left by two. **F-1001** receives third payload shifted left by two.

**F-1002** supplies a result forwarded to **F-1003**. **F-1004–F-1007** supply a distinct **intermediate final-update result**.

A fresh byte-selected write reads final base plus `0x01` and stores the high-byte extraction of third payload. **F-1008** receives the intermediate final-update result shifted left by two.

Only then do **F-1009–F-1012** supply the **final payload**. Another fresh byte read at final base plus `0x01` selects a destination receiving the low byte of four times final payload. **F-1013** receives that captured selector byte widened unsigned.

**F-1014** supplies a result forwarded to **F-1015**. **F-1016–F-1019** supply the separate **last shifted-result operand**.

A byte-selected write now reads **eight times final payload plus `0x01`**, not final base plus `0x01`, and stores the high-byte extraction of final payload.

**F-1020** receives the last shifted-result operand shifted left by two. **F-1021–F-1024** occur before the unresolved endpoint. No ordinary return is established.

## Side effects

Direct effects comprise byte stores, word stores, and every listed indirect-call occurrence. No transaction boundary or restoration action is visible. If an access faults or a dependency does not return, earlier completed effects are not established to be undone.

Offsets `0x01`, `0x0f`, `0x11`, `0x14`, `0x24`, `0x44`, and `0x74` are access offsets, not proven structure fields.

Byte-selected destinations range from absolute address `0x11` through `0x110`. Other accesses explicitly involve low absolute addresses. Replacing them with object-relative destinations changes the specified behavior.

Dependencies may independently affect memory, globals, external state, registers, or ownership. This contract neither grants ownership nor assumes any dependency acquires, preserves, or releases it.

## Dependencies

The occurrence ledger identifies every visible indirect call without assigning application semantics or concrete implementation identities.

Established interactions include:

- Empty displayed argument lists, with their exact occurrence counts and placement.
- Forwarding of represented results.
- Explicit scalar, selector, scaled-base, and shifted-result arguments.
- Two represented 64-bit results split into 32-bit halves.
- Auxiliary register-derived operands outside the four displayed parameters.

Actual signatures, hidden arguments, calling convention, preservation guarantees, failure behavior, memory effects, and relationships between call targets are **documented_only**. An unused represented return does not make its call unobservable.

## Acceptance examples

These are conditional static examples, not executed tests.

- **Input reduction:** Fourth arguments 8 and 15 both yield a reduced fourth argument of 1. F-0101’s third displayed argument and F-0201’s second and third displayed arguments therefore match. Complete execution equivalence is not established.
- **Word-to-byte truncation:** A first primary-base word read of `0x123456ab` produces a byte store of `0xab`. F-0201 still receives the separately reread word, not necessarily that earlier captured value.
- **Fresh selector:** If the first high-stride selector is 2 and a dependency changes its source byte to 9 before the second read, the retained selector becomes 9 and the second byte destination is `0x1a`.
- **Retained selector:** If clear selector is 9, the later word clear targets `0x4d`, even if an intervening dependency changes its original source byte. Whether that unaligned access succeeds is unresolved.
- **Destination-base argument:** If F-0510 returns 3, the link-one word store targets `0x20`, and F-0511 receives `0x0c`, under the modular model.
- **Post-clear shifted result:** If F-0617 returns 5, F-0618 receives 20 after the fresh record-base-selected zero store and before the half-payload-plus-`0x0f` clear.
- **Final intermediate result:** If F-1007 returns 7, F-1008 receives 28 after the second final-base-selected update and before F-1009–F-1012 obtain final payload.
- **High-bit products:** Primary base `0x02000000` yields top-seven value 1 and high stride `0x00100000`. The large-product event writes zero at `0x14`.
- **Extraction truncation:** Payload `0x3ff00000` yields high-byte extraction `0x3ff`; the byte written is `0xff`.
- **Interrupted execution:** A nonreturning F-0929 prevents the later third-payload sequence from being required. No rollback is established.

## Uncertainties

The following remain **documented_only**:

- Machine ABI, instruction-set details, endianness, alignment rules, memory mapping, and fault behavior.
- The origin and calling-convention meaning of incoming register state and auxiliary values.
- Whether represented 64-bit results correspond to source-level returns or aggregated machine registers.
- Concrete dependency identities, preservation behavior, ownership effects, and failure modes.
- Aliasing or special memory behavior beyond the stated conditional address model.
- The real continuation, return, or termination behavior after the truncated boundary.

The representation includes warnings concerning invalid instruction data and overlapping global symbols. These limit confidence in recovery; they do not establish a particular runtime trap or justify inventing missing behavior.

## Evidence

**Established static evidence:** The supplied representation contains the displayed widths, call occurrences and argument expressions, ordered direct accesses, captured-value reuse, repeated absolute clears, and truncated endpoint described here.

**Conditional deductions:** Range bounds, byte truncation, masked-product bounds, and modular zero-product conclusions follow from the stated arithmetic model.

**Inferred intent:** Repeated scaled stores might maintain encoded relationships or state, but no application-level interpretation is required or established.

**Verification status:** Static examination of the supplied representation only. No runtime tests, dependency execution, independent binary validation, or machine-level confirmation have been performed.
