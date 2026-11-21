<!-- Trusted provenance: F-b872f3fbd55f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b872f3fbd55f

## Purpose

F-b872f3fbd55f performs an ordered sequence of global-memory updates and dependency calls, constructs and repeatedly overwrites a small memory record, and then enters an unconditional recurring call-and-write cycle.

**Established static evidence:** the supplied representation has no explicit parameters, no returned value, no normal return path, and no locally implemented error recovery. Its observable behavior includes absolute-memory accesses, derived-address accesses, calls with explicitly supplied values, and repeated writes to overlapping record layouts.

**Inferred intent:** the long initial call sequence may represent initialization or an execution exercise. The recurring section may represent ongoing processing. Neither interpretation establishes an application-level purpose, device protocol, or meaningful record type.

This contract distinguishes what the supplied representation expresses from properties that require dependency definitions, an execution model, or runtime observation.

## Interface

### Inputs and outputs

The represented interface accepts no explicit arguments and supplies no return value.

Implicit inputs include:

- Two 32-bit global words at byte addresses 0x20 and 0x30.
- Memory reached through values read from those words.
- Two additional global locations written with zero during entry.
- Results of dependencies, including results represented as 64-bit quantities.
- Additional 32-bit call-associated register outputs whose semantic contracts are unavailable.
- Memory at absolute addresses 0x288 and 0x28c.
- Memory at several offsets from a later computed base.

The zeroed global locations are called the **interrupt-labelled location** and **unused-labelled location** here. These are descriptive labels, not established hardware semantics.

There is no explicit success result, error result, output-buffer argument, size argument, cancellation input, or ownership-transfer indicator.

### Widths and layouts

The supplied representation uses:

- 32-bit integer words and address-bearing values.
- 64-bit quantities whose low and high 32-bit halves are separately forwarded.
- A 64-bit store at the start of a recurring output record.
- 32-bit scalar and pointer stores following that store.

Record offsets below are byte offsets. Pointer stores are represented as 32 bits in this execution model. No byte order, bus transaction width, alignment-fault policy, or atomicity guarantee is established.

### Preconditions

For execution to reach any specified later stage:

- Every preceding dependency must return normally.
- Every preceding memory access must be valid under the actual execution environment.
- Derived call targets must be executable and callable under the applicable calling convention.
- The environment must provide meaningful values for call-associated outputs used by the function.

There are no visible null checks, bounds checks, address-range checks, or allocation checks. These conditions are execution prerequisites, not conditions that the function validates.

## Behavior

### Entry effects and alias-sensitive initialization

Before the first dependency call, the function:

- Captures the 32-bit word at address 0x20.
- Writes zero to the interrupt-labelled location.
- Writes a word at an address obtained by reading the word at 0x20 and adding 0x10. The stored value is also read from 0x20.
- Uses twice the initially captured word as the address of a 32-bit address-bearing word, then stores a word obtained from that same intermediate location through the address it supplies.
- Writes zero to the unused-labelled location.
- Reads the word at 0x30 and stores that word at an address equal to twice its value.

The captured initial value and later reads of 0x20 are distinct aspects of the representation. Aliasing with the zeroed locations or intervening destinations must not be ruled out.

Similarly, the indirect assignment involving the doubled initial value does not establish a single physical read of its intermediate location. Evaluation order between reads within one expression is not established. It does establish that the destination is indirect and that no pointer validation precedes the store.

The widths of the two labelled zero stores are not independently established by their displayed assignments.

### Ordered direct-call phase

The direct dependencies execute in the order identified under **Dependencies**. Calls without explicit arguments have no explicit arguments in the supplied representation; that does not establish the absence of implicit register or global inputs.

Several direct calls have additional data-flow requirements:

- F-4620e1344832 receives two 32-bit zeros.
- F-673ac3a35259 produces a represented 64-bit result. F-0a1488aa04cf receives its low half, its high half, and a 32-bit value formed by shifting its low half left by 12 bits.
- F-e3c2f94963b1 produces another represented 64-bit result. The next, unmapped dependency receives its low half, high half, and the earlier shifted value shifted right by 12 as a signed 32-bit quantity.
- After the final short direct-call sequence, F-b3ffbf60eb0b receives a call-associated 32-bit output from the preceding call, increased by four.

The signed right shift is important: it is not an unsigned recovery of the original low half. Bits already lost during the left shift cannot be recovered.

A value initialized to ten also gives rise to:

- An unsigned right shift by 12, yielding zero.
- A left shift by four, yielding 160.
- A later signed right shift of 160 by four, yielding ten.
- Later signed right shifts of 160 by 28 and 20, each yielding zero.
- A subsequent signed right shift of 160 by 16, yielding zero.

These are local value relationships in the representation. They are not claims that dependencies preserve global state.

### Computed-call phase and memory inputs

After F-b3ffbf60eb0b, execution proceeds through a long sequence of computed calls.

Targets are formed from embedded 32-bit address tokens by clearing their lowest two bits and adding a displacement. Most use displacement 0xf3c; later calls also use 0xefc. This is address arithmetic, not a table lookup through memory.

Distinct tokens can resolve to the same aligned target. Repeated calls therefore must not be treated as calls to necessarily distinct functions. The supplied dependency map does not identify these computed targets.

Observable data-bearing events within this sequence occur in this order:

1. A computed call receives ten.
2. A computed call receives 32-bit words read at absolute addresses 0x288 and 0x28c.
3. After further calls, another computed call receives freshly read words at those same absolute addresses.
4. A later computed call supplies a 32-bit result used to derive a new base by multiplication by 0x10000000.
5. After additional calls, a word is captured at base offset 0x228. Another call receives words read at offsets 0x21c and 0x220.
6. After further calls, a 32-bit result is forwarded together with the unsigned top nibble of the captured word.
7. Subsequent calls receive word pairs from offsets 0x44c/0x450 and then 0x67c/0x680.
8. A word is captured at offset 0x8b4. A record address is formed at base offset 0x8c0. Another call receives words at offsets 0x8ac and 0x8b0.
9. A call-associated unsigned 32-bit output is shifted right by 20. A later represented 64-bit result supplies the first record payload.

Reads in different events are not interchangeable. Dependencies between them may change memory. In particular, the two absolute-address word pairs are read separately rather than cached from the first pair.

The relative order of the two argument reads within any individual paired-argument expression is not established.

### Record publication before the recurring cycle

Let the **record address** denote base plus 0x8c0, and the **saved word** denote the earlier read from base plus 0x8b4.

The initial publication layout is:

| Offset from base | Width | Published value |
|---|---:|---|
| 0x8c0 | 64 bits | Current represented 64-bit result |
| 0x8c8 | 32 bits | Saved word |
| 0x8cc | 32 bits | Call-associated unsigned output shifted right by 20 |
| 0x8d0 | 32 bits | Zero, derived from the signed shift of 160 by 20 |
| 0x8d4 | 32 bits | Record address |

Each publication writes these fields in table order. The representation repeats the publication multiple times without an intervening call or source-value refresh.

Further computed calls supply another 64-bit payload. Its publication uses the same layout, except that offset 0x8cc now receives zero, derived from the signed shift of 160 by 16. This publication is also repeated.

After additional calls:

- One represented 64-bit result is split and forwarded to another dependency with its low half shifted left by 12 as the third argument.
- A subsequent represented 64-bit result is published repeatedly using the zero-valued layout.

Repeated publication is an observable sequence in the supplied representation, not an array append. The record address does not advance.

### Unconditional recurring cycle

The recurring cycle has no visible termination condition and no dependency-result test controlling continuation.

Each cycle includes the following data-bearing transitions, with computed calls between them:

- A represented 64-bit result is split into low and high halves. A signed 32-bit value four less than the high half is formed. A dependency receives the low half, high half, and adjusted high-half value.
- A call-associated 32-bit output is repeatedly published using a shorter layout:

| Offset from base | Width | Published value |
|---|---:|---|
| 0x8c0 | 32 bits | Call-associated output |
| 0x8c4 | 32 bits | Saved word |
| 0x8c8 | 32 bits | Zero |
| 0x8cc | 32 bits | Zero |
| 0x8d0 | 32 bits | Record address |

- Another represented 64-bit result is published using the longer layout. Its high half also appears in an overlapping compact layout: high half at 0x8c0, saved word at 0x8c4, zero at 0x8c8, and record address at 0x8cc. Longer-layout publications subsequently restore the longer arrangement.
- A further represented 64-bit result is split and forwarded with the earlier adjusted high-half value shifted right by 12 as a signed value.
- Another call-associated output is repeatedly published in the compact four-word layout.
- A final represented 64-bit result is published in both longer and compact arrangements. A five-word arrangement also appears before the final longer-layout publications.

Within each publication, stores occur in ascending listed offset order. Shorter publications do not clear trailing words that they omit. Consequently, different layouts overlap and can leave residual values from earlier publications.

After the final longer-layout publications, execution proceeds directly to the next cycle.

## Side effects

### Memory and state

Direct writes affect the entry globals, derived destinations, and the record region from base offset 0x8c0 through 0x8d7.

The function also invokes dependencies whose side effects are unavailable. No claim is made that those calls preserve:

- Global words or memory-mapped state.
- Memory later read by this function.
- Previously written record contents.
- External resources or ownership relationships.

The initial zero write to the interrupt-labelled location does not establish that interrupts remain disabled, that hardware interrupt control occurs, or that later calls cannot change that location.

### Ownership and concurrency

No direct allocation or release operation is identified. The record address is derived from an integer result, not from an established allocator contract.

Publishing that address into the record does not establish ownership transfer, reference counting, lifetime extension, or permission to release it. Dependencies may manage resources, but their behavior is unknown.

There is no visible synchronization around the multi-field publications. A concurrent observer may encounter intermediate layouts unless the actual environment supplies stronger guarantees. A represented 64-bit assignment is not proof of an atomic 64-bit memory transaction.

### Errors and progress

There is no visible local error reporting, rollback, retry policy, timeout, or cleanup path.

A dependency that does not return prevents subsequent events. Invalid memory or call targets may cause environment-specific faults; their exact effects are undocumented. If calls return normally and accesses succeed, the recurring cycle continues indefinitely.

## Dependencies

All dependency IDs below are opaque. Their meanings, ownership behavior, and global-preservation guarantees are unknown.

The initial ordered direct-call sequence is:

F-4620e1344832 → F-9980d0788a2b → F-c5e4d08fe5fe → F-3e81f119b631 → F-5769623ebe14 → F-ebc1c7c3943f → F-fac498071031 → F-4e720f9e7f4e → F-ab54501c66a2 → F-73f72226f9dd → F-e89c7e408cde → F-f02eb0b457d4 → F-963edeaaa848 → F-237cceee0b67 → F-33431aa3ece5 → F-38799630b0da → F-dcde8bb56963 → F-ccd27cb367dc → F-819827103479 → F-7e0b9101ac5f → F-f7cfba3e4834 → F-428b3cb037ed → F-61a813a5c339 → F-2f5bafbb90de → F-d2fdab904e8c → F-64830d130d80 → F-585031876a28 → F-e014c51ed3d3 → F-9289be9260ff → F-3864e2a59f9a → F-b2337b5a653b → F-f9da9bfae206 → F-79f85d3078fb → F-7849600b69bc → F-4e4b8124be04 → F-42b177e4803d → F-49788ee8c2f0 → F-b48e174c1816 → F-f6faadc98024 → F-ce083b379b88 → F-b3b41cb35474 → F-d6062e20f431 → F-6ffc922d5e42 → F-a56b6407f53e → F-0f1d49e0efaf → F-cd37d86ba3dd → F-047472f951f4 → F-c8540e6e44ee → F-217bd8dfc43a → F-0692e5fca513 → F-bee932b110e9 → F-a57057e76408 → F-7aa7b401d884 → F-ac7e46c495bc → F-82b7e694ce32 → F-0dbd4fcf512c → F-e1ff00b5e4f3 → F-6007f5aa6e42 → F-1ffc71802c88 → F-b4416c8c3cd5 → F-701f4a4949ef → F-887251a9a0d0 → F-90d97f334de4 → F-268e55560d85 → F-a39341e608f1 → F-734df0d1cfe7 → F-38ed702542b5 → F-71a75019d028 → F-6fe58a208e82 → F-d034f959020a → F-9ec245403ed8 → F-ab9a4cd4ca01 → F-1204c05a373c → F-025624191b72 → F-b8ef8e69c288 → F-140e03798a13 → F-466f83957ce4 → F-dfa34ff1c743 → F-ac39eb9c0f77 → F-e7d4d0003037 → F-40aa3d9f7e0a → F-8823e35b0ad0 → F-d8df282c694e → F-8e2f45d308d7 → F-c21976f3419a → F-85bee4a85f50 → F-df990b2a828a → F-f5407114d826 → F-ebc08fd533d5 → F-7b5aa666eecd → F-6c6590bc7dce → F-9c3675279ef6 → F-c4596d5d3372 → F-474c940ac2af → F-de0867e0a74a → F-2f1e6bf339e5 → F-e75b788d99f4 → F-c783949bf42e → F-916a27466c97 → F-b7ff7665dfb9 → F-1137f296b623 → F-aec5ea935b1a → F-bb879b0cb3b7 → F-e8d190b571ad → F-ab3f6b3fa8ee → F-da7d5f1345f9 → F-66101e60f4d3 → F-673ac3a35259 → F-0a1488aa04cf.

The sequence then continues:

F-1dbb66a1bf5e → F-7d60495f9672 → F-486cfb9cf44c → F-018974bb5028 → F-11fc88fca488 → F-159bc3dbfe95 → F-f6ce70656a95 → F-ea50a5741a70 → F-2e81f02c3992 → F-ec448ee9e833 → F-76e9cbc99cf0 → F-8684bc695032 → F-3a53d54358d3 → F-79e2f8e2bf2f → F-4d26902d6113 → F-6a48a5e6929b → F-4da10377a1f9 → F-7d045da09887 → F-2b65a81b6b32 → F-897e5d90f991 → F-b2232fc166ac → F-d4601a9e34f1 → F-814a04b60333 → F-a852b60a6e6a → F-8be329dffeee → F-67235bb6acd7 → F-7bbe7e4748f2 → F-21dd4f513467 → F-0097d3daeeab → F-956031af5c83 → F-fda29c7f284d → F-d0a8c7d8ea45 → F-11d944cb6c41 → F-16a7d396b7b2 → F-ebc063a6e5db → F-9b96ce6e7cc6 → F-007bf6eaad08 → F-645acb860441 → F-79b4e7103b93 → F-3c78916d2b64 → F-e7b9aa427432 → F-7ee62f6ed364 → F-f017a89c6d3f → F-c87593d0354d → F-605a1b650884 → F-9cd64ff7e00c → F-0df240cd9601 → F-9495a441d325 → F-2681eae4971c → F-ff738813fc0b → F-b8f7efa568d2 → F-5491f54909b4 → F-e3c2f94963b1.

Eighteen unmapped direct dependencies follow. Their document-local opaque IDs are U-001 through U-018, in call order. U-001 receives the split result and signed-shifted argument described above.

The remaining mapped direct calls are:

F-fdcd1d8cdc97 → F-7684741173a2 → F-06f4e36118df → F-2cb2117db9a9 → F-adc80b28c04f → F-3cc4eb2156cc → F-0ea7c3c52aa8 → F-90f463c3dd51 → F-4d0aacfa2558 → F-c6c1a97a371b → F-544822b86520 → F-b86df740454d → F-bd1859266f60 → F-674966a49c41 → F-91379f4bd2c8 → F-879af81c5010 → F-45944e44d04a → F-a5c86e3ec702 → F-d6099f155454 → F-d60a800d7002 → F-cb8ad047309c → F-d043b32e01f3 → F-3b47c842055f → F-04595dfeaf6e → F-aa7fc5f12663 → F-c0f9b7a588d6 → F-fff4619892aa → F-7d1e1916d53c → F-3a6a97bf6f22 → F-e72d13fe044d → F-d739ee8c0186 → F-d8d87e133b12 → F-f6b96a802a64 → F-a6adfccd1045 → F-9ef583d4c364 → F-48499dd75907 → F-b3ffbf60eb0b.

Computed-call dependencies remain unmapped. No semantic identity is inferred from their address proximity or repeated target calculation.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Early dependency does not return.** If F-4620e1344832 never returns, entry memory effects precede it, but none of the later direct calls or record publications is reached.

2. **Signed shift loses information.** If F-673ac3a35259 supplies low half 0x00080000, its immediate consumer receives third argument 0x80000000 under 32-bit bit-vector shifting. A later arithmetic right shift of that intermediate by 12 gives 0xfff80000, not 0x00080000.

3. **Repeated absolute reads can differ.** If intervening dependencies modify memory at 0x288 and 0x28c, the second paired call receives the later values. Matching the first pair is not required.

4. **Top nibble is unsigned.** If the word captured at base offset 0x228 is 0xf1234567, the explicitly forwarded shifted value is 15, not a sign-extended negative value.

5. **Record reuse is not append behavior.** Two successive payload results are written at the same record address. The second overwrites the first rather than advancing to another record.

6. **Compact layout leaves trailing state.** A compact publication ending at 0x8cc does not itself modify 0x8d0 or 0x8d4. Their prior contents remain unless another actor changes them.

7. **No normal completion.** If every dependency returns and all accesses succeed, reaching the recurring cycle does not eventually return to the caller merely because a payload or call result is zero.

## Uncertainties

The following are **documented_only**:

- The actual purpose of the function and the meanings of the global locations and record fields.
- Complete contracts for all dependencies, including unmapped direct and computed calls.
- Semantic provenance of call-associated register outputs.
- Exact multiplicities of repeated record publications and a complete event-by-event computed-call trace; this document specifies their ordering relationships and layouts but is not a complete bus-event oracle.
- Physical read counts and evaluation order within expressions containing multiple memory reads.
- Endianness, store atomicity, alignment requirements, memory-mapped-device behavior, and concurrency rules.
- Machine behavior for signed overflow and signed left shifts. Low-32-bit arithmetic examples are conditional on a 32-bit bit-vector execution interpretation; the representation alone does not establish a portable signed-arithmetic contract.
- Whether overlapping global symbols denote independent storage. The supplied material explicitly warns of overlap.
- Resource ownership, cleanup obligations, and failure behavior internal to dependencies.

No dependency-preservation, allocation, release, validation, or hardware-control guarantee is supplied to fill these gaps.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence comprises the displayed interface, explicit memory operations, explicit argument relationships, ordered direct calls, computed target formation, overlapping publication layouts, and unconditional recurring control flow.

Application-level initialization, processing, hardware control, and record semantics remain inferred or unknown as identified above. Comments and labels are not treated as authoritative descriptions of behavior.

No implementation access, runtime execution, smoke run, or runtime tests have been performed. No snapshot hash or independently verified provenance is asserted.
