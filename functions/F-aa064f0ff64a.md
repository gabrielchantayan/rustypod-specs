<!-- Trusted provenance: F-aa064f0ff64a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aa064f0ff64a

## Purpose

F-aa064f0ff64a performs a long sequence of dependency invocations interleaved with byte, halfword, and word memory accesses. Its established behavior is primarily an ordered trace of calls, captured values, fresh reads, and writes to calculated addresses.

No application-level purpose is established. The supplied static representation does not support classifying the function as an initializer, serializer, allocator, renderer, or cleanup operation.

This contract distinguishes:

- **Established static evidence:** operations and data dependencies explicitly represented in the supplied material.
- **Inferred intent:** possible interpretations that impose no acceptance requirement.
- **documented_only:** behavior not resolved by the available evidence.

Control-flow recovery warnings, implicit entry-state inputs, incomplete dependency interfaces, overlapping global storage, and a terminal decoding failure limit the contract. The described obligations are conditional on execution following the represented sequence and reaching the relevant operation. No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface has four 32-bit input slots.

| Input | Represented interpretation | Established use |
|---|---|---|
| First | Opaque 32-bit value | No explicit use appears in the represented body. |
| Second | Opaque 32-bit value | Delivered twice to the first dependency invocation. |
| Third | Signed 32-bit integer | Delivered to the first invocation; later shifted left by 25 bits for another invocation. |
| Fourth | Signed 32-bit integer | Shifted left by three bits for the first invocation. |

The apparent absence of a use of the first input does not establish runtime independence from it. The complete entry convention and dependency interfaces are unknown.

### Implicit inputs

The represented body also consumes:

- An entry control-transfer value from which the initial indirect target is calculated using displacement 4,076.
- Two unsigned 32-bit entry values used for logical shifts and byte extraction.
- An entry address-like value used for a byte read at displacement 1.
- Secondary outputs associated with dependency invocations.
- Memory contents, including low-address storage and one shared word whose address is unavailable.

The four explicit inputs alone are insufficient to establish a safe invocation.

### Outputs and completion

The displayed interface has no declared return value. Dependency results include individual 32-bit values and paired results represented as 64 bits. For a paired result, “low component” means bits 0–31 and “high component” means bits 32–63. This numeric split does not establish memory byte order.

No normal return is visible. The representation ends after an indirect invocation at a decoding-failure boundary. Whether actual execution traps, continues through unrecovered instructions, or takes a different path is **documented_only**. No deliberate error-return API is established.

### Preconditions

For any described operation to complete:

- Its dependency target must be executable under the actual calling convention.
- Every memory address must permit the specified access width.
- Required alignment and address-arithmetic conditions must hold.
- Retained addresses must remain usable until their later accesses.
- The environment must permit reached low-address accesses, including writes at address zero.

No explicit null check, bounds check, target validation, rollback, or recovery branch is visible. These are conditions on successful traversal, not evidence that the function enforces safety.

## Behavior

### Widths, arithmetic, and observation

The represented accesses comprise unsigned 8-bit reads, 8-bit stores, 16-bit stores, and 32-bit reads and stores. Byte stores retain the low eight bits; the halfword store retains the low sixteen bits. Unsigned right shifts are logical.

Unsigned 32-bit arithmetic is evaluated in a 32-bit domain. Signed multiplications and signed left shifts do not establish portable wraparound behavior. In particular, a calculated address must not be simplified by assuming signed overflow wraps. Actual machine behavior for those cases is **documented_only**.

Masks precede shifts where represented. Retaining 30 low bits and shifting right by 20 differs from shifting the unmasked value right by 20.

Within the represented straight-line sequence, separate accesses and invocations remain ordered. An intervening dependency may change memory or other observable state. Consequently:

- A captured value remains the input to later operations explicitly using that capture.
- A later memory-read expression is a fresh read, not an implicit reuse of a capture.
- Two reads at the same address are not necessarily equal.
- Separate writes remain separate, including identical consecutive writes.

Relative evaluation order between argument expressions in a single invocation is not established. Where two arguments independently read the same address, neither a left-to-right order nor one shared read is required by this evidence.

### Entry and initial word propagation

The initial indirect invocation receives, in order, the second input twice, the third input, and the fourth input shifted left by three bits.

The opening sequence also passes an implicit unsigned entry value shifted right by ten bits to a dependency. It reads an unsigned byte at displacement 1 from the implicit entry address. That byte, plus 17, selects a byte-store destination; the stored value is the low byte of another implicit unsigned entry value shifted right by 22 bits.

Further invocations receive dependency results shifted left by two bits. One later invocation receives the third explicit input shifted left by 25 bits and a dependency result shifted left by two bits.

The initial word-propagation region has a retained secondary output with three distinct uses:

1. It selects the address of a 32-bit zero store: 20 plus four times that secondary output.
2. That same captured secondary output is passed to a subsequent dependency.
3. After intervening invocations produce a primary-result selector, the captured secondary output is stored at 20 plus four times that primary selector.

The later store does **not** obtain a replacement secondary output from the intervening calls.

A subsequent primary selector determines a different store: 20 plus sixteen times the selector receives four times the selector. A word is then read at 20 plus four times the selector. These addresses generally differ; the store does not generally initialize the read location.

After further calls, the captured word shifted right by 11 bits is stored through another four-times-selector address. Its value shifted right by 22 bits, plus 20, independently selects a word read.

### Recurring byte-selected writes

A recurring operation reads an unsigned byte and writes one byte at the address equal to that byte plus 17. Such destinations range from 17 through 272 inclusive.

Selector-byte addresses include:

- Address 1.
- A retained value plus 1.
- Scaled values plus 1, including factors 4, 16, 64, 1,024, and 4,096.
- Values shifted right by 9, 11, 12, or 21 bits, plus 1.
- Masked and shifted values, plus 1.

The selector is an unsigned byte, not a full-width pointer read.

Stored values include zero, low bytes of scaled dependency results, and extracted bit fields. Extraction precedes truncation: an 8-bit store of a value shifted right by 22 retains original bits 22–29.

Address 1 is read repeatedly, often across intervening dependency calls. Each represented read is fresh. A destination selected earlier does not determine a later destination.

### Shared-state captures and mapped calls

F-23325cd5e50c is the first mapped direct dependency. It receives one 32-bit value captured from an earlier indirect invocation. No allocation, retention, or release semantics follow from this call.

Later, a dependency-produced 32-bit value is stored at address 124 before an invocation receives that retained value twice.

Another region reads a word at displacement 20 from a secondary-output-derived address and retains it for subsequent shifts and stores.

The handling of the shared word at address 40 contains **three represented reads**, not one:

- A read captures a 32-bit value for a later store.
- The next relevant invocation has two argument expressions, each independently reading the word at address 40.
- After subsequent calls, the earlier captured value is stored at 20 plus four times a secondary-output selector.

The argument reads are not represented as uses of the first capture. Their relative evaluation order is unspecified, and equality among the three values is not guaranteed. The later store nevertheless uses the first capture.

Mapped direct dependencies occur in this order, with substantial intervening activity:

1. F-23325cd5e50c.
2. F-bbb54b5f036d.
3. F-f80b3ac065fc.
4. F-de15789688a8.
5. F-ea83f9a17404.
6. F-674e75a953e8.

Two additional direct invocations lack supplied opaque identities. One receives a paired result’s low component, high component, low component, and low component. The other receives the same retained 32-bit value twice.

### Structured reads and the two displacement-4 accesses

One paired dependency result supplies a low component and a high address-like component. The function first reads a word at displacement 8 from the high component, then reads a word at displacement 12 from it.

The first captured word supplies a later byte-store value. The second supplies the address base for a later unsigned byte read at displacement 1. Calls intervene before that byte read and store; the word values are retained rather than reread.

There are also two distinct displacement-4 word reads:

- **Earlier access:** after further calls, a word is read at displacement 4 from the retained low component of the earlier pair. It is captured before an invocation receives a newer primary result twice. Following another invocation, the captured word’s low byte is written through a selector byte read at an address derived from the earlier low component shifted right by nine bits, plus 1.
- **Later access:** a different paired result supplies a low component from which an address is formed by adding 12. An invocation receives that address, the pair’s high component, and the address twice more. The address is retained across a dependency sequence containing byte reads, byte writes, and further invocations. After a newer paired result arrives, the function reads a word at displacement 4 from the retained address, before invoking the next dependency with the newer pair’s components. After that invocation and another call, the captured word selects an unsigned byte read at displacement 1; the resulting selector determines a byte store of zero at selector plus 17.

The later word access therefore uses the earlier low component plus 12, then displacement 4. It is neither a reread of the earlier displacement-4 location nor an access based on the newer pair.

These accesses establish no field meanings, object type, ownership, or complete allocation layout.

### Low-address stores and repeated word writes

A retained unsigned intermediate explicitly set to zero selects a 32-bit store at address zero. The stored value comes from a dependency result.

After intervening invocations and byte accesses, the same zero-valued intermediate determines a 16-bit store at address zero. Its source is a different dependency result.

These writes overlap but are separate ordered effects. Byte order and intervening dependency effects prevent specifying a final four-byte pattern.

A subsequent byte-store address combines four times the zero-valued intermediate with a retained quantity obtained by masking an earlier paired result’s low component to 30 bits and shifting right by seven. The stored byte comes from a newer paired result.

Elsewhere, a paired result’s high component plus 28 selects a byte destination; its low component supplies the stored byte.

Two later repeated-store groups must remain distinct:

- A word at displacement 52 from a retained result shifted right by nine receives a zero-extended selector byte, then that byte shifted left by one.
- A word at displacement 52 from a secondary output receives a primary result shifted left by two, twice consecutively.

The second group does not reuse the first group’s shifted byte value.

### Unavailable-address shared word and late fresh argument reads

Late in the sequence, a dependency-produced value is stored through a secondary-output selector. Before the next invocation, the unavailable-address shared word is then read **twice in sequence**:

1. The first read is captured as an unsigned 32-bit value.
2. The second read independently supplies the value multiplied by four to form a retained scaled address.

These captures have separate downstream roles. The first later supplies an argument obtained by retaining its low 30 bits and shifting right by ten. The second determines byte-read addresses at scaled address plus 1, used on separate occasions across intervening calls.

No established memory model requires the two shared-word reads to agree.

The second capture also feeds a later shifted base: the retained four-times-scaled quantity is shifted left by 26 bits. After intervening calls, one invocation has **two independent 32-bit word-read argument expressions at that derived base plus 28**.

These reads occur at that late invocation. They do not reuse an earlier word capture. Their relative argument-evaluation order is unspecified, and the two argument values are not guaranteed identical. Signed scaling and shifting limitations apply to the derived address.

The shifted base is also multiplied by 1,024 for subsequent address calculations. This contract does not simplify those calculations to zero using an unestablished signed-wraparound model.

### Final propagation and termination

The remaining region repeatedly stores through four-times-selector and sixteen-times-selector address relationships, interleaved with dependency invocations. It includes a paired result whose high component selects a zero word store and is later stored through another primary-result selector.

Near the end, a word captured at 20 plus a scaled selector is shifted right by 12 bits and stored through another four-times-selector address. Later arguments include:

- The low six bits of a four-times-scaled selector shifted left by 14.
- A retained selector shifted left by 30.
- An unsigned result shifted left by four.

Further byte-selected writes use fresh selector reads across intervening invocations. The final displayed operation is an indirect invocation, followed by the decoding-failure boundary. No successful completion behavior is established.

## Side effects

Established caller-side effects include:

- 8-bit, 16-bit, and 32-bit writes.
- Repeated and sometimes overlapping memory accesses.
- Invocations of mapped, unidentified direct, and unresolved indirect dependencies.
- Use of retained values across dependency invocations.

Dependency effects are additional and unknown. No callee is assumed to preserve globals, keep an address valid, retain memory, release memory, or leave shared storage unchanged.

No caller-side allocation, deallocation, ownership transfer, locking, synchronization, or reference-count protocol is identifiable. No rollback is visible. Earlier effects are not shown being undone if a later access or invocation fails.

Reentrancy, thread safety, atomicity, idempotence, and compatibility with null-address protection are not established.

## Dependencies

| Opaque ID | Established displayed arguments |
|---|---|
| F-23325cd5e50c | One 32-bit value. |
| F-bbb54b5f036d | The same retained 32-bit value twice. |
| F-f80b3ac065fc | A paired result’s low component, high component, low component, and low component. |
| F-de15789688a8 | The same retained 32-bit value twice. |
| F-ea83f9a17404 | The same retained opaque 32-bit value twice. |
| F-674e75a953e8 | The same retained integer-like 32-bit value twice. |

Most invocations are indirect and lack supplied identities. Two direct targets also lack supplied identities. Their complete interfaces, secondary-output conventions, failure behavior, memory effects, and ownership contracts are **documented_only**.

No displayed arguments does not mean no inputs or side effects. An unused primary result does not make an invocation removable.

## Acceptance examples

These are conditional static obligations, not execution results.

1. **Opening arguments.** With second input 9, third input 3, and fourth input 5, the first invocation receives 9, 9, 3, and 40. No signed overflow is needed.

2. **Retained initial secondary output.** If the initial relevant secondary output is 6, the zero word store addresses 44. The subsequent relevant invocation receives 6. If the later primary selector is 9, the later word store addresses 56 and stores the same captured 6.

3. **Unsigned selector and truncation.** A selector byte of 255 selects destination 272. If the source is an unsigned 0xFFC00000 shifted right by 22, the byte written is 0xFF.

4. **Address-40 read distinction.** If the first read captures 0x12345678 and later argument reads observe other values, the eventual capture-based store still uses 0x12345678. The two call arguments need not equal that capture or each other.

5. **Independent unavailable-address reads.** If the first read yields 0x00400000 and the second yields 3, the retained scaled address is 12. Later applicable selector reads address 13, while the first capture supplies a masked-and-shifted argument of 0x1000.

6. **Late displacement-28 arguments.** If the two word reads at the derived base plus 28 produce different values under the applicable memory semantics, they remain separate argument observations. Their assignment to first versus second argument must follow actual evaluation behavior, which this evidence does not resolve.

7. **Later displacement-4 chain.** If the retained low component is 100, the retained address is 112 and its later displacement-4 read addresses 116. If that read captures 200, the subsequent selector-byte read addresses 201, even if intervening calls change the word at 116.

8. **Repeated writes.** A retained selector byte of 7 produces successive displacement-52 word values 7 and 14. The other duplicate-store group instead writes its own primary result shifted left by two, twice.

9. **Overlapping zero-address writes.** Successful traversal includes a word store at zero and later a halfword store there. No final combined byte pattern is required without byte-order and intervening-effect evidence.

10. **Failure.** An invalid reached target or inaccessible address has no established error code, recovery branch, or normal-return requirement. The failure mechanism remains **documented_only**.

## Uncertainties

- **documented_only — Executable validity:** recovery warnings and terminal decoding failure prevent treating the representation as a verified executable control-flow graph.
- **documented_only — Entry and dependency conventions:** implicit inputs, secondary outputs, and full call interfaces remain incomplete.
- **documented_only — Arithmetic:** signed overflow, signed left shifts, pointer interpretation, and address wrapping lack machine-level confirmation.
- **documented_only — Memory semantics:** byte order, alignment, mappings, concurrency, device behavior, and consistency between repeated reads are unknown.
- **documented_only — Lifetime and ownership:** no dependency-preservation or storage-lifetime guarantee is supplied.
- **documented_only — Completion and errors:** no normal return or intentional terminal failure protocol is established.
- **documented_only — Full trace enumeration:** this contract describes the identified access patterns and retained-value relationships but does not enumerate every unresolved invocation or every secondary output.
- **Inferred intent only:** the shifts, masks, and narrow writes may manipulate packed state; recovery artifacts are another possibility. Neither interpretation is an acceptance requirement.

## Evidence

The supplied static representation supports the stated widths, access relationships, mapped-call order, repeated writes, and distinctions between retained values and fresh reads. Dependency names are presented only through supplied opaque identities.

In particular, the dataflow supports retaining the initial secondary output across calls; two separate captures of the unavailable-address shared word; two fresh late word-read arguments at displacement 28; and both displacement-4 access chains. It also distinguishes the initial address-40 capture from the two subsequent argument reads.

Comments, prior prose, and feedback supply no independent behavioral authority. Recovery warnings limit confidence rather than define an application error protocol.

No execution, runtime tracing, or tests have been performed. The acceptance examples express conditional requirements derived from static evidence, not observed outcomes.
