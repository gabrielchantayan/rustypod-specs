<!-- Trusted provenance: F-83fec14ce0dc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-83fec14ce0dc

## Purpose

F-83fec14ce0dc performs an ordered sequence of indirect calls interleaved with memory accesses. It forwards input fields, scales dependency results, copies a retained word to computed destinations, writes link-like values, updates a shared word twice, and finally attempts a call through address zero.

The supplied representation establishes these operations, but not their application-level purpose. Allocation, list manipulation, device initialization, and decoding are not established interpretations.

This contract describes static behavior conditional on each preceding call returning and each preceding access succeeding. No runtime tests have been performed.

## Interface

### Explicit arguments and return

| Argument position | Represented width and interpretation | Direct use |
|---|---|---|
| First | 32-bit uninterpreted value | No direct use shown. |
| Second | 32-bit uninterpreted value | Forwarded to F-U001. |
| Third | Signed 32-bit integer | Left-shifted by 11 and 12 for separate calls. |
| Fourth | Unsigned 32-bit integer | Logically right-shifted by 23 for F-U003. |

The represented return type is void. No explicit status result or output-buffer argument is shown. A return appears after the final address-zero call; ordinary completion therefore requires that call to return.

### Hidden execution inputs

Execution also depends on:

- An incoming link-register value used to calculate the first call target.
- An incoming register-derived address used for an early byte read.
- An incoming unsigned 32-bit register value supplying an early byte write.
- A late dependency-exposed pointer used for a zero-byte store.
- A shared 32-bit word, called the **supervisor word** here.
- A separate shared 32-bit value used as a saved address.
- A shared 24-bit value read during final supervisor-word construction.
- Memory and executable targets selected by the represented address calculations.

These are execution requirements, not additional established source-language parameters. Physical overlap between the shared 24-bit value and the supervisor word is not established.

### Widths and arithmetic

Bytes, halfwords, and words contain 8, 16, and 32 bits respectively. Exactly three dependency results are represented as 64-bit quantities: F-U018, F-U070, and F-U080. Each is consumed through its low and high 32-bit halves.

Byte stores retain the low eight bits of the supplied value. Unsigned right shifts are logical. Word arithmetic and address calculations use represented 32-bit operands, except for extracting halves from 64-bit results.

Several scaling operations use signed 32-bit multiplication or signed left shifts. The representation does not establish defined behavior for all negative or overflowing operands. Statements explicitly conditioned on **32-bit modular semantics** describe bit-vector results, not a proven language or machine overflow policy.

Word and halfword accesses require whatever alignment and permissions the actual platform demands. No such requirements are checked by the visible body.

## Behavior

### Call notation and sequencing

F-U001 through F-U098 identify call occurrences in execution order. They are specification-local opaque IDs, not verified dependency identities. Different occurrence IDs do not establish different target functions.

“No explicit arguments” describes only the represented call expression. Such calls may still consume implicit state. An unused result does not imply an effect-free call.

No conditional branch, loop, retry, or recovery path is visible. The following phases form one ordered sequence, subject to interruption by a call or memory access.

### 1. Initial forwarding and byte selection

F-U001 receives the second explicit argument. Its target is calculated from the incoming link-register value plus 4,076.

F-U002 returns a 32-bit value. F-U003 receives that value followed by the fourth explicit argument logically shifted right by 23. F-U004, F-U005, and F-U006 follow without explicit arguments. F-U007 then returns a 32-bit value.

Before F-U008, the function reads an unsigned byte at the incoming register-derived address plus 1. The zero-extended byte becomes the **early byte base**. It is used directly as an address base; no table origin is added.

The next direct store writes bits 22–29 of the incoming unsigned register value to early byte base plus 17. Thus the destination lies between addresses 17 and 272 inclusive.

F-U008 receives the F-U007 result left-shifted by two.

### 2. Supervisor snapshot and first word-to-byte copy

F-U009 through F-U011 are called without explicit arguments. F-U012 returns a 32-bit value. F-U013 receives two copies of that value left-shifted by two.

F-U014 returns an unsigned 32-bit value. The function retains it as the **supervisor snapshot** and writes it to the supervisor word before calling F-U015 with the snapshot. F-U016 follows without explicit arguments.

After F-U016, the function reads a word at early byte base plus 116 and writes its low byte at early byte base plus 1. This is a fresh word read at that point in the sequence, not a value captured before the intervening calls.

### 3. Input scaling and repeated word-to-byte copies

F-U017 receives the third explicit argument left-shifted by 11.

F-U018 returns a 64-bit quantity. F-U019 receives its low 32-bit half, its high 32-bit half, and the supervisor snapshot logically right-shifted by 23, in that order. The two halves are represented as 32-bit integer arguments. The third argument uses the retained snapshot, not a new supervisor-word read.

F-U020 through F-U022 have no explicit arguments. F-U023 returns a 32-bit value; four times that value becomes the **first scaled base**.

The function reads a word at first scaled base plus 116 and writes its low byte at first scaled base plus 1. F-U024 then receives the first scaled base.

F-U025 through F-U027 have no explicit arguments. F-U028 returns a 32-bit value. Its two-bit left shift becomes the **retained aligned value**, represented thereafter as unsigned.

The function again reads a word at first scaled base plus 116 and writes its low byte at first scaled base plus 1. This second read may observe changes made since the first read.

F-U029 receives four arguments:

1. The third explicit argument left-shifted by 12.
2. The retained aligned value.
3. The retained aligned value again.
4. The F-U028 result left-shifted by 18.

### 4. Retained data capture and two word copies

F-U030 returns a word forwarded to F-U031. F-U032 returns another word forwarded to F-U033.

F-U034 through F-U036 have no explicit arguments. F-U037 returns a 32-bit value. Four times that result becomes the **second scaled base**.

The function reads a word at second scaled base plus 116 and retains it as the **data word**. It writes the data word’s low byte at second scaled base plus 1, then calls F-U038 with the second scaled base.

F-U039 through F-U041 have no explicit arguments. F-U042 returns a 32-bit value. The function stores the retained data word at sixteen times that result plus 20.

F-U043 receives the retained aligned value logically right-shifted by two and then left-shifted by 16. This staged calculation is part of the represented behavior. Under 32-bit modular shift semantics, its resulting bits equal those of the original F-U028 result left-shifted by 16: the bits lost in the initial two-bit shift would also be discarded by the final sixteen-bit shift. This equivalence does not resolve the original signed-shift uncertainty.

F-U044 returns a word forwarded to F-U045. F-U046 returns a word forwarded to F-U047.

F-U048 through F-U050 have no explicit arguments. F-U051 returns a 32-bit value. The function stores the retained data word at four times that result plus 20, then calls F-U052 with four times the same result.

Both word-copy stores use the captured data word. Neither rereads its original memory source.

### 5. Link-like store and unconditional word clear

F-U053 through F-U055 have no explicit arguments. F-U056 returns a 32-bit value whose two-bit left shift becomes the **first link value**.

The function stores this value at the retained aligned value, masked to clear its bottom two bits, plus 20. For an aligned value produced by a two-bit bit-vector left shift, this mask leaves the value unchanged.

F-U057 receives the first link value twice.

F-U058 returns a word forwarded to F-U059. F-U060 returns a word forwarded to F-U061.

F-U062 through F-U064 have no explicit arguments. F-U065 returns a 32-bit value. The function stores a zero word at four times that result plus 20, then calls F-U066 with four times the same result.

The zero-word store has no visible guard. In particular, no comparison prevents it from overwriting an earlier destination.

### 6. Two further link-like stores and saved-address use

F-U067 through F-U069 have no explicit arguments. F-U070 returns a 64-bit quantity.

Its low half, represented as a signed 32-bit integer and left-shifted by two, becomes the **second link value**. Its high half is represented as a signed 32-bit destination index. The function stores the second link value at four times that index plus 20.

Only after that store does it read the separate shared 32-bit address value, retaining it as the **saved address**.

F-U071 receives:

1. The second link value.
2. The high half of the F-U070 result.
3. The second link value.
4. The second link value.

F-U072 through F-U074 have no explicit arguments. F-U075 returns a 32-bit value. The function stores the second link value at four times that result plus 20.

It next reads a word at saved address plus 20 and retains it as the **saved payload**. The address uses the earlier shared-value snapshot even if intervening calls have changed that shared value. The payload itself is read now, after the preceding store.

F-U076 receives four times the F-U075 result.

### 7. Two late byte lookups and stores

F-U077 through F-U079 have no explicit arguments. F-U080 returns a 64-bit quantity. Both halves are retained; the low half is represented as unsigned.

Before F-U081, the function logically right-shifts the retained data word by 23 and reads an unsigned byte at that result plus 1. It uses the byte directly as an address base and stores the saved payload’s low byte at that base plus 17.

The lookup address lies between 1 and 512 inclusive; the store address lies between 17 and 272 inclusive.

F-U081 receives the high half of F-U080 three times, followed by its low half left-shifted by two.

F-U082 returns a word forwarded to F-U083. F-U084 through F-U086 have no explicit arguments. F-U087 returns a 32-bit value.

Before F-U088, the function masks the retained low half of F-U080 to its bottom 30 bits, logically right-shifts it by 21, and reads an unsigned byte at the result plus 1. It stores bits 22–29 of the saved payload at the byte-derived base plus 17.

This second lookup and store have the same address ranges as the first. F-U088 then receives the F-U087 result left-shifted by two.

### 8. Final reconstruction, halfword clear, and null-target call

F-U089 through F-U091 have no explicit arguments. F-U092 returns a 32-bit value. F-U093 receives two copies of that value left-shifted by two.

F-U094 has no explicit arguments. F-U095 returns a word forwarded to F-U096. F-U097 then returns a byte-sized value.

After F-U097 returns, the visible ordering is:

1. Store a zero byte through the late dependency-exposed pointer.
2. Read the separate shared 24-bit global value.
3. Form a 32-bit word with that value in bits 8–31 and the F-U097 byte in bits 0–7.
4. Store the constructed word to the supervisor word.
5. Store a zero halfword at the retained low half of F-U080, masked to 30 bits and logically right-shifted by seven.
6. Attempt F-U098 through address zero, with three explicit zero arguments.

The 24-bit source is read after the zero-byte store. Its physical relationship to the supervisor word, the pointer destination, and other storage is unresolved. Consequently, neither its value before the byte store nor its physical independence can be assumed.

The concatenation specifies logical bit placement; it does not establish byte order in memory.

## Side effects

### Direct memory effects

The visible sequence contains:

- Three unsigned byte lookups.
- Four word reads whose low bytes are copied to byte destinations.
- One shared-address read and one subsequent saved-payload word read.
- One late 24-bit shared-value read.
- Eight byte stores, including the final zero-byte store.
- Six computed word stores at destinations ending in an added offset of 20.
- Two supervisor-word stores.
- One zero-halfword store.

These counts exclude dependency-internal effects. The ordering in Behavior governs alias-sensitive outcomes.

Computed destinations may coincide with each other, later sources, globals, or dependency-visible storage. Conceptual labels do not establish distinct allocations. No atomicity, rollback, synchronization, or transaction guarantee is visible.

### Ownership and validity

No direct allocation, release, ownership transfer, or lifetime extension is established. Passing a value to a dependency does not establish whether that dependency retains or releases anything.

Every access must be valid when it occurs. Earlier validity does not establish validity after intervening calls. The function performs no visible bounds, null, alignment, or permission checks.

### Errors and interruption

There is no explicit error result or recovery path. Invalid accesses, invalid targets, exceptions, and nonreturning calls have unresolved platform-dependent consequences.

If execution stops partway through, earlier effects are not visibly undone. In particular, the final address-zero call follows the final stores; failure there does not imply those stores were rolled back.

## Dependencies

| Consumed result shape | Call occurrences |
|---|---|
| 64-bit, split into two 32-bit halves | F-U018, F-U070, F-U080 |
| 8-bit | F-U097 |
| 32-bit | F-U002, F-U007, F-U012, F-U014, F-U023, F-U028, F-U030, F-U032, F-U037, F-U042, F-U044, F-U046, F-U051, F-U056, F-U058, F-U060, F-U065, F-U075, F-U082, F-U087, F-U092, F-U095 |
| No result explicitly consumed | All remaining occurrences |

F-U001 has the link-register-derived target described above. F-U002 through F-U097 use targets calculated from fixed code-location expressions. F-U098 explicitly targets address zero.

Actual dependency identities, implementations, signatures, and calling conventions are unresolved. No dependency is assumed to preserve globals, validate memory, transfer ownership, or return normally. Retained values and fresh memory reads are distinguished according to the supplied representation, not an independently verified register-preservation model.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless otherwise stated, calls return, accesses succeed, and arithmetic remains within its applicable defined range.

1. **Fourth-argument extraction.** A fourth argument of 8,388,608 supplies 1 as F-U003’s second argument. A value of 4,294,967,295 supplies 511.

2. **Early byte selection.** If the early lookup yields 5 and the incoming unsigned register value is 4,194,304, the following store writes byte 1 at address 22.

3. **Fresh repeated read.** If the first scaled-base word read yields `0x123456AB`, its following byte store writes `0xAB`. If intervening calls change the source to `0x123456CD`, the repeated read produces a store of `0xCD`, not reuse of `0xAB`.

4. **Retained data.** If the captured data word is `0x89ABCDEF`, both later word-copy stores write that value even if the original source changes. Later calls or aliasing stores may subsequently change either destination.

5. **Different destination scales.** F-U042 returning 3 selects address 68 for its word store. F-U051 returning 3 selects address 32.

6. **Staged shift equivalence.** Under 32-bit modular semantics, an F-U028 bit pattern of `0xC0001234` produces retained aligned value `0x000048D0` and F-U043 argument `0x12340000`. Directly shifting the original pattern left by 16 produces the same bits. This is not an assertion that the original signed shift is defined for that input.

7. **Saved address, later payload.** If the shared address value is 256 when captured, the payload read uses address 276 despite later changes to that global. If a preceding store aliases address 276, the read observes the memory value then present, not a pre-store payload snapshot.

8. **Mask boundaries.** If F-U080’s low half is `0xFFFFFFFF`, the second late lookup reads address 512 and the final halfword store targets `0x007FFFFF`. Successful execution still depends on platform alignment and access rules.

9. **Final construction order.** If, after the late zero-byte store, the 24-bit source reads as `0x123456` and F-U097 returned `0xAB`, the supervisor-word store writes `0x123456AB`. If aliasing makes the zero-byte store change that source, the subsequently read value must be used.

10. **Termination boundary.** Successful completion of the final stores does not authorize an early return. F-U098 must still be attempted through address zero with three zero arguments.

## Uncertainties

The following remain **documented_only**:

- Actual dependency identities, calling conventions, implicit inputs, and preservation rules.
- Executability and interpretation of calculated call targets, especially address zero.
- Exact signed-overflow, signed-shift, conversion, and address-wrapping behavior.
- Endianness, physical shared-symbol layout, and aliasing of the 24-bit source.
- Origin, validity, and lifetime of the late pointer.
- Whether accesses target ordinary memory, volatile storage, or devices.
- Application meaning of the supervisor word, bit fields, scale factors, and offset-20 stores.

Repeated scaling and offset-20 writes may suggest structured storage. That is inferred intent only; no object layout or ownership model is established.

## Evidence

Established static evidence consists of the supplied function representation: represented widths, call order, argument expressions, consumed result halves, memory-access widths, retained values, shared-state accesses, and the final address-zero call.

The final construction uses a separately represented 24-bit global read after the zero-byte store. The supplied overlap warning does not establish that this particular source occupies the supervisor word’s upper bytes.

No runtime tests, memory traces, dependency implementations, or independent calling-convention validation were provided. This contract does not establish executable correctness or real-platform reachability.
