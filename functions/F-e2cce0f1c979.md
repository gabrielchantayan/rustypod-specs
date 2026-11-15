<!-- Trusted provenance: F-e2cce0f1c979; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e2cce0f1c979

## Purpose

F-e2cce0f1c979 exposes a linear prefix of indirect calls, byte accesses, and 32-bit word accesses. It copies numeric low bytes from words, uses loaded bytes to select low-address destinations, transforms returned values into addresses, and propagates words through several indexed destinations.

No application-level purpose is established. Repeated offsets and scaling factors do not establish an object model, tagging scheme, allocation protocol, or data format.

The supplied representation ends at an invalid-instruction boundary, without an ordinary return. This contract describes the recovered prefix, not a proven complete function.

## Interface

### Explicit inputs and output

Three explicit arguments are represented, each 32 bits wide:

- The first two have no visible direct use.
- The third is forwarded unchanged as the second displayed argument of the first dependency call.

No normal return value or successful-completion condition is established.

Additional incoming state is required:

- An **incoming base**, used for the first word read and byte store.
- An unsigned 32-bit **seed**, used in arithmetic and addressing.
- An incoming link-related address, from which the first indirect target is selected using displacement 0xfec.
- Additional register-derived values used later as dependency inputs or address indices. Their producers are not established.

These are unresolved interface requirements, not implicit zeroes. The explicit arguments alone do not determine behavior.

### Widths and notation

A byte is 8 bits; a word is 32 bits. A paired dependency result is represented as 64 bits, divided into low and high 32-bit halves.

Throughout the contract:

- **Low byte** means the least significant eight bits of a numeric value.
- Loaded bytes are unsigned and zero-extended.
- **Right n** means logical right shift by n bits.
- **Left n** denotes the represented left shift; the stored or passed word retains only 32 bits where that interpretation is established.
- A dependency ID used as an input denotes that occurrence’s returned word. “Low” and “high” identify halves of a paired result.
- Argument lists preserve displayed argument order.

Unsigned 32-bit arithmetic has modulo-\(2^{32}\) behavior. Some address calculations and shifts involve signed intermediates in the supplied representation. Overflow behavior for those expressions is not independently established; modulo-\(2^{32}\) interpretations of them remain conditional.

Word endianness, required alignment, address validity, and memory-mapped-device semantics are unknown. Numeric low-byte extraction does not require an endianness assumption; overlapping byte and word accesses do.

## Behavior

### Ordering and dependency notation

The following tables describe observable stages in execution order. Every memory event occurs after the preceding listed calls and before the following listed calls.

F-d001 through F-d165 are opaque identifiers for **call occurrences**, not claims about distinct target identities. A range denotes one call for every included ID, in increasing order. For example, F-d004–F-d007 denotes four calls.

“None displayed” means the representation supplies no explicit arguments. It does not establish an input-free machine-level interface. Results not identified as retained have no visible subsequent consumption.

Every later event is conditional on earlier calls returning and earlier accesses completing. There are no recovered branches, loops, retries, bounds checks, or explicit error-result checks.

### Entry exchange and first scaled copy

Let **doubled seed** be twice the unsigned seed, represented as a 32-bit value.

| Event | Established behavior or displayed inputs |
|---|---|
| Initial memory effect | Read a word at incoming base + 0x74; store its low byte at incoming base + 1. |
| F-d001 | Doubled seed; third explicit argument; seed right 2; incoming base. |
| F-d002 | None displayed; retain paired result. |
| F-d003 | F-d002 low; F-d002 high; F-d002 low again; seed masked with 0x7fffffff, then right 1. |
| F-d004–F-d007 | None displayed on each call; retain F-d007. |
| Scaled copy | Form **first scaled base** as four times F-d007. Read its word at +0x74; store the low byte at +1. |
| F-d008 | First scaled base. |
| F-d009–F-d012 | None displayed on each call. |
| First selector clear | Read a byte at doubled seed + 1; store zero at that byte’s value + 0x11. |

The initial memory copy precedes every dependency call. The first selector destination lies between 0x11 and 0x110, inclusive.

### Early selector reloads and retained bases

| Event | Established behavior or displayed inputs |
|---|---|
| F-d013 | An unresolved register-derived word. |
| F-d014 | None displayed; retain its result as **early base**. |
| F-d015 | Early base. |
| F-d016–F-d019 | None displayed on each call; retain F-d019. |
| Selector reload and store | Reload the byte at doubled seed + 1. At its value + 0x11, store doubled seed right 24, truncated to a byte. |
| F-d020 | F-d019 left 2. |
| F-d021–F-d024 | None displayed on each call; retain F-d024 left 2 as **retained scaled base**. |
| Word-directed clear | Read a word at early base + 4, retained as **address word**. Read a byte at address word + 1; store zero at that byte’s value + 0x11. |
| F-d025 | Retained scaled base, repeated three times. |
| F-d026–F-d029 | None displayed on each call; retain F-d029. |
| Shifted-word-directed clear | Read a byte at (address word right 2) + 1. Retain it as **small base**; store zero at small base + 0x11. |
| F-d030 | F-d029 left 2. |
| F-d031–F-d034 | None displayed on each call; retain F-d034. |

The two doubled-seed selector reads are separate observations. Likewise, the two address-word-directed byte reads use different addressing transformations and occur on opposite sides of five dependency calls.

### Word extraction and first indexed stores

| Event | Established behavior or displayed inputs |
|---|---|
| Ordered reads and copy | Read a word at retained scaled base + 4, retained as **field word**. Then read a word at small base + 0x74, retained as **propagated word**. Store propagated word’s low byte at small base + 1. |
| F-d035 | Propagated word; retained scaled base left 19 twice; F-d034 left 2. |
| F-d036 | None displayed; retain result. |
| F-d037 | F-d036; propagated word right 2. |
| F-d038–F-d041 | None displayed on each call; retain F-d041. |
| Scaled copy | Form **second scaled base** as four times F-d041. Read its word at +0x74, retained as **copied word**; store copied word’s low byte at +1. |
| F-d042 | Second scaled base. |
| F-d043–F-d046 | None displayed on each call. |
| Indexed store | Store copied word at four times an unresolved register-derived index + 0x14. |
| F-d047 | That unresolved index. |
| F-d048 | None displayed; retain result. |
| F-d049 | F-d048; the low nine bits of the ten-bit field left 21, repeated twice. |
| F-d050–F-d053 | None displayed on each call; retain F-d053. |
| Indexed field store | Store ten-bit field left 23 at four times F-d053 + 0x14. |
| F-d054 | Four times F-d053. |
| F-d055–F-d058 | None displayed on each call; retain F-d058 as **initial index**. |

The **ten-bit field** is field word right 22, so its range is 0–1023. Its low-nine-bit left-21 form has range 0–0x3fe00000. Its 32-bit left-23 form discards the field’s highest bit.

F-d049 has three displayed arguments: F-d048’s returned word is the first, not an unspecified dependency input.

### Index propagation and repeated word replacement

Define:

- **Initial scaled value**: four times initial index.
- **Masked index**: initial index masked with 0x3fffffff.
- **Quarter index**: masked index right 2.

| Event | Established behavior or displayed inputs |
|---|---|
| Indexed store | Store initial scaled value at sixteen times initial index + 0x14. |
| F-d059 | Masked index. |
| F-d060–F-d063 | None displayed on each call; retain F-d063 as **selected index**. |
| Indexed store | Store masked index at four times selected index + 0x14. |
| F-d064 | Four times selected index. |
| F-d065–F-d068 | None displayed on each call; retain F-d068 as **returned base word**. |
| Two ordered stores | Store the low byte of eight times selected index at propagated word + 1. Then store returned base word left 2 at eight times selected index + 0x34. |
| F-d069 | Returned base word left 2. |
| F-d070 | None displayed; retain result. |
| F-d071 | F-d070. |
| F-d072–F-d075 | None displayed on each call; retain F-d075 as a paired result. |
| Two ordered replacements | Mask returned base word with 0x3fffffff to form **masked base**. Store the low byte of eight times selected index at masked base + 1. Then store F-d075 high at eight times selected index + 0x34. |
| F-d076 | F-d075 low left 2. |
| F-d077–F-d080 | None displayed on each call; retain F-d080. |

The second store at eight times selected index + 0x34 replaces the earlier word only if execution reaches it and intervening effects have not changed the relevant storage semantics.

### Low-address writes and later scaling

Define **sixteenfold index** as sixteen times selected index.

| Event | Established behavior or displayed inputs |
|---|---|
| Two ordered byte stores | Store sixteenfold index’s low byte at (masked base right 20) + 0x11. Then store initial scaled value’s low byte at sixteenfold index + 0x0d. |
| F-d081 | F-d080 left 2, repeated twice. |
| F-d082 | None displayed; retain unsigned word result. |
| F-d083 | F-d082; F-d082 right 3. |
| F-d084–F-d087 | None displayed on each call; retain F-d087. |
| Repeated low-address effects | Store sixteenfold index’s low byte at absolute address 0x11. Then store initial scaled value’s low byte again at sixteenfold index + 0x0d. |
| F-d088 | F-d087 left 2. |
| F-d089–F-d092 | None displayed on each call; retain F-d092 as **scaling result**. |
| Masked-base stores | Store initial scaled value at masked base + 0x14. Then store the low byte of masked base left 1 at masked base + 0x0d. |
| F-d093–F-d095 | None displayed on each call; retain F-d095. |
| Scaled-base stores | Store scaling result left 7 at **thirty-twofold base** + 0x14. Then store the low byte of F-d095 left 2 at that base + 0x0d. |
| F-d096 | F-d095 left 2. |
| F-d097–F-d100 | None displayed on each call. |
| Extracted-address store | At **extracted value** + 7, store the low eight bits of twice quarter index. |
| Selector-directed store | Read a byte at thirty-twofold base + 1. At its value + 0x11, store extracted value’s low byte. |
| F-d101 | F-d095 left 21, repeated three times. |

Here:

- **Thirty-twofold base** is scaling result multiplied by 32.
- **Extracted value** is scaling result masked with 0x07ffffff, then right 17; its range is 0–1023.
- The payload at extracted value + 7 is exactly the low byte of **twice (masked index right 2)**. It is not twice masked index followed by a right shift.

### Captured selector and absolute overwrite

| Event | Established behavior or displayed inputs |
|---|---|
| F-d102 | None displayed; retain unsigned result as **saved word**. |
| F-d103 | Saved word. |
| F-d104–F-d107 | None displayed on each call; retain F-d107. |
| Capture and clear | Read and retain **captured selector**, a byte at thirty-twofold base + 1. Store zero at captured selector + 0x11. |
| F-d108 | F-d107 left 2. |
| F-d109–F-d112 | None displayed on each call. |
| Two byte stores | Store the low byte of saved word right 3 at thirty-twofold base + 0x0d. Then store captured selector at absolute address 0x0d. |
| F-d113 | Twice captured selector; 0x98; 0x260. |
| F-d114 | None displayed. |
| F-d115 | An unresolved unsigned register-derived word right 3. |
| F-d116–F-d119 | None displayed on each call; retain F-d119. |
| F-d120 | F-d119 left 2; retain its returned word. |
| Two byte stores | Store the low byte of saved word right 3 at **doubled-byte base** + 0x0d. Then store 0xc0 at absolute address 0x0d. |
| F-d121 | F-d120 left 2. |
| F-d122–F-d125 | None displayed on each call; retain F-d125. |

**Doubled-byte base** is twice captured selector, with range 0–510. The captured selector is retained across calls; it is not reloaded for these effects.

If captured selector is zero, doubled-byte base + 0x0d aliases absolute address 0x0d. The two stores still occur separately and in the stated order.

### Final word propagation

| Event | Established behavior or displayed inputs |
|---|---|
| Store, then read | Store F-d125 left 2 at doubled-byte base + 0x24. Then read a word at doubled-byte base + 0x74, retained as **loaded base word**. |
| F-d126 | Loaded base word; loaded base word left 1; F-d125 left 5; loaded base word masked with 0x7fffffff, then right 2. |
| F-d127–F-d130 | None displayed on each call. |
| Store, then dependency read | Store F-d125 left 5 at loaded base word + 0x24. Read a word at loaded base word + 0x74 for the next call. |
| F-d131 | The just-read word. |
| F-d132–F-d135 | None displayed on each call; retain F-d135 as a paired result. |
| Indexed store | At four times F-d135 high + 0x14, store saved word right 3 as a full word. |
| F-d136 | F-d135 low left 2, repeated three times; F-d135 high right 3. |
| F-d137 | None displayed; retain paired result. |
| F-d138 | F-d137 low; F-d137 high; F-d137 low again; F-d137 high right 3. |
| F-d139–F-d142 | None displayed on each call; retain F-d142. |
| Indexed store | At four times F-d142 + 0x14, store F-d135 low left 2. |
| F-d143 | Four times F-d142. |
| F-d144–F-d147 | None displayed on each call; retain unsigned F-d147 as **final word**. |

The full-word store derived from saved word right 3 is not truncated to the byte previously stored elsewhere.

### Final selector reloads and clears

Define **final scaled word** as final word left 2.

| Event | Established behavior or displayed inputs |
|---|---|
| Selector-directed store | Read a byte at loaded base word + 1; store final scaled word’s low byte at that selector + 0x11. |
| F-d148–F-d150 | None displayed on each call; retain F-d150. |
| Selector reload and store | Reload the byte at loaded base word + 1. At that selector + 0x11, store the low byte of final word masked with 0x3fffffff, then right 20. |
| F-d151 | F-d150 left 2. |
| F-d152–F-d155 | None displayed on each call; retain F-d155. |
| First final clear | Read a byte at 1024 times final scaled word + 1; store zero at that selector + 0x11. |
| F-d156 | Final scaled word left 3 twice; final scaled word; F-d155 left 2. |
| F-d157–F-d160 | None displayed on each call; retain F-d160. |
| Second final clear | Reload the byte at 1024 times final scaled word + 1; store zero at that selector + 0x11. |
| F-d161 | F-d160 left 2. |
| F-d162–F-d165 | None displayed on each call. |
| Endpoint | The recovered path terminates at an invalid-instruction boundary. |

Neither selector reload nor repeated clear may be eliminated based on equal addresses alone. Dependencies may change the selector, destination storage, or other shared state.

## Side effects

Established direct effects are byte reads and stores, 32-bit reads and stores, and the 165 ordered indirect-call occurrences above. Absolute stores include addresses 0x0d and 0x11. Byte-selected destinations occupy 0x11–0x110; other addresses can depend on arbitrary incoming state, loaded words, or dependency results.

Offsets identify access locations only. In particular, +1 and +0x0d are not established tags, and +0x14, +0x24, +0x34, and +0x74 are not established pointer fields.

There is no explicit allocation, release, reference counting, synchronization, or ownership transfer attributable directly to this operation. Dependencies may perform any of these. Passing a value or address does not establish borrowed, transferred, or retained ownership.

No rollback is visible. A fault, nonreturning call, or other interruption can leave any already-completed prefix observable. No stable final memory image is guaranteed: aliasing and dependency side effects can overwrite prior stores or alter subsequent reads.

## Dependencies

F-d001–F-d165 identify unresolved indirect-call occurrences solely to preserve their order, inputs, and consumed results. Distinct IDs may resolve to the same target; no target equivalence is established.

Apart from the entry target’s incoming-link-relative selection, targets are selected through embedded address-derived expressions. Their executable identities and contracts are unknown.

No dependency is known to preserve:

- Shared or pointed-to memory.
- Globals or additional register state.
- Ownership or allocation lifetime.
- Any state not explicitly retained by the recovered dataflow.

Displayed paired results occur at F-d002, F-d075, F-d135, and F-d137. This describes consumption in the supplied representation, not an independently verified return ABI.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Initial copy and interruption.** If the incoming word at +0x74 is 0x123456ab, the first direct store writes 0xab at +1 before F-d001. If F-d001 does not return, no later event is required.

2. **Unsigned entry arithmetic.** For seed 0x80000003, F-d001 receives first argument 0x00000006 and third argument 0x20000000. F-d003’s fourth argument is 1.

3. **Unsigned selector.** A selector byte of 0xff chooses address 0x110, not a sign-extended address.

4. **Reload across dependencies.** If the doubled-seed selector is initially 0x20 and becomes 0x21 during F-d013–F-d019, its clear targets 0x31 and its subsequent store targets 0x32.

5. **Ten-bit extraction.** If field word is 0xffc00000, ten-bit field is 0x3ff. F-d049 receives 0x3fe00000 in its second and third positions. Under the 32-bit shift interpretation, the indexed field store writes 0xff800000.

6. **Exact extracted-address payload.** If masked index is 7, quarter index is 1 and the byte at extracted value + 7 receives 2. Shifting twice masked index right by two would produce 3 and would not satisfy this contract.

7. **Complete word-propagation inputs.** If loaded base word is 0x80000005 and F-d125 is 3, F-d126 receives 0x80000005, 0x0000000a, 0x00000060, and 1.

8. **Byte versus word truncation.** If saved word is 0x12345678, the saved-byte stores write 0xcf. The later indexed full-word store writes 0x02468acf.

9. **Aliased absolute overwrite.** If captured selector is zero, the doubled-byte-base store targets 0x0d and is immediately followed by the separate 0xc0 store to that same address. Subsequent dependencies may still change it.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and actual calling convention.
- Dependency identities, contracts, and effects.
- Producers and executable liveness of unresolved register-derived values.
- Accuracy of displayed argument and result recovery.
- Overflow semantics of signed scaling and left shifts.
- Endianness, alignment requirements, valid address ranges, and device semantics.
- Allocation lifetime, ownership, concurrency, and reentrancy.
- Fault, exception, and error propagation.
- Whether the recovered endpoint corresponds to an executable trap and what lies beyond it.

Accessible memory, usable indirect targets, compatible dependencies, and sufficiently defined incoming state are necessary for progressing through the prefix. They are not proven sufficient preconditions for successful completion.

## Evidence

Established evidence consists of the supplied static representation’s access widths, explicit arithmetic, argument order, retained results, memory-event order, selector reloads, absolute-address stores, and truncated endpoint.

The tables preserve all displayed dependency occurrences, including calls whose results are not consumed. No purity or memory-preservation assumption is used to collapse calls or reads.

Repeated offsets and scaling factors suggest relationships between compact values and address-like values. That is inferred intent only; no application-level interpretation is established.

No runtime tests, executable observations, dependency inspections, or independent implementation checks have been performed.
