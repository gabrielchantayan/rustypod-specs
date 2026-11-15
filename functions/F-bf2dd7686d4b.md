<!-- Trusted provenance: F-bf2dd7686d4b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bf2dd7686d4b

## Purpose

F-bf2dd7686d4b represents an ordered sequence of dependency calls, coprocessor transfers, and fixed-width arithmetic, ending with an attempted 16-bit memory write and an invalid-instruction termination marker.

No application-level purpose is established. The available evidence does not identify initialization, decoding, resource management, or successful completion.

This contract separates:

- **Established static evidence:** represented operations, operands, dataflow, and ordering.
- **Conditional arithmetic deductions:** consequences of interpreting ordinary words as 32-bit bit vectors and wide results as 64-bit bit vectors.
- **documented_only:** unresolved execution, dependency, hardware, and environmental behavior.

The represented body has no explicit branch, loop, validation path, or normal return. A warning about invalid instruction data limits confidence that this representation corresponds to a traversable executable path.

## Interface

### Explicit inputs

The routine accepts three uninterpreted 32-bit inputs. Their original values are forwarded, in order, as the first three arguments to F-a574095572d7.

No caller-side validation or normalization precedes that forwarding. The inputs are not established to be pointers, handles, lengths, signed quantities, or owned resources. They are not directly referenced again afterward.

### Implicit inputs

The representation also consumes:

- Three incoming register-derived words used in early left shifts.
- Six additional-register results associated with dependency calls.
- Eight distinct coprocessor-related implicit operands.

The calling convention supplying these inputs is **documented_only**. Providing the three explicit arguments alone does not account for all represented input dependencies.

For the transfer inventory below, the coprocessor-related operands are labeled C1, C3, C5, C6, C7, C9, C10, and C15. These labels preserve their represented distinctions; they do not establish hardware register semantics.

### Results and completion

No ordinary return value or normal return is represented.

If execution reaches the endpoint, it attempts to write the 16-bit numeric value 0x00CD to a computed byte address. The next represented operation is an invalid-instruction termination marker. Whether this produces a particular exception, signal, trap, or process outcome is **documented_only**.

Earlier nonreturning calls, faults, or failed accesses can prevent later operations.

### Widths and interpretation

The arithmetic deductions in this contract use:

- 32-bit ordinary words.
- 64-bit wide results.
- Low word: bits 0–31 of a wide result.
- High word: bits 32–63.
- Left shifts truncated to 32 bits where the represented operand is a word.
- Logical right shifts that insert zero bits.

Signed shifts are identified separately. These conventions do not assert that the represented signed operations are defined by a particular high-level language.

No structure layout, array format, string encoding, or input object representation is established. Splitting a wide result into words specifies numerical bit positions, not memory byte order.

## Behavior

### Ordering requirements

The body represents one ordered path. Calls with unused results remain observable dependency boundaries.

Retained values describe represented dataflow; they do not imply that an unknown dependency preserves physical registers, globals, memory, or device state. No call or transfer may be reordered on an unsupported preservation assumption.

Transfer labels T01–T19 refer to the complete operand inventory under **Side effects**.

### 1. Entry forwarding and early fields

Before the first call, one incoming word is shifted left by 5. F-a574095572d7 receives the three explicit inputs followed by that shifted word.

An unidentified no-explicit-argument call produces a word. Another incoming word is shifted left by 16, and T01 transfers the returned word. F-3fefa19f994b then receives:

1. That returned word.
2. The remaining incoming word shifted left by 9.
3. The incoming word shifted left by 16.

F-3c068908b58f is called without explicit arguments. T02 transfers its result, which is then passed to F-0a099a08c5e0 and through another dependency.

The next significant four-argument call receives a preceding result, zero, the retained left-shift-by-16 value shifted left by 11, and the retained left-shift-by-5 value shifted left by 30.

Under 32-bit truncation:

- The third argument retains only five original low bits, positioned at bits 27–31.
- The fourth argument is zero because its accumulated shift is 35 bits.

After result-forwarding calls, the bits-27–31 field is shifted left by 13 and passed to another dependency. That argument is zero.

Further no-explicit-argument calls surround F-97401b4d3e4e. A later four-argument call receives zero, an additional-register result, the retained bits-27–31 field logically shifted right by 10, and zero. Its second argument is unknown; its third can occupy bits 17–21.

### 2. First wide result and zero-derived arguments

After another call, a dependency receiving four zeros produces a word. A subsequent dependency consumes that word and produces a wide result.

Another dependency receives the wide result’s low word, high word, and zero, in that order. Its returned word is passed onward with a second argument of zero.

Subsequent calls receive:

- The least significant bit of the early bits-27–31 field, shifted left by 26.
- A signed right shift by 1 of an explicitly zero word.
- That zero word logically shifted right by 2 and then left by 3.

All three expressions are zero under the stated word interpretation.

A sequence of calls without explicit arguments follows. Another four-zero-argument call produces a word that passes through one dependency before T03 transfers the resulting word.

### 3. Middle wide results and retained high-word bits

Following intervening calls, F-ea490577ce35 is called without explicit arguments.

A newly returned word is supplied to a four-argument dependency as the word itself, its left shift by 17, zero, and its left shift by 27.

Later, another returned word is transferred by T04. Several calls precede a wide-result producer. That wide result’s low word passes through three dependencies before T05 transfers the resulting word. Its high word is retained separately.

A second wide result is obtained. A dependency receives its low word and its high word shifted left by 17. After intervening calls, the earlier retained high word is passed onward after successive left shifts by 19 and 9. With word truncation, this retains four original low bits in positions 28–31.

Further result-forwarding calls lead to a call receiving a returned word and two zeros. The caller then establishes a zero word, and T06 transfers zero.

The following four-argument call receives zero, an additional-register result shifted left by 7, zero, and zero. The additional-register-derived argument is not established to be zero.

### 4. Masked fields and further wide-result processing

After additional calls, a returned word is supplied with its left shift by 20.

A later wide result provides:

- Its low word to one dependency.
- Its low word shifted left by 14 to another.
- Its high word shifted left by 9 and then by 14 to two dependencies.

The latter high-word expression retains nine original low bits in positions 23–31. One consumer also receives an additional-register result.

A following four-argument call receives zero, zero, the low word masked to 18 bits and shifted left by 4, and zero. The nonconstant argument occupies only bits 4–21.

A newly returned word is transferred by T07. After intervening calls, another wide result is obtained. T08 transfers its low word. A dependency then receives that low word and the high word shifted left by 20, producing another wide result.

The next consumer receives this new result’s low word, high word, and high word shifted left by 12.

After further calls, a dependency returns a word whose left shift by 11 is retained. F-0e02e4c13579 receives the unshifted word. A subsequent call receives zero, an additional-register result, and zero.

### 5. Later zero-derived arguments and signed shifting

Following intervening calls, a wide-result consumer receives:

1. The low word shifted left by 4.
2. The high word.
3. The word retained before F-0e02e4c13579, shifted successively left by 11 and 17.

The third argument retains four original low bits in positions 28–31.

The resulting wide value is consumed as low word, high word, and a zero-derived third argument. Later calls include a two-zero-argument producer and a consumer of its result.

A subsequent four-argument call receives zero, an additional-register result, zero, and zero. T09 transfers its returned word, which is also passed onward.

After further calls, including explicit-zero argument calls and another call with an additional-register argument, a newly returned word is supplied as the word itself, zero, and its left shift by 5.

F-3de949054bdd is subsequently called without explicit arguments. Further result-forwarding calls precede a two-argument call receiving zero and a signed right shift by 5 of the retained left-shift-by-5 word.

That signed expression is not generally the original word: the left shift discarded five high bits. Its exact interpretation depends on signed-right-shift semantics.

A later wide result has its low word transferred by T10 and its high word shifted left by 1 and passed to a dependency. Another returned word is then transferred twice consecutively, by T11 and T12. These transfers have different coprocessor operand pairs.

### 6. Final fields and transfer ordering

A dependency produces a word that is shifted left by 27. A four-argument call receives zero, zero, this field, and zero.

Six no-explicit-argument calls follow. T13 transfers the word returned by the sixth. Three more no-explicit-argument calls precede T14, which transfers the retained left-shift-by-27 field shifted left by another 24. This payload is zero under 32-bit truncation.

A two-argument call then receives that zero-valued expression and zero. One no-explicit-argument call follows, then another produces a word transferred by T15.

After one further no-explicit-argument call, a three-zero-argument call produces a word. T16 transfers its left shift by 1.

A four-argument consumer receives:

1. That word shifted left by 1.
2. The earlier left-shift-by-27 field masked to 23 bits and shifted left by 4.
3. The newly returned word shifted left by 4.
4. Zero.

The second argument is zero because the earlier field has no set bits below bit 27. The third argument is retained.

After two further no-explicit-argument calls, T17 and T18 transfer explicit zeros, with a local zero-derived field calculation between them.

A single-zero-argument call produces another word. **Before the next dependency call, T19 transfers the retained left-shift-by-4 word shifted left by 8.**

The next three-argument dependency receives that same shifted-by-8 value, the retained word shifted left by 12, and the newly returned word shifted left by 9.

Another wide result is then supplied as low word, high word, and low word shifted left by 1. A later dependency receives a zero-derived expression: masking the retained left-shift-by-4 word with 3 gives zero, as do the subsequent shifts applied to that masked value.

### 7. Terminal address and write

Near the endpoint, a wide result is consumed as low word, high word, and high word shifted left by 11. This dependency produces the final address-producing wide result.

The caller logically shifts that result right by 41 and retains a 32-bit word. This field is at most 23 bits wide.

**Four calls occur after the address-producing result and before the store, in this order:**

1. A consumer of the result’s low word.
2. A no-explicit-argument word producer.
3. A consumer of that returned word and a zero-derived argument.
4. A final no-explicit-argument call.

Only afterward does the caller compute the terminal address from the retained field.

Let \(q\) be the unsigned quantity in bits 52–63 of the final address-producing wide result. The byte address is:

\[
q+\left\lfloor q/4\right\rfloor.
\]

Although an intermediate right shift is represented as signed, its input is a nonnegative, at-most-23-bit field. The address deduction therefore does not require negative signed-shift behavior.

Here \(0\leq q\leq4095\), giving addresses from 0 through 5118 inclusive. No alignment adjustment, validity check, or bounds check is represented.

The caller attempts a 16-bit write of numeric value 0x00CD, then reaches the invalid-instruction marker if the write completes.

## Side effects

### Coprocessor-transfer inventory

There are **19 represented transfers**, ordered below.

Each selector record lists the first three numeric operands followed by the final two coprocessor-related operands. The payload occupies the fourth position in the represented six-operand operation. Recording these operands does not establish their hardware meaning.

| Event | Selector record | Payload |
|---|---|---|
| T01 | 0, 7, 7; C9, C15 | First unidentified returned word |
| T02 | 0, 3, 7; C1, C15 | Result of F-3c068908b58f |
| T03 | 0, 0, 7; C1, C15 | Word obtained through the later four-zero-argument chain |
| T04 | 0, 4, 7; C1, C15 | Returned word after the F-ea490577ce35 region |
| T05 | 0, 1, 7; C10, C15 | Low word transformed through three dependencies |
| T06 | 0, 6, 7; C6, C15 | Zero |
| T07 | 0, 5, 7; C15, C15 | Returned word after the 18-bit-mask consumer |
| T08 | 0, 4, 7; C7, C15 | Low word of a subsequent wide result |
| T09 | 0, 6, 7; C3, C15 | Result of a four-argument call containing an additional-register argument |
| T10 | 0, 6, 7; C3, C15 | Low word of the later wide result |
| T11 | 0, 3, 7; C6, C15 | Newly returned word |
| T12 | 0, 3, 7; C15, C15 | Same returned word as T11 |
| T13 | 0, 1, 7; C1, C15 | Newly returned word after the left-shift-by-27 consumer |
| T14 | 0, 5, 7; C5, C15 | Retained field shifted successively by 27 and 24; zero |
| T15 | 0, 6, 7; C9, C15 | Newly returned word before the three-zero-argument producer |
| T16 | 0, 5, 7; C10, C15 | Three-zero-argument producer’s result shifted left by 1 |
| T17 | 0, 7, 7; C9, C15 | Explicit zero |
| T18 | 0, 0, 7; C15, C15 | Explicit zero |
| T19 | 0, 2, 7; C9, C15 | Retained left-shift-by-4 word shifted left by 8 |

No dependency call intervenes between T11 and T12. Equal payloads do not make those operations interchangeable or redundant.

Transfer privilege requirements, device effects, synchronization, and faults are **documented_only**.

### Memory, ownership, and interruption

The terminal store is the only explicit ordinary data-memory write represented in the caller. No ordinary caller-side data-memory read is explicitly shown. Indirect target calculations do not establish a function-pointer-table layout.

The store’s byte order, memory mapping, alignment requirements, and partial-fault effects are unknown. Address 5118 identifies the first byte of a two-byte access, not a one-byte operation.

No caller-side allocation, release, ownership transfer, cleanup, retry, rollback, or checked error result is represented. Dependencies may nevertheless allocate, release, retain, or modify resources. No borrowing or ownership contract can be inferred merely from argument forwarding.

Earlier effects are not established to be transactional or reversible.

## Dependencies

Named dependency occurrences appear in this order:

| Opaque function ID | Established interaction |
|---|---|
| F-a574095572d7 | Receives the three explicit inputs and an entry-derived fourth word |
| F-3fefa19f994b | Receives a returned word and two shifted incoming words |
| F-3c068908b58f | Produces the word used by T02 and passed onward |
| F-0a099a08c5e0 | Consumes that word |
| F-97401b4d3e4e | Called without explicit arguments |
| F-ea490577ce35 | Called without explicit arguments |
| F-0e02e4c13579 | Consumes a word whose left shift by 11 is retained |
| F-3de949054bdd | Called without explicit arguments later in the sequence |

Their semantic contracts are **documented_only**.

Other dependencies are unidentified indirect targets. Their identities, possible aliasing with each other or named dependencies, calling conventions, and effects remain unresolved. No independent function identities are assigned to them.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Input forwarding:** If the first call occurs, its first three arguments equal the three original 32-bit inputs in order.

2. **Truncating shifts:** A word shifted left by 5 and then by 30 contributes zero. Retaining a wider intermediate would change the represented argument.

3. **Unknown additional-register value:** A four-argument call containing zero, an additional-register result, zero, and zero cannot be replaced with an all-zero call without further evidence.

4. **Wide-word order:** A consumer specified to receive low word followed by high word must not receive the halves exchanged.

5. **Distinct consecutive transfers:** T11 and T12 use the same payload but respectively end in C6, C15 and C15, C15. Both operations remain represented.

6. **Late nonzero transfer:** If the three-zero-argument producer returns 1, T19 carries 4096 under the word interpretation. It is not one of the adjacent explicit-zero transfers.

7. **Terminal call boundary:** If the fourth terminal call does not return, the store is not required to occur, even if the address-producing wide result is already available.

8. **Address boundaries:** For \(q=0\), the attempted store starts at address 0. For \(q=4095\), it starts at 5118. For \(q=1\), it starts at the potentially unaligned address 1.

9. **Store interruption:** If the terminal store faults, the representation does not establish that the invalid-instruction marker executes.

## Uncertainties

The following remain **documented_only**:

- Fidelity and reachability of the represented control flow.
- Validity and identity of indirect call targets.
- Complete calling convention and additional-register-result semantics.
- Dependency effects on memory, globals, registers, devices, and ownership.
- Instruction-level interpretation of signed shifts.
- Hardware interpretation of the recorded transfer operands.
- Memory mapping, byte order, alignment, and fault behavior.
- Runtime consequences of the invalid-instruction endpoint.
- Application-level intent.

The visible selector operands and ordering are established despite unknown hardware semantics. Conversely, recording those operands does not resolve their meaning.

## Evidence

The evidence is the supplied static representation and opaque dependency mapping.

It supports the argument dataflow, 19 ordered coprocessor transfers with their represented operands, four terminal dependency calls, attempted 16-bit store, and absence of a visible normal return.

Zero-valued expressions, retained-bit ranges, and terminal-address bounds are arithmetic deductions conditional on the stated widths and truncation rules.

No runtime tests, execution traces, instruction-level validation, or dependency inspections have been performed.
