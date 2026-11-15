<!-- Trusted provenance: F-c1dd6fe5eb43; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c1dd6fe5eb43

## Purpose

F-c1dd6fe5eb43 is an apparently nonreturning, stateful sequence of indirect calls interleaved with byte and 32-bit memory accesses. Each completed traversal derives inputs and retained machine state for the next traversal.

The supplied representation establishes the sequence of operations, but not a reliable application-level purpose. It does not establish that the accessed addresses denote ordinary objects, hardware registers, allocated storage, or valid memory. Interpreting the routine as an allocator, serializer, device initializer, or other named subsystem would be speculative.

This contract describes the visible execution path conditionally: execution must reach each operation, indirect calls must return when continuation is required, and memory accesses must be supported by the execution environment.

## Interface

### Explicit inputs

The visible interface has four 32-bit arguments:

| Position | Represented type | Visible use |
|---|---|---|
| First | Uninterpreted 32-bit value | Forwarded with the second argument to the first indirect call of each traversal; replaced near the traversal end. |
| Second | Signed 32-bit integer | Forwarded to that first call; subsequently replaced with a shifted feedback value. |
| Third | Uninterpreted 32-bit value | No visible use. |
| Fourth | Signed 32-bit integer | Used in scaled addresses and call arguments; replaced near the traversal end. |

There is no visible ordinary return value or normal return path.

### Additional incoming state

The four arguments do not fully describe the first traversal. Its behavior also depends on incoming machine state:

- A call-target anchor used for the first indirect invocation.
- An 8-bit value used in the first byte store.
- An address-like 32-bit value used to locate the selector for that store.
- A 32-bit value shifted into an early four-argument invocation.
- A secondary result-register value consumed late in the traversal without an explicit assignment in the supplied representation.
- A global 32-bit word read in the middle of each traversal.

These are not established public arguments. Their initial values, validity requirements, and calling-convention guarantees are **documented_only**.

### Width and address conventions

For describing intended machine-level calculations:

- Bytes are 8 bits.
- Words, pointers represented by integers, and most arithmetic values are 32 bits.
- Certain indirect results are represented as 64 bits and separated into low and high 32-bit halves.
- Logical right shifts apply where the source value is explicitly unsigned.
- Byte stores retain only the low eight bits.
- Address expressions have no visible bounds checks or pointer validation.

Calculations below describe 32-bit bit-vector behavior where truncation matters. This is not a claim that signed overflow or left shifting negative signed values is portable source-language behavior. Exact instruction semantics remain an uncertainty.

No byte order is established for word accesses.

## Behavior

### Dependency notation and ordering

The indirect callees have no resolved identities in the supplied dependency map.

This document uses opaque dependency IDs F-e001 through F-e017 for successive call regions. A region can contain multiple unresolved call sites; it is not a claim that those sites share a target or implementation.

An “advance chain” below means three indirect invocations with no explicit arguments, followed by a fourth such invocation whose result is consumed. Every invocation remains observable: discarded results do not establish absence of side effects. No global or memory preservation is assumed.

Descriptions of saved values refer to the supplied representation’s retained scalar values, not to a promise that callees preserve any particular physical register.

### 1. Initial exchange — F-e001

The first indirect call receives the first and second explicit arguments. Its target depends on the incoming call-target anchor.

Next:

1. A no-explicit-argument call produces a represented 64-bit result.
2. Another call receives its low half first and high half second.
3. A no-explicit-argument call occurs.
4. A further call produces a 32-bit result.

Only after these calls, the routine reads one unsigned byte at the incoming selector-source address plus 1. It adds 17 to that byte value and stores the incoming 8-bit value at the resulting address.

Consequently, that destination lies between 17 and 272 inclusive; the selector is not multiplied by a record size.

The last call result, shifted left by 2, is passed to another indirect call.

### 2. Initial scaled word — F-e002

An advance chain produces a word. Its left shift by 2 becomes the **initial scaled word**.

The routine writes that word at:

- Fourth argument multiplied by 4096, plus 20.

It then invokes a dependency with four explicit arguments:

1. Fourth argument multiplied by 1024.
2. The same value.
3. Incoming retained 32-bit state shifted left by 12.
4. Initial scaled word.

An advance chain then supplies an index. The initial scaled word is written at four times that index plus 20, and four times the index is passed to the next dependency.

The second destination is independent of the first unless their calculated addresses coincide.

### 3. First repeated scaled pair — F-e003

An advance chain supplies another index.

- Four times the index becomes the **first repeated word**.
- That word is stored at sixteen times the index plus 20.
- A dependency receives the word three times.

A subsequent advance chain supplies a separate index. The same first repeated word is stored at four times this new index plus 20. Four times the new index is passed onward.

An internal byte value is set to zero before the three-argument invocation. Later byte clears use that zero value.

### 4. Second repeated scaled pair and retained base — F-e004

An advance chain supplies another index.

- Four times it becomes the **second repeated word**.
- That word is stored at sixteen times the index plus 20.
- The original index multiplied by \(2^{30}\) becomes the **retained byte base**.
- A dependency receives the second repeated word three times.

Another advance chain supplies a new index, called the **retained word index** here. The second repeated word is stored at four times that index plus 20, and four times that index is passed onward.

Under 32-bit truncating arithmetic, the retained byte base has only four possible values. This does not establish that any resulting address is valid.

### 5. First byte clears and exchanged word — F-e005

An advance chain produces the **masked-read source**.

The routine writes zero first at retained byte base plus 17, then at retained byte base plus 1.

It passes the masked-read source shifted left by 2 to a dependency. A subsequent no-explicit-argument call produces a word, which is passed unchanged to another dependency.

An advance chain then produces another word. The same two byte clears are repeated, in the same order, before that result shifted left by 2 is passed onward.

Calls between these clearing sequences can change memory. The repeated stores therefore cannot be treated as redundant.

### 6. Masked read and saved result — F-e006

An advance chain produces an index. Four times this index becomes the **saved scaled index**.

The routine then:

1. Stores the retained word index shifted left by 31 at sixteen times the new index plus 20.
2. Reads a 32-bit word at  
   \[
   20 + ((\text{masked-read source}\mathbin{\&}(2^{30}-1)) \gg 20).
   \]
3. Passes the saved scaled index to a dependency.
4. Saves the result of a subsequent no-explicit-argument call as the **saved late word**.
5. Passes the loaded word shifted right by 22, then left by 28, to another dependency.

The read address ranges from 20 through 1043 inclusive. It need not be word-aligned.

Under 32-bit truncation, the final shift expression retains only bits 22–25 of the loaded word, placed in bits 28–31.

### 7. Zero word and first selector update — F-e007

An advance chain supplies an index. The routine stores a zero word at four times that index plus 20, then passes four times the index onward.

Another advance chain produces the **selector-update word**.

The routine reads a byte at retained byte base plus 1, adds 17 to the zero-extended byte, and stores the low byte of the selector-update word shifted left by 2 there.

It passes the full shifted word to a dependency. A subsequent call produces a word. Another dependency receives that word followed by the selector-update word shifted left by 30.

The selector read is performed at this point; it is not replaced by an assumption that the earlier clear remains effective.

### 8. Saved selector and associated word store — F-e008

An advance chain supplies a word.

The routine reads and saves the byte at retained byte base plus 1. It writes the low eight bits of

\[
(\text{selector-update word}\mathbin{\&}(2^{30}-1)) \gg 20
\]

at the saved selector plus 17.

It passes the chain result shifted left by 2 onward. After four additional no-explicit-argument invocations, it stores the saved late word shifted left by 2 at the saved selector plus 20.

This word-store address uses the saved selector. It does not reread the selector after the intervening calls.

The routine then reads the global 32-bit word.

### 9. Global-derived selector update and split result — F-e009

The saved scaled index is passed to a dependency. An advance chain produces a represented 64-bit result.

Its low half shifted left by 2 becomes the **middle scaled word**.

The routine freshly reads the byte at retained byte base plus 1 and writes the low byte of the previously captured global word shifted right by 22 at the selected address plus 17.

A dependency receives the high half of the 64-bit result. A subsequent call produces a word. Another dependency receives that word followed by the middle scaled word shifted left by 28.

An advance chain then supplies a word. Before forwarding that word shifted left by 2, the routine freshly reads the selector at retained byte base plus 1 and clears the selected byte at selector plus 17.

### 10. Middle word replication — F-e010

An advance chain supplies a word, whose left shift by 2 is passed twice to one dependency.

Another advance chain supplies a word. Its left shift by 2 becomes the **middle replicated word**.

The routine:

- Stores it at middle scaled word multiplied by 4096, plus 20.
- Passes middle scaled word shifted left by 14 onward.
- Executes an advance chain to obtain a new index.
- Stores the middle replicated word at four times that index plus 20.
- Passes four times the index onward.

No equality or non-aliasing requirement between these destinations is established.

### 11. Alternate selector source and another split result — F-e011

An advance chain produces another represented 64-bit result.

Its low half shifted left by 2 becomes the **late scaled word**.

The routine reads a byte at middle scaled word multiplied by 16384, plus 1. At that selector plus 17, it writes the low byte of:

\[
(\text{middle replicated word}\mathbin{\&}(2^{30}-1)) \gg 20.
\]

It passes the high half of the result onward, obtains another word, and passes that word to another dependency.

An intermediate byte-source base is formed by multiplying the late scaled word by \(2^{28}\).

A further call returns a represented 64-bit value, whose low and high halves are passed, in that order, to another dependency.

### 12. Two selector clears and retained selector source — F-e012

Following two no-explicit-argument calls, the latter result is retained.

The routine reads a byte at the intermediate byte-source base plus 1 and clears the byte at that selector plus 17. It then forwards the retained result shifted left by 2.

An advance chain supplies the **late source word**. Four times that word becomes the **late source scale**.

The next traversal’s selector-source address is set to four times the late scaled word. The routine immediately reads a byte at that address plus 1 and clears the byte at selector plus 17.

It passes late scaled word shifted left by 10 onward.

### 13. Further clear and late block word — F-e013

A call produces a word. The next dependency receives it followed by the saved late word shifted left by 28.

An advance chain supplies another word.

The routine freshly reads the byte at the next traversal’s selector-source address plus 1, clears the selected byte at selector plus 17, and forwards the chain result shifted left by 2.

Four no-explicit-argument calls follow.

The **late block word**, equal to late source word shifted left by 14, is stored at late source scale multiplied by 4096, plus 20. Late source scale shifted left by 14 is passed onward.

### 14. Secondary-result invocation and final replication — F-e014

A no-explicit-argument call occurs.

The next dependency receives:

1. Late block word shifted left by 28.
2. A secondary result-register value whose provenance is not explicit in the representation.

Under 32-bit truncating arithmetic, the first argument is zero: the word was already formed by a left shift of 14, and the subsequent left shift of 28 discards all original bits.

An advance chain supplies an index. The late block word is stored at four times that index plus 20, and four times the index is passed onward.

### 15. Feedback and repetition — F-e015

An advance chain produces a word. Its left shift by 2 becomes the **feedback scale**.

The routine freshly reads a byte at the next traversal’s selector-source address plus 1. At selector plus 17 it writes the low byte of:

\[
(\text{late block word}\mathbin{\&}(2^{30}-1)) \gg 20.
\]

Before repeating:

- The fourth argument becomes the feedback scale.
- Retained state for the next early four-argument call becomes the feedback scale.
- A dependency receives late source word shifted left by 26.
- The next first-call target anchor is replaced with an internally selected anchor.
- A further call supplies the next first argument.
- The second argument becomes feedback scale shifted left by 28.
- The next traversal’s initial byte-store value becomes zero.

Execution then repeats without a visible exit condition.

## Side effects

The visible direct effects comprise:

- Single-byte reads used as unsigned address selectors.
- Single-byte writes at selector-derived and retained-base addresses.
- 32-bit writes at scaled addresses with displacement 20.
- A masked-index 32-bit read.
- A global 32-bit read.
- Indirect calls throughout the traversal.

Selector-derived byte destinations range from 17 through 272; selector-derived word destinations range from 20 through 275. These ranges do not imply mapped storage or alignment.

All addresses can alias unless external constraints prohibit it. Earlier writes, intervening calls, and overlapping byte/word accesses can therefore affect later reads.

No allocation, release, ownership transfer, synchronization, rollback, or explicit error reporting is directly established. Unknown dependencies may perform any of these actions.

If execution stops during a traversal, the representation provides no mechanism for undoing preceding writes.

## Dependencies

The opaque regions F-e001 through F-e015 denote unresolved indirect-call sequences described above. Their identities, signatures beyond the explicit operand presentations, and effects are **documented_only**.

Two additional external dependencies are distinguished:

- **F-e016:** the environment supplying initial machine state, address interpretation, access validity, and calling convention.
- **F-e017:** the storage source of the global word, including possible overlap with other storage.

The available map identifies only F-c1dd6fe5eb43 itself. None of these opaque dependency labels asserts a recovered function identity.

Calls shown without explicit arguments are not guaranteed to consume no incoming registers or shared state. Likewise, a represented 64-bit result is not proof of a conventional 64-bit return type.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **First selector-derived byte write.** If the first selector read yields 6 and the incoming byte value is 171, the corresponding byte store targets address 23 with value 171. Earlier dependency calls must occur before that read.

2. **Initial scaled word.** If the relevant dependency result is 7 and the fourth argument is 3, the initial scaled word is 28 and the first word destination is 12308. The following four-argument call receives 3072 twice and 28 fourth.

3. **Masked-read boundary.** A masked-read source with all bits set produces address 1043, not an address scaled by four. An implementation that silently aligns this read changes the visible behavior.

4. **Byte truncation.** If the selector-update word is 255, its shifted-by-2 byte store writes 252. The dependency receiving the full shifted word receives 1020.

5. **Saved versus reread selector.** If the selector saved in F-e008 is 9, its later word store targets address 29 even if an intervening dependency changes the selector memory to 12.

6. **Repeated clears.** If a dependency changes retained byte base plus 1 between the two clear pairs in F-e005, the second pair still writes zero there. Omitting it is not behaviorally equivalent.

7. **Feedback width.** A feedback-producing result of 1 gives feedback scale 4 and a next second argument of 1073741824 under the stated bit-vector model.

8. **Nonreturning dependency.** If any invoked dependency never returns, later operations and the next traversal do not occur. No completed-traversal guarantee is implied.

## Uncertainties

**Established static evidence:** the represented widths, explicit operands, direct access ordering, saved versus reread selectors, feedback assignments, and unconditional repetition.

**Inferred intent:** the repeated shifts may construct encoded fields or scaled indices. The evidence does not identify such a format or object model.

**Documented_only:**

- Actual indirect targets and whether every invocation is executable.
- Initial hidden-state values and their external contract.
- Native overflow, alignment, pointer, and fault behavior.
- Storage validity and whether accesses are ordinary memory or device effects.
- Ownership and lifetime of any referenced storage.
- Dependency effects on globals and memory.
- Secondary-result provenance.
- Interrupt, exception, concurrency, and reentrancy behavior.
- Application meaning of the global word and its overlapping storage.
- Whether omitted unreachable material affects interpretation of the recovered interface.

No precondition checks, recovery paths, or success/error status values are visible. Their absence here must not be replaced with invented validation or fallback behavior.

## Evidence

This contract is based solely on the supplied static representation of F-c1dd6fe5eb43 and the supplied self-only dependency mapping.

No implementation access, runtime execution, tests, call-target resolution, or instruction-level verification has been performed. Arithmetic examples are conditional deductions, not observed outputs. No snapshot hash or independently verified provenance is asserted.
