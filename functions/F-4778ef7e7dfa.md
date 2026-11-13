<!-- Trusted provenance: F-4778ef7e7dfa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4778ef7e7dfa

## Purpose

The supplied static representation describes a sequential routine that invokes numerous external targets, performs thirteen coprocessor-transfer operations, and finally attempts a 16-bit memory write before reaching an abnormal instruction boundary.

No application-level purpose is established. Initialization, device configuration, decoding, and memory management would all be speculative interpretations. The contract below therefore describes observable sequencing and recovered data dependencies without assigning those meanings.

The representation explicitly reports invalid instruction data and truncated control flow. Consequently, this is a conditional contract for the represented path, not proof that the entire path is executable or that its recovered calling conventions are correct.

## Interface

### Entry and completion

The recovered interface has no explicit parameters and no declared return value. This does **not** establish that the routine is independent of incoming machine state.

Its represented behavior depends on:

- An incoming link-related value used to derive the first call target.
- An incoming 32-bit register value used in an early three-argument call.
- Incoming coprocessor-related operands used by the transfer operations.
- Additional register values recovered as secondary outputs of preceding calls, without established return-value contracts.
- The implementations and execution environment of all invoked targets.

No normal return is shown. The final represented actions are a halfword store and an invalid-instruction termination marker. The marker does not establish a particular runtime exception, operating-system signal, or recovery behavior.

### Value widths and notation

The following notation describes data values, not implementation variables:

- A **word** is 32 bits.
- A **pair** is a recovered 64-bit result, divided numerically into a low word and a high word.
- `Lₙ(x)` means a left shift by \(n\), retaining only the low 32 bits.
- A logical right shift fills with zeroes.
- An arithmetic right shift propagates bit 31.

These shift rules express the recovered fixed-width dataflow. They are not claims about the validity of equivalent signed operations in a particular programming language.

Numerical pair splitting does not establish memory byte order or the actual machine ABI used to return the pair.

### Memory and ownership

The only explicit ordinary-memory access outside the unknown dependencies is the final 16-bit store. No input buffer, output structure, allocation, release, ownership transfer, or lifetime contract is established.

Several call arguments are word-sized values. Their interpretation as integers, addresses, handles, or encoded fields is unknown. A value passed to a dependency must not be described as borrowed, owned, consumed, or released without additional evidence.

## Behavior

### Execution model and ordering

The represented path contains no conditional branches or loops. Calls and coprocessor transfers occur in the displayed order, provided each preceding operation completes and execution remains on that path.

For concise ordering references, this document assigns opaque, document-local IDs **F-X01 through F-X95** to the unresolved call occurrences, in order. These IDs identify occurrences, not proven distinct implementations. Their target identities and signatures remain unresolved.

Two additional calls have supplied dependency identities:

- **F-0e02e4c13579**, between F-X21 and F-X22.
- **F-3de949054bdd**, between F-X53 and F-X54.

The descriptions below establish ordering among the recovered operations. They do not assert that dependencies preserve globals, memory, hardware state, or incoming registers. Calls whose results are not used remain part of the observable sequence.

### 1. Initial result propagation and incoming-state use

F-X01 is invoked through a target derived from the incoming link-related state. Its word result is passed to F-X02. F-X02’s word result is then supplied to F-X03 both unchanged and shifted left by 20 bits.

F-X04 produces the first recovered pair. Its low word is supplied unchanged to F-X05. F-X06 then receives:

- That low word shifted left by 14 bits.
- An unresolved secondary register output associated with the preceding call.
- The first pair’s high word shifted left by a total of 23 bits.

F-X07 receives the same shifted high-word value. F-X08 follows.

F-X09 receives three explicit values, in order:

1. The incoming unsigned register value shifted logically right by 3, then left by 16.
2. Zero.
3. The first pair’s low 18 bits, shifted left by 4.

The first argument retains incoming bits 3 through 18 in output bits 16 through 31. The third argument is a multiple of 16 and cannot exceed `0x003ffff0`.

F-X10 produces the word used by the first coprocessor transfer. F-X11 through F-X14 follow in order, with no explicit arguments recovered.

### 2. Pair transformations before the first identified dependency

F-X15 produces a pair. Its low word is transferred to the coprocessor before F-X16 is invoked.

F-X16 receives that low word and the corresponding high word shifted left by 20. It produces another pair. F-X17 receives the new low word, new high word, and new high word shifted left by 12.

F-X18 through F-X20 follow. F-X21 then supplies a word to F-0e02e4c13579.

The recovered dataflow also retains a version of F-X21’s result shifted left by 11 for later argument construction. The identified dependency’s result, if any, is not explicitly consumed.

### 3. Zero-valued arguments and intermediate result chains

F-X22 receives zero, an unresolved secondary register output from the preceding identified dependency, and zero. F-X23 through F-X26 follow.

F-X27 supplies a pair to F-X28, with these transformations:

- The low word is shifted left by 4.
- The high word is unchanged.
- The third argument is the earlier F-X21 result shifted left by a total of 28 bits.

F-X28’s pair result is passed to F-X29 together with an explicit zero third argument. F-X29’s word result is passed to F-X30. F-X31 follows.

F-X32 receives two zero arguments; its word result is passed to F-X33.

F-X34 receives four explicit arguments. The first, third, and fourth are zero under the represented 32-bit dataflow. The second is an unresolved secondary register output associated with F-X33. In particular, the first argument is zero because its accumulated left shift is 42 bits across two truncating 32-bit operations.

F-X34’s word result is transferred to the coprocessor and then passed to F-X35.

The remaining order in this phase is:

- F-X36 supplies a word to F-X37.
- F-X37 supplies its result to F-X38.
- F-X39, F-X40, and F-X41 execute.
- F-X42 supplies a word to F-X43.
- F-X44 receives zero.
- F-X45 and F-X46 execute.
- F-X47 receives zero, an unresolved secondary register output associated with F-X46, and zero.
- F-X48 and F-X49 execute.

No return-value predicate, retry, or alternate path is represented in this sequence.

### 4. Shifted signed value and the second identified dependency

F-X50 produces a word. F-X51 receives that word, zero, and that word shifted left by 5.

F-X52 produces a word passed to F-X53. F-3de949054bdd is then invoked with no explicit arguments recovered.

Two further result-consuming pairs follow: F-X54 supplies F-X55, and F-X56 supplies F-X57.

F-X58 receives:

- Zero.
- The arithmetic-right-shift-by-5 result of F-X50’s previously left-shifted word.

The second argument is **not necessarily the original F-X50 result**. The left shift discards its upper five bits, and the arithmetic right shift sign-extends the retained bit 26. Its signed range is therefore \(-2^{26}\) through \(2^{26}-1\).

F-X58’s result is passed to F-X59.

F-X60 produces a pair. Its low word is transferred to the coprocessor before F-X61 receives the pair’s high word shifted left by 1.

F-X62 then supplies a word used by two consecutive coprocessor transfers. No intervening function call is represented between those transfers.

### 5. Restricted-bit values and additional coprocessor operations

F-X63 produces a word whose low five bits are shifted into bits 27 through 31. Call this transformed value the **upper-five-bit value**.

F-X64 receives zero, zero, the upper-five-bit value, and zero. F-X65 through F-X69 follow. F-X70 supplies a word for a coprocessor transfer, followed by F-X71 through F-X73.

The next coprocessor transfer has a zero payload: the upper-five-bit value shifted left by another 24 bits has no surviving bits. F-X74 then receives zero and zero.

F-X75 executes. F-X76 supplies a word for a coprocessor transfer, and F-X77 follows.

F-X78 receives three zero arguments. Let its word result be the **late result**. The routine:

- Transfers the late result shifted left by 1 to the coprocessor.
- Invokes F-X79 with the late result shifted left by 1, zero, the late result shifted left by 4, and zero.

F-X79’s second argument is zero because the upper-five-bit value has no bits within a low-23-bit mask.

F-X80 and F-X81 follow. Two zero-payload coprocessor transfers then occur consecutively.

F-X82 receives zero. After it returns:

- A coprocessor transfer uses the late result shifted left by a total of 12 bits.
- F-X83 receives the late result shifted left by 12, the late result shifted left by 16, and F-X82’s result shifted left by 9.

A separate low-two-bit extraction from the late result shifted left by 4 is necessarily zero. All later arguments derived solely by left-shifting that extraction are also zero.

### 6. Final calls, address construction, and termination boundary

F-X84 produces a pair. F-X85 receives its low word, high word, and low word shifted left by 1.

F-X86 receives zero. Its result flows through F-X87 to F-X88. F-X89 follows.

F-X90 produces another pair. F-X91 receives its low word, high word, and high word shifted left by 11. F-X91 itself produces a pair that has two separate uses:

- Its low word is passed to F-X92.
- Bits 41 through 63 of the complete pair form an unsigned 23-bit value used for the final address.

F-X93 produces a word passed to F-X94 together with zero. F-X95 follows. The final store occurs only after these calls complete.

Let \(Q\) be the unsigned value comprising bits 41 through 63 of F-X91’s pair result. The final address is:

\[
B=\left\lfloor Q/2048\right\rfloor,\qquad
A=B+\left\lfloor B/4\right\rfloor.
\]

Although the first right shift is represented through a signed 32-bit interpretation, \(Q\) cannot have bit 31 set: its maximum is `0x007fffff`. Thus that shift produces the same numerical result as unsigned division by 2048.

The resulting bounds are:

- \(0 \leq B \leq 4095\).
- \(0 \leq A \leq 5118\), or `0x13fe`.

The routine attempts to write the 16-bit value `0x00cd` at address \(A\). No base pointer, alignment adjustment, bounds validation, or address-validity check is represented.

Execution then reaches the invalid-instruction termination marker. No subsequent instruction or normal return is established.

## Side effects

### Calls and ordinary memory

All 95 unresolved call occurrences and both identified dependencies may have externally visible effects. Their memory reads, writes, resource operations, failure behavior, and hardware interactions are unknown.

The final halfword store is the sole explicit ordinary-memory write in the supplied representation. Its byte layout depends on target endianness. A particular byte ordering must not be promised.

The calculated address can be zero or odd. The representation contains no protection against either case. Whether the access succeeds, faults, performs an unaligned transaction, or targets special memory is environment-dependent.

### Coprocessor-transfer order

The thirteen transfers have the following recovered ordering and payloads. The selector notation preserves the displayed fields without assigning architectural meanings. “Operand” refers to an incoming coprocessor-related operand, not an ordinary call argument.

| Order | Selector fields | Payload | Remaining operand selectors |
|---|---|---|---|
| 1 | `(0,5,7)` | F-X10 result | 15, 15 |
| 2 | `(0,4,7)` | F-X15 low word | 7, 15 |
| 3 | `(0,6,7)` | F-X34 result | 3, 15 |
| 4 | `(0,6,7)` | F-X60 low word | 3, 15 |
| 5 | `(0,3,7)` | F-X62 result | 6, 15 |
| 6 | `(0,3,7)` | Same F-X62 result | 15, 15 |
| 7 | `(0,1,7)` | F-X70 result | 1, 15 |
| 8 | `(0,5,7)` | Zero | 5, 15 |
| 9 | `(0,6,7)` | F-X76 result | 9, 15 |
| 10 | `(0,5,7)` | Late result shifted left by 1 | 10, 15 |
| 11 | `(0,7,7)` | Zero | 9, 15 |
| 12 | `(0,0,7)` | Zero | 15, 15 |
| 13 | `(0,2,7)` | Late result shifted left by 12 | 9, 15 |

These transfers are represented as side-effecting operations. Their privilege requirements, synchronization effects, device significance, and validity on the actual processor are unknown. They must not be renamed as cache, translation, interrupt, or barrier operations without further evidence.

## Dependencies

- **F-0e02e4c13579:** receives the word returned by F-X21. No result is explicitly used. Its secondary register output participates in F-X22’s recovered arguments. Memory ownership and state effects are unknown.
- **F-3de949054bdd:** occurs after F-X53 and before F-X54. No explicit arguments or consumed return value are recovered. State effects are unknown.
- **F-X01–F-X95:** document-local opaque occurrence IDs for unresolved external calls. Their ordering and visible argument relationships are described above. No equivalence or distinctness between their implementations is established.
- **Execution environment:** must provide meaningful target resolution, call conventions, coprocessor behavior, and memory semantics for the represented path to execute.

A call displayed without explicit arguments establishes only that no arguments were recovered there. It does not prove a zero-parameter interface or exclude consumption of implicit machine state.

There is no evidence that any dependency preserves globals, owns argument memory, releases resources, or returns normally.

## Acceptance examples

These are conditional acceptance criteria derived from static dataflow. **No runtime tests have been performed.**

1. **Initial low-word masking.**  
   If F-X04’s low word is `0xffffffff`, and intervening calls return normally, F-X09’s third explicit argument is `0x003ffff0`. Its second remains zero.

2. **Preservation of relevant incoming bits.**  
   If the incoming unsigned register value is 8, F-X09’s first argument is `0x00010000`. Incoming bits above bit 18 do not survive the represented shifts.

3. **Accumulated shifts yielding zero.**  
   Regardless of F-X21’s word result, F-X34’s first explicit argument is zero. This does not make F-X34 removable: its other inputs and side effects remain relevant.

4. **Signed reconstruction is not identity.**  
   If F-X50 returns `0x04000000`, F-X51’s third argument is `0x80000000`. F-X58’s second argument is consequently `0xfc000000`, interpreted as \(-67{,}108{,}864\), rather than the original positive value.

5. **Upper-five-bit propagation.**  
   If F-X63 returns `0xffffffff`, F-X64’s third argument is `0xf8000000`. The later payload formed by shifting that value left by 24 is nevertheless zero.

6. **Final address boundaries.**  
   Conditional on all intervening operations completing:

   | F-X91 result | \(Q\) | \(B\) | Attempted store address |
   |---|---:|---:|---:|
   | `0x0000000000000000` | 0 | 0 | `0x0000` |
   | `0x0010000000000000` | 2048 | 1 | `0x0001` |
   | `0xffffffffffffffff` | 8,388,607 | 4095 | `0x13fe` |

   Each case attempts a 16-bit `0x00cd` store; none guarantees that the address is writable.

7. **Failure before completion.**  
   If a dependency does not return, a coprocessor operation traps, or execution otherwise leaves the represented path, later operations—including the final store—are not established as occurring.

## Uncertainties

The following are **documented_only**:

- The routine’s application-level purpose.
- Actual instruction validity and complete reachable control flow.
- The true public calling convention and implicit entry requirements.
- The identity and signatures of unresolved call targets.
- The architectural meaning of recovered secondary outputs.
- Coprocessor selector interpretation and privilege requirements.
- Dependency effects on globals, memory, registers, and resources.
- Target endianness, alignment behavior, and address-space mapping.
- The concrete runtime outcome at the invalid-instruction boundary.

Possible hardware-configuration intent is merely an inference from the transfer operations. It is not part of the established contract.

## Evidence

**Established static evidence:** the supplied representation contains an ordered call sequence, two mapped dependencies, recovered 32-bit and 64-bit dataflow, thirteen coprocessor transfers, one final halfword store, and an explicitly truncated invalid-instruction boundary.

**Derived static consequences:** zero-valued arguments from truncating shifts, the signed reconstruction of F-X50’s shifted result, the 23-bit bound on \(Q\), and the final address range follow from the represented widths and operations.

**Not established:** successful execution, dependency semantics, architectural correctness, normal completion, or any runtime observation. No runtime tests have been performed.
