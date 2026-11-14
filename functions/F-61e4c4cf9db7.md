<!-- Trusted provenance: F-61e4c4cf9db7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-61e4c4cf9db7

## Purpose

F-61e4c4cf9db7 performs an unconditional sequence of indirect calls, calculated memory accesses, and global updates. Its observable behavior includes **138 call occurrences**, interleaved word stores, byte transfers, byte clears, and a terminal halfword clear.

No application-level purpose is established. The offsets, scales, and selectors suggest an encoded representation, but that is inferred intent only. They do not establish objects, arrays, allocation, serialization, or ownership conventions.

This contract describes the supplied static representation. Execution is conditional on every reached call returning and every preceding memory access completing.

## Interface

### Explicit inputs and completion

| Input | Width | Established use |
|---|---:|---|
| First input | 32 bits | No explicit use. |
| Second input | 32 bits | Argument to F-001. |
| Third input | Unsigned 32 bits | Its upper nine bits form the **input selector**. |

The input selector is the third input logically shifted right by 23; its range is 0–511.

No explicit return value is represented. Normal completion occurs only if the final call, F-138, returns. The first input’s apparent lack of use does not establish its irrelevance to unresolved machine-level calling conventions.

### Ambient inputs

Additional inputs include:

- An incoming link-register value, used to select F-001’s target.
- An incoming 32-bit ambient word, stored after F-007.
- Secondary outputs associated with F-012, F-024, F-040, and F-137.
- Dependency results, including three represented 64-bit results.
- The byte at absolute address 1.
- A global base word, a global publication destination, and a three-byte global quantity.
- Memory at all calculated read and write addresses.

No input-only validity predicate is established. Accessible destinations depend on unresolved results and mutable memory.

### Widths and arithmetic

A **word**, **halfword**, and **byte** contain 32, 16, and 8 bits respectively. A represented 64-bit result has a low word in bits 31–0 and a high word in bits 63–32.

Throughout this contract:

- An index’s **word slot** is at address 4 × index + 20.
- A **byte-selected destination** is obtained by reading a byte at a specified lookup address, zero-extending it, and adding 17.
- A word-to-byte transfer writes the word’s low eight bits.
- Right shifts of unsigned values are logical.
- Explicitly unsigned 32-bit arithmetic retains the low 32 bits where required.
- Signed index scales and signed left shifts are not assigned an invented overflow model. Examples involving them assume nonnegative operands and representable intermediate results unless stated otherwise.

Pointer width, address conversion, byte order, and alignment behavior remain unresolved.

## Behavior

### Reading the call ledger

F-001 through F-138 are opaque **call-site IDs**, not verified function identities. Their numeric order is execution order. Every occurrence is listed; no additional calls are implied between rows.

In the tables:

- **None shown** means no explicit arguments in the representation, not a proven zero-argument ABI.
- **—** means no primary result is explicitly consumed; other effects remain unknown.
- **Index** means a represented signed 32-bit result.
- **Word** means a represented 32-bit result.
- Values identified by their producing call are retained results, not repeated calls.

Memory events below each table specify their exact positions among the calls.

### 1. Initial dispatch and the shared secondary value

| Call | Explicit arguments | Consumed output |
|---|---|---|
| F-001 | Second input | — |
| F-002 | None shown | Word forwarded to F-003 |
| F-003 | F-002 result | — |
| F-004 | None shown | — |
| F-005 | None shown | — |
| F-006 | None shown | — |
| F-007 | None shown | Index |
| F-008 | 4 × F-007 index | — |
| F-009 | None shown | — |
| F-010 | None shown | — |
| F-011 | None shown | — |
| F-012 | None shown | **Shared secondary**, unsigned 32 bits |
| F-013 | Shared secondary | — |

F-001’s target is the incoming link-register value plus 3,896.

After F-007 and before F-008, the ambient word is stored in F-007’s word slot. Thus **four separate calls, F-004 through F-007, occur after F-003 and before this store**.

After F-012 and before F-013, the word slot selected by the shared secondary is cleared.

The shared secondary is one retained value. It selects this clear, is passed to F-013, and later supplies the masked address calculation in phase 2. Intervening calls do not make that later use a fresh secondary output.

### 2. Selector publication and first byte transfer

| Call | Explicit arguments | Consumed output |
|---|---|---|
| F-014 | None shown | Word forwarded to F-015 |
| F-015 | F-014 result | — |
| F-016 | None shown | — |
| F-017 | None shown | — |
| F-018 | None shown | — |
| F-019 | None shown | Index |
| F-020 | 4 × F-019 index | — |
| F-021 | None shown | — |
| F-022 | None shown | — |
| F-023 | None shown | — |
| F-024 | None shown | **Repeat secondary**, 32 bits |
| F-025 | Repeat secondary, repeat secondary, repeat secondary, zero | — |

After F-019 and before F-020, in order:

1. The input selector is stored as a full word in F-019’s word slot.
2. A word is read at 20 + ((shared secondary AND 0x1fffffff) logically shifted right by 19).

The second address has an unscaled offset of 0–1,023; no alignment adjustment is represented.

After F-024 and before F-025, the captured word is logically shifted right by 22 to form the **extracted selector**, ranging from 0–1,023. A byte is read at input selector + 1. The extracted selector’s low byte is written to that byte-selected destination.

The repeat secondary used by F-025 is distinct from the shared secondary retained from F-012.

### 3. Selector clears and retained shift source

| Call | Explicit arguments | Consumed output |
|---|---|---|
| F-026 | None shown | — |
| F-027 | Zero | — |
| F-028 | None shown | — |
| F-029 | None shown | — |
| F-030 | None shown | — |
| F-031 | None shown | Index |
| F-032 | F-031 index shifted left by 2 | — |
| F-033 | None shown | — |
| F-034 | None shown | — |
| F-035 | None shown | — |
| F-036 | None shown | Index |
| F-037 | F-036 index shifted left by 2, repeated twice | — |
| F-038 | None shown | Retained unsigned word |
| F-039 | F-038 result | — |
| F-040 | None shown | **Shift source**, signed 32-bit secondary |
| F-041 | Shift source, repeated three times | — |

F-027’s zero comes from a locally established zero shifted right by 23, not from the third input.

After F-031 and before F-032, a byte is read at 4 × extracted selector + 1; the corresponding byte-selected destination is cleared. The lookup address ranges from 1–4,093.

After F-040 and before F-041, the byte at absolute address 1 is read and its byte-selected destination is cleared. The shift source is retained for later calls.

### 4. Byte-base capture and global publication

| Call | Explicit arguments | Consumed output |
|---|---|---|
| F-042 | None shown | Word forwarded to F-043 |
| F-043 | F-042 result, F-038 result logically shifted right by 23 | — |
| F-044 | None shown | — |
| F-045 | None shown | — |
| F-046 | None shown | — |
| F-047 | None shown | Index |
| F-048 | F-047 index shifted left by 2 | — |
| F-049 | None shown | — |
| F-050 | None shown | — |
| F-051 | None shown | — |
| F-052 | None shown | Index |
| F-053 | F-052 index shifted left by 2, repeated twice | — |
| F-054 | None shown | **Published word**, unsigned 32 bits |
| F-055 | Published word | — |
| F-056 | None shown | — |

After F-047 and before F-048, address 1 is read again. Its byte is retained as the **byte base**, and the byte at byte base + 17 is cleared. This is a fresh lookup, independent of the earlier address-1 read.

After F-054 and before F-055, the published word is stored in the global publication destination.

After F-056 and before F-057, a word is read at byte base + 116, and its low byte is written at byte base + 1. The retained byte base is used even if dependencies have changed address 1.

### 5. Wide result and repeated field transfer

| Call | Explicit arguments | Consumed output |
|---|---|---|
| F-057 | Shift source shifted left by 11 | — |
| F-058 | None shown | First represented 64-bit result |
| F-059 | F-058 low word, F-058 high word, published word logically shifted right by 23 | — |
| F-060 | None shown | — |
| F-061 | None shown | — |
| F-062 | None shown | — |
| F-063 | None shown | Index |
| F-064 | First field base | — |
| F-065 | None shown | — |
| F-066 | None shown | — |
| F-067 | None shown | — |
| F-068 | None shown | Link index |
| F-069 | Shift source shifted left by 12, link base, link base, F-068 index shifted left by 18 | — |

The **first field base** is 4 × F-063’s index. After F-063 and before F-064, a word is read at first field base + 116; its low byte is written at first field base + 1.

The **link base** is F-068’s index shifted left by 2, retained as an unsigned 32-bit value after that represented signed shift.

After F-068 and before F-069, the read at first field base + 116 and byte write at first field base + 1 occur again. This second read must not be replaced by reuse of the first word.

### 6. Captured-word propagation

| Call | Explicit arguments | Consumed output |
|---|---|---|
| F-070 | None shown | Word forwarded to F-071 |
| F-071 | F-070 result | — |
| F-072 | None shown | Word forwarded to F-073 |
| F-073 | F-072 result | — |
| F-074 | None shown | — |
| F-075 | None shown | — |
| F-076 | None shown | — |
| F-077 | None shown | Index |
| F-078 | Capture field base | — |
| F-079 | None shown | — |
| F-080 | None shown | — |
| F-081 | None shown | — |
| F-082 | None shown | Index |
| F-083 | Link base logically shifted right by 2, then shifted left by 16 | — |
| F-084 | None shown | Word forwarded to F-085 |
| F-085 | F-084 result | — |
| F-086 | None shown | Word forwarded to F-087 |
| F-087 | F-086 result | — |
| F-088 | None shown | — |
| F-089 | None shown | — |
| F-090 | None shown | — |
| F-091 | None shown | Index |
| F-092 | 4 × F-091 index | — |

The **capture field base** is 4 × F-077’s index. After F-077 and before F-078, a word at capture field base + 116 is retained as the **captured word**; its low byte is then written at capture field base + 1.

After F-082 and before F-083, the captured word is stored at 16 × F-082 index + 20.

After F-091 and before F-092, the same captured word is stored in F-091’s word slot. It is not reloaded between these full-word stores.

F-083’s unsigned right-shift-then-left-shift expression is significant; it is not an unrestricted multiplication.

### 7. Link stores, clear, and saved-base capture

| Call | Explicit arguments | Consumed output |
|---|---|---|
| F-093 | None shown | — |
| F-094 | None shown | — |
| F-095 | None shown | — |
| F-096 | None shown | Index |
| F-097 | Link value, link value | — |
| F-098 | None shown | Word forwarded to F-099 |
| F-099 | F-098 result | — |
| F-100 | None shown | Word forwarded to F-101 |
| F-101 | F-100 result | — |
| F-102 | None shown | — |
| F-103 | None shown | — |
| F-104 | None shown | — |
| F-105 | None shown | Index |
| F-106 | 4 × F-105 index | — |
| F-107 | None shown | — |
| F-108 | None shown | — |
| F-109 | None shown | — |
| F-110 | None shown | Second represented 64-bit result |
| F-111 | Paired link, F-110 high word, paired link, paired link | — |

After F-096 and before F-097, the **link value**, F-096’s index shifted left by 2, is stored at (link base AND 0xfffffffc) + 20.

After F-105 and before F-106, F-105’s word slot is cleared.

After F-110 and before F-111, in order:

1. Its low word, interpreted as signed 32 bits and shifted left by 2, forms the **paired link**.
2. The paired link is stored in the word slot indexed by its signed high word.
3. A global base word is read and retained as the **saved base**.

The global-base read occurs after the paired-link store; aliasing can therefore affect it.

### 8. Saved word and final selector transfers

| Call | Explicit arguments | Consumed output |
|---|---|---|
| F-112 | None shown | — |
| F-113 | None shown | — |
| F-114 | None shown | — |
| F-115 | None shown | Index |
| F-116 | 4 × F-115 index | — |
| F-117 | None shown | — |
| F-118 | None shown | — |
| F-119 | None shown | — |
| F-120 | None shown | Third represented 64-bit result |
| F-121 | F-120 high word, repeated three times, then F-120 low word shifted left by 2 | — |
| F-122 | None shown | Word forwarded to F-123 |
| F-123 | F-122 result | — |
| F-124 | None shown | — |
| F-125 | None shown | — |
| F-126 | None shown | — |
| F-127 | None shown | Index |
| F-128 | F-127 index shifted left by 2 | — |

After F-115 and before F-116, the paired link is stored in F-115’s word slot. A word is then read at saved base + 20 and retained as the **saved word**. The address uses the earlier base snapshot; the memory contents are sampled only now.

After F-120 and before F-121:

- The captured word logically shifted right by 23 selects a lookup byte at selector + 1.
- The saved word’s low byte is written to the resulting byte-selected destination.

After F-127 and before F-128:

- A selector is formed from (F-120 low word AND 0x3fffffff) logically shifted right by 21.
- A byte is read at selector + 1.
- The low byte of saved word logically shifted right by 22 is written to the byte-selected destination.

Both selectors range from 0–511. The second transfer writes saved-word bits 29–22, not all ten bits produced by its shift.

### 9. Terminal operations

| Call | Explicit arguments | Consumed output |
|---|---|---|
| F-129 | None shown | — |
| F-130 | None shown | — |
| F-131 | None shown | — |
| F-132 | None shown | Index |
| F-133 | F-132 index shifted left by 2, repeated twice | — |
| F-134 | None shown | — |
| F-135 | None shown | Word forwarded to F-136 |
| F-136 | F-135 result | — |
| F-137 | None shown | Byte result and secondary pointer |
| F-138 | Zero, zero, zero | — |

After F-137 and before F-138, these explicit effects occur in order:

1. A zero byte is stored through F-137’s secondary pointer.
2. The three-byte global quantity is read and used as the upper 24 bits of a replacement publication word. F-137’s byte supplies its low eight bits.
3. That replacement word is stored in the same global publication destination written earlier.
4. A zero halfword is stored at (F-120 low word AND 0x3fffffff) logically shifted right by 7.

The halfword address ranges from 0–8,388,607 and need not be aligned. F-138 calls address zero. The routine returns only if that call returns.

## Side effects

Calls and memory operations are interleaved; their order is not interchangeable.

Byte-selected destinations range from 17–272. Other accesses may be unaligned. Calculated addresses can alias lookup bytes, earlier destinations, or globals. In particular, the terminal pointer clear can affect the subsequently read three-byte quantity.

The representation warns that global symbols overlap smaller symbols. Independent storage for separately described globals is not guaranteed.

No allocation, deallocation, ownership transfer, reference-count change, rollback, or cleanup obligation is established. A dependency receiving a value is not evidence that it owns or releases associated memory.

There is no explicit validation branch, status-dependent exit, or recovery path. Faults and nonreturning calls can leave earlier effects visible. Atomicity, transactionality, and synchronization are not established.

## Dependencies

The ledger identifies every unresolved call occurrence and its explicit argument list, consumed outputs, and ordering constraints.

Distinct IDs do not prove distinct targets; repeated-looking calls do not prove a shared target. Apart from F-001’s link-relative target and F-138’s zero target, target identities remain unresolved.

No dependency is assumed to preserve globals, memory, ambient registers, or ownership. Retained values describe the representation’s data dependencies, not an independently verified ABI. Unconsumed primary results do not make calls removable.

## Acceptance examples

These are conditional static criteria, not executed tests. Each assumes preceding operations complete and relevant arithmetic and accesses are supported.

1. **Initial call boundary.** After F-003 returns, F-004, F-005, F-006, and F-007 must occur before the ambient-word store. F-008 occurs after that store.

2. **Shared secondary continuity.** If the shared secondary is 0x00080000, the early clear selects word-slot address 0x00200014, F-013 receives 0x00080000, and the later masked word read selects address 21. An intervening secondary output must not replace this retained value.

3. **Input selector.** Third input 0x00800000 yields selector 1, publication value 1, and lookup address 2. Third input 0xffffffff yields selector 511 and lookup address 512.

4. **Distinct repeated argument.** If the shared secondary and repeat secondary differ, F-025 receives the repeat secondary three times followed by zero; it does not receive the shared secondary merely because that value selected the earlier read.

5. **Ten-bit extraction.** A masked-address word of 0xffc00000 yields extracted selector 1,023. The first byte transfer writes 0xff; the later selector-based clear reads its lookup byte at address 4,093.

6. **Fresh lookup and retained base.** If address 1 contains 7 after F-040, that clear targets address 24. If it contains 9 after F-047, the later clear targets address 26 and captures byte base 9. Subsequent changes to address 1 do not change the base used after F-056.

7. **Repeated field read.** Different words observed after F-063 and F-068 produce their respective low-byte writes. Reusing the earlier word is not equivalent.

8. **Captured propagation.** Captured word 0x123456ab produces byte 0xab and later full-word stores of 0x123456ab, without an intervening reload.

9. **Terminal replacement.** If the three-byte quantity is numerically 0x123456 when read after the pointer clear, and F-137 returns byte 0xab, the replacement word is 0x123456ab. This does not specify memory byte order.

10. **Terminal target.** Reaching F-138 attempts address zero with three zero arguments. Return remains conditional on that target being callable and returning.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings and valid object layouts.
- Dependency identities, signatures, and effects.
- Exact ABI provenance of secondary outputs and represented 64-bit results.
- Signed overflow, signed-left-shift behavior, and pointer conversions.
- Address width, byte order, memory mapping, and alignment faults.
- Whether low addresses denote ordinary memory, devices, or representation artifacts.
- Whether address zero is callable.
- Concurrency, reentrancy, and visibility guarantees.

## Evidence

Established static evidence comprises the supplied signature, 138 ordered call occurrences, explicit arguments, represented result dependencies, memory widths, masks, shifts, address expressions, global updates, and terminal call.

The shared secondary’s three uses are linked by the supplied data dependency, not inferred from similar values. The later repeat secondary is separately represented.

Encoded-layout interpretations are inferred intent only. Calling conventions, runtime validity, and application semantics are not established.

No runtime tests have been performed.
