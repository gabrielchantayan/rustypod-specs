<!-- Trusted provenance: F-ff04ccfdd1aa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ff04ccfdd1aa

## Purpose

F-ff04ccfdd1aa repeatedly invokes unresolved dependencies and uses their results to perform ordered byte and 32-bit memory accesses. The visible body has an unconditional loop and no normal return.

The established behavior is an access-and-invocation sequence, not a terminating transformation. No application-level purpose, object model, allocation scheme, or device protocol is established. Scaled addresses and small offsets do not establish valid objects or mapped storage.

This contract separates visible behavior from unresolved dependency behavior. Calls may affect memory or machine state; they are not assumed to preserve globals, return normally, or acquire or release ownership.

## Interface

### Explicit inputs

The represented interface has four 32-bit argument positions:

| Position | Visible use |
|---|---|
| First | No direct read in the visible body. |
| Second | Supplied twice to the first invocation and subsequently stored as a word. Replaced before repetition. |
| Third | Supplied as the third argument to the first invocation. Replaced before repetition. |
| Fourth | Its low eight bits supply four direct byte stores. Set to zero late in the iteration. |

A lack of direct use does not establish that an argument is unobservable through the actual calling convention.

### Additional entry state

Two incoming machine-state values have no visible initialization:

- A **base seed**, used to derive the first byte-access base.
- A **call-target seed**, used to calculate the first invocation address.

Their origins and ABI roles are **documented_only** unknowns. They are not established as additional public parameters.

A later two-argument invocation consumes an unresolved secondary result. Its producer and ABI interpretation are also **documented_only**.

### Results, preconditions, and errors

There is no explicit returned result. If every invocation returns and every access succeeds, another iteration begins.

Execution requires the encountered invocation targets and memory accesses to be valid under the actual execution environment. The body performs no visible validation of targets, mappings, alignment, or permissions.

No local error result, exception handler, rollback, or recovery path is visible. A fault or non-returning dependency can prevent all subsequent behavior.

### Width and address notation

For this contract:

- A **word** is 32 bits; a **byte** is eight bits.
- \(W(x)\) retains the low 32 bits.
- \(L(x,n)=W(x\,2^n)\).
- \(B(x)\) retains the low eight bits.
- \(E(x)=((x\mathbin{\&}\text{0x3fffffff})\mathbin{\gg}20)\), with an unsigned shift.
- All displayed address arithmetic uses \(W\), including where wrapping is implicit.
- A **four-call chain** comprises four consecutive invocations with no explicit arguments represented. Its result, when used, is the fourth invocation’s primary result.
- A represented 64-bit result is separated into low and high 32-bit halves by significance, not by assumed memory endianness.

Wrapping formulas describe the 32-bit interpretation. Some supplied operations have signed-looking operands whose overflow and left-shift semantics are not independently established. These formulas do not assert portable signed-language behavior.

## Behavior

### Ordering and dependency notation

F-U001 through F-U030 identify ordered invocation groups, not necessarily distinct routines. An occurrence suffix identifies position within a group: F-U002-2 is that group’s second invocation.

Each step below is conditional on preceding calls returning and accesses completing. Calls may change memory subsequently read. Saved values remain distinct from later reads in the represented dataflow.

Some fixed target calculations produce equal numeric addresses. In particular, F-U002-2 and F-U003-1 have the same calculated target address. This equality does not establish stable code contents, a common signature, or equivalent behavior under different machine states.

### 1. Entry capture and initial indexed store

At each iteration, retain:

- **Entry byte**: the low byte of the current fourth argument.
- **Entry base**: \(L(\text{base seed},28)\).

F-U001 comprises an invocation receiving the current second argument twice and then the current third argument, followed by a four-call chain. Its first target is the current call-target seed plus \(\text{0xffe}\).

Call the chain’s result the **initial index**. Store the current second argument as a word at:

\[
W(4\,\text{initial index}+20).
\]

F-U002 receives \(L(\text{initial index},2)\), followed by a four-call chain. Retain its final result as the **first selector**.

Then write the entry byte at entry base plus 17, followed by entry base plus 1.

### 2. Repeated direct stores and selector lookup

F-U003 comprises:

- An invocation receiving \(L(\text{first selector},2)\).
- A no-explicit-argument invocation whose result is passed to the next invocation.
- That receiving invocation.
- A four-call chain.

Retain the final result as the **second selector**. Repeat the direct byte stores: entry base plus 17 first, entry base plus 1 second.

F-U004 receives \(L(\text{second selector},2)\), followed by a four-call chain. Retain the final result as the **table index**, and retain \(L(\text{table index},2)\) as the **table argument**.

Next:

1. Store \(L(\text{initial index},31)\) as a word at \(W(16\,\text{table index}+20)\).
2. Read a word at \(W(E(\text{first selector})+20)\), retaining the **selected word**.

The selector-derived read address ranges from 20 through 1043. No alignment adjustment is visible. The preceding store can affect the read if their locations overlap.

### 3. Selected-word dispatch and indexed clearing

F-U005 receives the table argument, then makes one no-explicit-argument invocation. Retain the latter result as the **dispatch value**.

F-U006 receives:

\[
L(\text{selected word}\mathbin{\gg}22,28),
\]

using an unsigned right shift, and then executes a four-call chain. Retain the **clear index**.

Store a zero word at \(W(4\,\text{clear index}+20)\).

F-U007 receives \(L(\text{clear index},2)\), then executes a four-call chain. Retain its final result as the **byte payload**.

### 4. Two indirect byte updates and a captured-address word store

Read a byte at entry base plus 1. Zero-extend it, add 17, and store \(B(L(\text{byte payload},2))\) at that resulting address.

F-U008 comprises an invocation receiving \(L(\text{byte payload},2)\), a no-explicit-argument invocation, an invocation receiving that result and \(L(\text{byte payload},30)\), and a four-call chain. Retain the chain’s result as the **next selector**.

Read entry base plus 1 again, retaining this byte separately as the **captured address byte**. Store \(B(E(\text{byte payload}))\) at captured address byte plus 17.

F-U009 receives \(L(\text{next selector},2)\), then executes a four-call chain. Its final primary result is not explicitly consumed.

Store \(L(\text{dispatch value},2)\) as a word at captured address byte plus 20. This uses the saved byte, not a fresh read, even if F-U009 changed the original byte location.

### 5. Global snapshot and first expanded value

After that word store, read an unresolved global word and retain the **global snapshot**.

F-U010 receives the table argument, then executes a four-call chain whose final result is represented as 64 bits. Define the **first expanded value** as its low half shifted left two bits.

Read entry base plus 1 again. At the zero-extended byte plus 17, store:

\[
B(\text{global snapshot}\mathbin{\gg}22).
\]

The global is not re-read after F-U010.

F-U011 comprises an invocation receiving the preceding 64-bit result’s high half, a no-explicit-argument invocation, an invocation receiving that result and \(L(\text{first expanded value},28)\), and a four-call chain. Retain its final selector.

Read entry base plus 1 again and store zero at the resulting byte plus 17.

F-U012 receives the selector shifted left two bits, then executes a four-call chain. Retain its final result as the **duplicated source**.

### 6. First expanded-address word propagation

F-U013 receives \(L(\text{duplicated source},2)\) twice, followed by a four-call chain. Define the **first stored word** as its final result shifted left two bits.

Store that word at:

\[
W(4096\,\text{first expanded value}+20).
\]

F-U014 receives \(L(\text{first expanded value},14)\), followed by a four-call chain. Store the same first stored word at four times the final result plus 20.

F-U015 receives four times that result, followed by a four-call chain producing a represented 64-bit result. Define the **second expanded value** as its low half shifted left two bits.

Read a byte at:

\[
W(16384\,\text{first expanded value}+1).
\]

At that byte plus 17, store \(B(E(\text{first stored word}))\).

### 7. Expanded-base clears

F-U016 comprises an invocation receiving the preceding 64-bit result’s high half, a no-explicit-argument invocation, and an invocation receiving that latter result.

F-U017 obtains another represented 64-bit result, passes its low and high halves to a two-argument invocation, and then performs two no-explicit-argument invocations. Retain the last result as a selector.

Read a byte at:

\[
W(2^{28}\,\text{second expanded value}+1),
\]

and store zero at that byte plus 17.

F-U018 receives the selector shifted left two bits, followed by a four-call chain. Retain the final result as the **middle index**. Define:

\[
\text{middle base}=L(\text{second expanded value},2).
\]

Read a byte at middle base plus 1 and store zero at that byte plus 17.

F-U019 comprises an invocation receiving \(L(\text{second expanded value},10)\), a no-explicit-argument invocation, an invocation receiving that result and \(L(\text{dispatch value},28)\), and a four-call chain.

Read middle base plus 1 again and store zero at that byte plus 17.

F-U020 receives F-U019’s final selector shifted left two bits, followed by a four-call chain. Its final primary result is not explicitly consumed.

### 8. Second expanded-address word propagation

Define:

\[
\text{middle scaled index}=L(\text{middle index},2),
\qquad
\text{second stored word}=L(\text{middle index},14).
\]

Store the second stored word at:

\[
W(4096\,\text{middle scaled index}+20).
\]

F-U021 receives \(L(\text{middle scaled index},14)\), then makes one no-explicit-argument invocation.

F-U022 receives \(L(\text{second stored word},28)\) and the unresolved secondary result, followed by a four-call chain. Store the second stored word at four times its final result plus 20.

F-U023 receives four times that result, followed by a four-call chain. Retain the **final table index**, and define:

\[
\text{final scaled index}=L(\text{final table index},2).
\]

Read middle base plus 1. At that byte plus 17, store \(B(E(\text{second stored word}))\).

### 9. Final indirect clear and word propagation

F-U024 receives \(L(\text{middle index},26)\), then makes one no-explicit-argument invocation.

F-U025 receives that invocation’s result and \(L(\text{final scaled index},28)\). It then obtains a represented 64-bit result, forwards its low and high halves to another invocation, and makes two further no-explicit-argument invocations. Retain the final selector.

Read middle base plus 1 again and store zero at that byte plus 17.

F-U026 receives the selector shifted left two bits, followed by a four-call chain. Define the **final propagated word** as its final result shifted left two bits.

Store that word at:

\[
W(4096\,\text{final scaled index}+20).
\]

F-U027 receives four arguments, in order:

1. \(L(\text{final scaled index},10)\).
2. The same value.
3. \(L(\text{final table index},14)\).
4. The final propagated word.

After its four-call chain, store the final propagated word at four times the chain’s result plus 20.

### 10. Late stores and iteration-state replacement

F-U028 receives four times F-U027’s final result, followed by a four-call chain. Define the **self-derived word** as four times its final result.

Store the self-derived word at sixteen times that final result plus 20. Then set the current fourth argument to zero.

F-U029 receives the self-derived word three times, followed by a four-call chain. Store the self-derived word at four times its final result plus 20.

F-U030 receives four times that result, followed by four no-explicit-argument invocations. Its internal state-update ordering is significant:

- Its first four invocations occur before the call-target seed replacement.
- Immediately before its fifth and final invocation, replace the call-target seed with the fixed code-relative value \(\text{0x0882d1fb}\).
- Then perform that final invocation.

Define the **next iteration word** as four times F-U030’s final result. Replace the current second argument with this word, store it at sixteen times that final result plus 20, and then replace the current third argument and base seed with the same word.

The unconditional repetition follows. There is no additional dependency invocation at the transition.

Under the represented state assignments, the next iteration’s first target calculation is:

\[
\text{0x0882d1fb}+\text{0xffe}=\text{0x0882e1f9},
\]

rather than a calculation from the original incoming call-target seed. This does not establish that intervening dependencies preserve the underlying machine state.

Under wrapping arithmetic, subsequent entry bases are restricted to 0, \(\text{0x40000000}\), \(\text{0x80000000}\), and \(\text{0xc0000000}\), because the replacement base seed is a multiple of four.

## Side effects

Per completed iteration, the visible body performs:

| Access | Count |
|---|---:|
| Direct entry-byte stores | 4 |
| Byte-derived-address byte stores | 10 |
| **Total byte stores** | **14** |
| Byte reads | **10** |
| Word stores | **13** |
| Word reads | **2** |

The word reads are the selector-derived read and the global snapshot. Counts exclude dependency-internal accesses and instruction fetches.

Each byte-derived destination uses a zero-extended byte as an address component, without another pointer dereference. Its byte-store address is between 17 and 272 inclusive. The captured-byte word-store address is between 20 and 275 inclusive.

Byte reads can observe earlier direct stores, indirect stores, word-store overlap, or dependency effects. Repeated reads and stores remain separate operations even when addresses and values happen to match.

There is no established non-aliasing guarantee. Accesses can overlap one another, the unresolved global, or dependency-accessible storage. Word alignment and byte order remain unresolved. Stores are not transactional.

The body visibly changes the evolving arguments, base seed, and late call-target seed. No explicit allocation, ownership transfer, deallocation, synchronization, or error reporting appears. Dependencies may nevertheless perform such operations.

## Dependencies

F-U001 through F-U030 identify invocation groups; suffixed IDs identify individual occurrences. IDs do not imply distinct target addresses or routines.

Established interface evidence includes explicit one-, two-, three-, and four-argument invocations, repeated invocations with no explicit arguments, word-valued results, and four represented 64-bit results.

“No explicit arguments” describes only the supplied representation. It does not establish a zero-argument ABI, absence of hidden inputs, or absence of secondary outputs.

Fixed expressions establish some numeric target equalities, including F-U002-2 and F-U003-1. Numeric equality is narrower than routine identity: code stability, entry-mode interpretation, signatures, hidden state, and behavior remain unresolved.

Dependency target validity, memory effects, ownership effects, failure behavior, and machine-state preservation are **documented_only**.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Input truncation.** With fourth argument \(\text{0x123456ab}\), all four direct entry-byte stores write \(\text{0xab}\), if reached.

2. **Repeated direct writes.** Even without intervening memory changes, both ordered pairs of direct stores occur. Replacing them with a single pair changes the access sequence.

3. **Fresh byte reads.** If successive address-byte reads produce 3 and 9, the corresponding byte-store destinations are 20 and 26. Reusing the first read is not equivalent.

4. **Captured word destination.** If the captured address byte is 7, the subsequent captured-address word store starts at 27, even if F-U009 changes the original byte location.

5. **Single-bit payload.** Under the 32-bit interpretation, an even initial index produces zero for the initial-index-left-31 store; an odd index produces \(\text{0x80000000}\).

6. **Extraction and truncation.** A byte payload of \(\text{0x3ff00000}\) gives \(E=\text{0x3ff}\), but the corresponding byte store writes only \(\text{0xff}\).

7. **Wrapped argument.** The second stored word is already shifted left 14 bits. Under wrapping arithmetic, shifting it another 28 bits for F-U022 produces zero. The invocation and its secondary argument still remain observable.

8. **Target equality.** F-U002-2 and F-U003-1 calculate equal numeric invocation addresses. Different explicit inputs and surrounding state prevent inferring equal results or effects.

9. **Target replacement timing.** Once F-U030’s first four invocations return, the fixed seed assignment precedes its fifth invocation. If that fifth invocation does not return, the assignment has still been reached; the next-iteration word store has not.

10. **Subsequent iteration.** If an iteration completes and dependencies preserve the represented transition state, the next iteration uses entry byte zero, equal second and third arguments, and first target address \(\text{0x0882e1f9}\).

11. **Partial failure.** A fault after earlier stores supplies no visible rollback. Effects already completed are not undone by this body.

## Uncertainties

### Established static evidence

The unconditional repetition, represented widths, explicit masks and unsigned shifts, access ordering, saved-value uses, numeric target equality example, late call-target replacement, and iteration-state replacements are visible.

### Inferred intent

The scaled indices and recurring offsets may suggest encoded references or structured storage. Neither interpretation is established or required for acceptance.

### Documented-only unknowns

- Accuracy of the represented function boundary and public signature.
- Origin and validity of incoming hidden state.
- Exact dependency identities, signatures, and hidden inputs or outputs.
- Producer and meaning of the unresolved secondary result.
- Target validity, instruction-set entry interpretation, and code stability.
- Dependency preservation or modification of machine state.
- Endianness, unaligned-access handling, and memory mappings.
- Whether represented signed arithmetic implements wrapping operations.
- Aliasing between the global and other accessed storage.
- Dependency ownership, allocation, release, and failure behavior.
- Significance of annotations about omitted unreachable material and overlapping global symbols.

These unknowns preclude an unconditional safety claim or a portable high-level execution contract.

## Evidence

This contract derives solely from the supplied static representation. Embedded commentary, the prior draft, and reviewer feedback were not treated as independent proof.

The access totals follow from four direct byte stores, ten byte-read/byte-store pairs, thirteen word stores, and two word reads. Target equality follows from comparing the relevant fixed target calculations. The later first-target calculation follows from the seed replacement immediately before F-U030’s final invocation.

No runtime execution, tests, memory-map validation, fault injection, or dependency inspection has been performed. Acceptance examples are derived conditions, not observed outcomes.
