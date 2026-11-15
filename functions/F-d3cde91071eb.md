<!-- Trusted provenance: F-d3cde91071eb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d3cde91071eb

## Purpose

F-d3cde91071eb describes a single represented execution path consisting of dependency calls, 32-bit word transformations, propagation of 64-bit results, **21 explicit coprocessor transfers**, and a final 16-bit memory write. The path ends at an invalid-instruction termination marker rather than an ordinary return.

No application-level purpose is established. The evidence does not justify describing this function as initialization, decoding, resource management, or a successful transaction.

This contract distinguishes:

- **Established static evidence:** operations, operand relationships, widths, and ordering visible in the supplied representation.
- **Inferred interpretation:** consequences of interpreting represented arithmetic as fixed-width machine-word operations.
- **documented_only:** unresolved execution validity, calling conventions, dependency behavior, and platform effects.

All descriptions of later operations are conditional on preceding calls and architectural operations completing and allowing execution to continue.

## Interface

### Explicit inputs

The represented interface has four positional inputs and no ordinary return value.

| Position | Represented type | Explicit contribution |
|---|---|---|
| First | Opaque 32-bit quantity | No explicit use |
| Second | Opaque 32-bit quantity | No explicit use |
| Third | Unsigned 32-bit word | Left shift by 30 for the first call; logical right shift by one for a later call |
| Fourth | Signed 32-bit quantity | Left shift by 20 for F-a574095572d7 |

Under a 32-bit bit-vector interpretation, the third input’s left shift retains only its lowest two bits, placing them in bits 30 and 31. Its logical right shift produces an unsigned value between zero and 0x7FFFFFFF.

The fourth input’s left shift retains its lowest 12 bits under that interpretation. Because the input is represented as signed, this contract does not treat portable signed-language left-shift rules as an independently established execution model.

The first two inputs are not explicitly consumed. That observation does not establish their irrelevance to an unresolved machine-level calling convention.

### Implicit inputs and preconditions

The sequence also depends on:

- An incoming link-related value used to derive the first indirect call target.
- An incoming general-purpose register value, shifted left by nine for F-3fefa19f994b.
- Incoming coprocessor-related state referenced by the transfer operations.
- Six secondary-register values appearing in call arguments without ordinary return-value assignments establishing their contents.
- Indirect targets derived from represented address calculations.

A complete public ABI is **documented_only**.

To reach the final store, every preceding target must be callable in the actual execution environment, every called dependency must return as required by the represented path, and every architectural transfer must permit continuation. These are conditions for reaching that point, not validations performed by the function.

### Outputs

The visible outputs are calls, coprocessor effects, and one ordinary data-memory store of a 16-bit value.

There is no represented success result, error result, output structure, length field, or ownership-transfer result. The represented void interface does not imply normal completion: no ordinary return is shown.

## Behavior

### Width and ordering rules

The body contains no explicit conditional branch or loop. Dependencies may themselves branch, fault, terminate, or fail to return.

Working words are represented as 32 bits. Several results are represented as 64 bits and split into low and high 32-bit words. “Low” and “high” describe numeric significance, not byte order in memory.

For the arithmetic deductions below, a **32-bit bit-vector interpretation** means that a left shift discards bits above bit 31. Successive shifts operate on the already-truncated intermediate word. These deductions are conditional where the representation uses signed operands.

Unknown dependency effects must not be erased merely because the arguments simplify to constants. Conversely, an unresolved secondary-register argument must not be assumed zero because nearby arguments are zero.

### Entry and direct-dependency sequence

The initial sequence is:

1. An indirect call receives the third input shifted left by 30. Its result supplies coprocessor transfer 1.
2. A no-explicit-argument call occurs, followed by another call whose result supplies transfer 2.
3. Another call produces a word. A subsequent call receives that word, zero, and the third input logically shifted right by one.
4. A further no-explicit-argument call occurs, followed by F-aceee75de56b and two more indirect calls.
5. F-a574095572d7 receives the fourth input shifted left by 20.
6. An indirect call produces a word used for transfer 3 and then passed to F-3fefa19f994b, together with the incoming general-purpose register value shifted left by nine.
7. F-3c068908b58f produces a word used for transfer 4 and then passed unchanged to F-0a099a08c5e0.

No explicit input validation precedes these interactions.

### Early propagation and zero-derived arguments

The result of F-0a099a08c5e0 enters an indirect forwarding chain. A four-argument call receives the propagated result followed by three zero-valued arguments. Subsequent calls consume the returned word, forward it, or receive zero-valued transformed words.

After additional calls, F-97401b4d3e4e is invoked. A later four-argument call receives zero, the first unresolved secondary-register value, zero, and zero.

Another four-zero-argument call feeds a dependency returning a 64-bit quantity. Its low word, high word, and zero are passed to the next dependency. Further interactions include a returned word with zero, two zero-valued transformed arguments, and another zero-valued argument.

Several no-explicit-argument calls follow. A four-zero-argument call then feeds a forwarding call whose result supplies coprocessor transfer 5.

The constants in this phase arise from explicitly zero-valued working words and their transformations. They do not establish constant dependency results.

### Middle transformations

After further indirect calls, F-ea490577ce35 is invoked. A later returned word is supplied to a four-argument call as:

- The word itself.
- The word shifted left by 17.
- Zero.
- The word shifted left by 27.

Further calls produce the word used for transfer 6.

A subsequent 64-bit result supplies its low word to a three-call forwarding chain; the final word supplies transfer 7. Another 64-bit result supplies its low word and its high word shifted left by 17 to a separate call.

The high word of the earlier 64-bit result is shifted left by 19, then by nine, before entering another forwarding chain. Under the bit-vector interpretation, this is equivalent to a left shift by 28, retaining only the original high word’s lowest four bits.

A later returned word is passed with two zeros. Transfer 8 then receives zero. The next four-argument call receives zero, the second unresolved secondary-register value shifted left by seven, zero, and zero.

Following further calls, a returned word is passed alongside a copy shifted left by 20.

### Additional 64-bit dataflow

A later 64-bit result supplies several related arguments:

- Its low word is passed to one call.
- A following three-argument call receives the low word shifted left by 14, the third unresolved secondary-register value, and the high word shifted left first by nine and then by 14.
- The same transformed high word is passed to another call.
- A later four-argument call receives zero, zero, the low word masked to its lowest 18 bits and shifted left by four, and zero.

The composed high-word shift is 23 bits under the bit-vector interpretation and retains the high word’s lowest nine bits.

After an intervening call, another returned word supplies transfer 9. Further calls produce a 64-bit result whose low word supplies transfer 10. That result is also passed onward as its low word and its high word shifted left by 20.

The resulting 64-bit quantity supplies the next call’s three arguments: low word, high word, and high word shifted left by 12.

After more calls, a returned 32-bit word is passed to F-0e02e4c13579. A copy shifted left by 11 is retained for later arguments.

The following phase includes:

- A call receiving zero, the fourth unresolved secondary-register value, and zero.
- Several no-explicit-argument calls.
- A 64-bit-producing call receiving a preceding low word shifted left by four, its high word, and the retained word shifted by another 17.
- A call receiving the resulting low word, high word, and zero.
- Forwarding calls and a two-zero-argument call.
- A four-argument call receiving the retained word shifted by another 31, the fifth unresolved secondary-register value, zero, and zero.

The retained word has already undergone a left shift by 11. Its additional shift by 17 is equivalent to an original shift by 28 under the bit-vector interpretation. Its additional shift by 31 is zero under that interpretation. The last call’s result supplies transfer 11.

### Late arithmetic and transfers

Further forwarding and no-explicit-argument calls include a call receiving zero, the sixth unresolved secondary-register value, and zero.

A later returned word is passed with zero and a copy shifted left by five. Following more calls, F-3de949054bdd is invoked.

Subsequent calls consume returned words. A later two-argument call receives zero and the arithmetic-right-shift-by-five result of the earlier left-shift-by-five word. This transformation is not generally the original word: the left shift discards high bits, and the represented arithmetic right shift can sign-extend the retained sign bit.

Another 64-bit result supplies its low word to transfer 12 and its high word shifted left by one to a call. A later returned word supplies two consecutive transfers, 13 and 14, with distinct represented coprocessor operands.

A new returned word is shifted left by 27 and passed as the third argument of a four-argument call whose other arguments are zero. After additional calls, another returned word supplies transfer 15.

The earlier left-shift-by-27 word is then shifted left by another 24. This transformed value supplies transfer 16 and a call argument; it is zero under the bit-vector interpretation.

After more calls, a returned word supplies transfer 17. A three-zero-argument call produces a new word:

- Its left shift by one supplies transfer 18.
- A four-argument call receives that shifted value, a masked-and-shifted form of the earlier left-shift-by-27 word, the new word shifted left by four, and zero.

The masked-and-shifted argument is zero: the mask retains bits below bit 23, whereas the earlier word can have bits set only in positions 27 through 31.

After two intervening calls, transfers 19 and 20 both receive explicit zero. A call receiving zero returns another word. Transfer 21 receives the earlier left-shift-by-four word shifted by another eight.

The next call receives that same value, the left-shift-by-four word shifted by 12 instead, and the separately returned word shifted left by nine.

A later 64-bit result is passed onward as low word, high word, and low word shifted left by one. Another forwarding chain starts from zero under the bit-vector interpretation: its argument is derived from the lowest two bits of a word already shifted left by four.

### Final address and write

Near the end, a call produces a 64-bit result. The next call receives its low word, high word, and high word shifted left by 11, and returns a second 64-bit result.

Let **R** denote this second result. Its low word is passed to another dependency. Independently, an unsigned right shift of R by 41 produces a 23-bit address-related intermediate.

Three calls then occur before the store:

1. A call with no explicit arguments.
2. A call receiving the preceding result and a transformed argument that is zero under the bit-vector interpretation.
3. Another call with no explicit arguments.

Define **Q** as R logically shifted right by 52. The final numeric address is:

**A = Q + floor(Q / 4).**

This follows because the 23-bit intermediate is nonnegative when represented as a signed 32-bit value; its subsequent right shift by 11 therefore yields Q without sign extension.

Q ranges from 0 through 4095. A ranges from 0 through 5118, with gaps; it is not a contiguous indexing transformation. The addition cannot overflow 32 bits.

The function writes the 16-bit value **0x00CD** at address A. No buffer base, bounds check, alignment adjustment, or address-validity check is shown. The next represented event is invalid-instruction termination.

## Side effects

### Ordinary memory and ownership

The final halfword store is the only explicit ordinary data-memory write. No explicit ordinary data-memory read, allocation, or release is shown outside dependency and architectural interactions.

The halfword’s byte order is unknown. Some derived addresses are odd; alignment is not guaranteed. Mapping, writability, unaligned-access behavior, and the consequences of writing low numeric addresses are **documented_only**.

No dependency contract establishes allocation ownership, release obligations, borrowing, or preservation of memory and globals. Calls may have effects not visible in this body.

### Architectural state

There are **21 explicit coprocessor-transfer operations**. Their chronological payload inventory is:

| Transfer | Payload |
|---|---|
| 1 | First indirect call’s result |
| 2 | Early indirect call’s result |
| 3 | Result also supplied to F-3fefa19f994b |
| 4 | F-3c068908b58f result |
| 5 | Forwarded result after a four-zero-argument call |
| 6 | Returned word after the middle four-argument transformation |
| 7 | Result of the three-call low-word forwarding chain |
| 8 | Zero |
| 9 | Returned word after the masked-low-18-bit argument phase |
| 10 | Low word of a 64-bit result |
| 11 | Result of the call containing the fifth unresolved secondary-register value |
| 12 | Low word of a later 64-bit result |
| 13 | A returned word |
| 14 | The same word as transfer 13, with different coprocessor operands |
| 15 | Returned word after the left-shift-by-27 argument phase |
| 16 | That earlier shifted word, shifted left by another 24 |
| 17 | A subsequent returned word |
| 18 | A new returned word shifted left by one |
| 19 | Explicit zero |
| 20 | Explicit zero |
| 21 | The new word shifted left by four and then by eight |

Transfer 16 is additionally deduced to carry zero under the bit-vector interpretation; that deduction does not introduce an additional transfer.

The architectural meaning, privilege requirements, visibility, and fault behavior of these operations are **documented_only**. They are not established as cache, translation, interrupt, or device operations.

### Failure and partial effects

No explicit error branch, rollback, cleanup, or retry is shown. Earlier calls and transfers may take effect before a later failure. A store fault would prevent reaching the represented terminal operation through ordinary continuation.

The invalid-instruction endpoint’s concrete manifestation—exception, trap, process termination, or another response—is unknown. Atomic completion and successful return are not established.

## Dependencies

The identified direct dependencies occur in this relative order, interleaved with indirect calls:

| Opaque ID | Represented interaction |
|---|---|
| F-aceee75de56b | No explicit arguments; no result explicitly consumed |
| F-a574095572d7 | Fourth input shifted left by 20 |
| F-3fefa19f994b | Preceding result and incoming register word shifted left by nine |
| F-3c068908b58f | Produces a word used in a transfer and the next direct call |
| F-0a099a08c5e0 | Consumes that word; result enters indirect propagation |
| F-97401b4d3e4e | No explicit arguments |
| F-ea490577ce35 | No explicit arguments |
| F-0e02e4c13579 | Receives a returned 32-bit word |
| F-3de949054bdd | No explicit arguments |

“No explicit arguments” is a property of the supplied representation, not proof of an argument-free ABI.

Opaque identities for the indirect dependencies are not supplied. Their identity relationships, signatures, secondary-register outputs, memory effects, ownership behavior, and return guarantees remain **documented_only**. No preservation guarantee is inferred for globals or architectural state.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Third-input selection:** If the third input is 3, the first call receives 0xC0000000 under the bit-vector interpretation. Its later logical-right-shift argument is 1.

2. **Independent transformations:** If the third input is 0x80000000, the first shifted argument is zero, but the later right-shifted argument is 0x40000000.

3. **Fourth-input truncation:** A fourth input of 1 produces 0x00100000 for F-a574095572d7. A fourth input of 4096 produces zero under the bit-vector interpretation.

4. **Unresolved arguments:** A nonzero secondary-register value remains part of its corresponding call argument, subject to any explicitly represented shift. Adjacent zeros do not justify replacing it with zero.

5. **Arithmetic shift distinction:** If the late word is 0x04000000, its 32-bit left shift by five is 0x80000000. Under arithmetic-right-shift semantics, shifting that result right by five yields 0xFC000000, not the original word.

6. **Transfer inventory:** If execution reaches every represented transfer, there are 21 explicit transfers. Transfers 13 and 14 are separate operations despite sharing a payload; transfers 19 and 20 are separate explicit-zero operations.

7. **Zero address:** If R’s bits 52 through 63 are clear, A is zero. This includes values with set bits below bit 52 and does not establish that the store is permitted.

8. **Address spacing and maximum:** Q equal to 3 produces address 3; Q equal to 4 produces address 5. Q equal to 4095 produces address 5118. The stored halfword is always 0x00CD if the store completes.

9. **Terminal behavior:** If all preceding operations and the store complete, the next represented event is invalid-instruction termination, not normal return.

## Uncertainties

- **documented_only — executable validity:** Invalid instruction data is reported. A valid executable path corresponding to the complete representation is not established.
- **documented_only — calling convention:** Hidden inputs and six unresolved secondary-register values prevent a complete ABI contract.
- **documented_only — signed arithmetic:** Machine-level confirmation is required for represented signed left shifts and the late arithmetic right shift.
- **documented_only — dependencies:** No supplied callee contract establishes preservation, ownership, failure behavior, or additional side effects.
- **documented_only — architectural transfers:** Operational meaning and fault behavior remain unresolved.
- **documented_only — final store:** Address mapping, alignment behavior, byte order, and fault handling are unknown.
- **Inferred intent:** The call-and-transfer sequence suggests architecture-sensitive orchestration, but no narrower purpose is supported.

## Evidence

The contract is based on the supplied static representation: explicit input uses, call ordering, result widths, argument transformations, architectural transfers, final halfword write, and invalid-instruction endpoint.

The architectural inventory identifies 21 explicit transfers in source order. Arithmetic simplifications are separately qualified by the fixed-width interpretation; they neither remove calls nor add transfer operations.

The final address reduction uses the unsigned 64-bit shift by 41, the resulting nonnegative 23-bit intermediate, its subsequent shift by 11, and addition of its unsigned quotient by four.

No runtime tests have been performed. No dependency implementation, validated instruction stream, platform memory map, or independent execution trace establishes the unresolved behavior.
