<!-- Trusted provenance: F-5cbb2ab262ad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5cbb2ab262ad

## Purpose

F-5cbb2ab262ad is a state-dispatched operation that can read and modify caller-supplied storage containing unsigned 16-bit elements. It combines configuration observations, retained arithmetic state, and dependency results. One action performs three ordered stores; another computes an intermediate without directly storing into the buffer.

The established behavior is a set of transitions, not a conventional bounded traversal. The supplied representation does not establish a required allocation size, iteration count, termination guarantee, or application-level numerical transformation.

**Inferred intent:** an iterative numerical buffer transformation with flattened control flow. This interpretation does not justify replacing its arithmetic, refreshing retained products automatically, or assuming dependency preservation.

## Interface

The operation accepts one pointer to unsigned 16-bit elements and returns no value. There is no explicit length, capacity, stride, ownership flag, or error-output argument.

Elements occupy two bytes. Offsets in this contract are element offsets, not byte offsets. Byte order is not established.

### Widths and terminology

The working pointer is initially null. Three indices are maintained: two signed 32-bit indices, called the **first** and **second indices**, and one unsigned 32-bit **third index**.

Other state includes:

- A signed 32-bit accumulator and intermediate.
- Six retained address or control products, designated A–F.
- Six signed 32-bit helper states, H0–H5.
- Seven unsigned 32-bit transition states, R0–R6.
- An unsigned 32-bit XOR state and selector.
- An entry-retained seed.

A–E occupy signed 32-bit state; F occupies unsigned 32-bit state. These storage types do not make all contributing arithmetic signed.

Configuration roles are named as follows:

| Name | Role |
|---|---|
| S | Address and arithmetic scale |
| K | Arithmetic multiplier |
| Δ | Index adjustment |
| G, Q | Coefficients captured at every dispatch and sometimes reread |
| L | Multiplier applied to loaded or intermediate values |
| U | Arithmetic coefficient |
| N, O | Result-8 dependency-result multipliers |
| Z | Result-8 subtraction coefficient |
| Y | Dependency argument used in result 8 |
| I, R | Result-12 argument and returned-value multipliers |
| J, V, W | Control and retained-product configuration |

Additional configuration supplies seeds, adjustments, masks, and continuation words.

Unless explicitly retained, each configuration occurrence denotes a separate represented observation. Separate statements establish order; operands within a single expression do not establish a total observation order. These distinctions are not synchronization guarantees.

Unsigned 32-bit arithmetic wraps modulo \(2^{32}\). Low-word retention means discarding all but the lowest 16 bits, without saturation. Signed overflow, out-of-range signed conversions, and some shifts require target details not supplied here.

### Storage and ownership

Every executed access requires valid, suitably aligned storage at its actual address. No explicit null, bounds, capacity, or address-overflow check is established.

The body does not directly allocate, release, or persistently retain the input pointer. It passes the working pointer to F-d2d6758a9b1b. That dependency’s access extent, retention, ownership, and release behavior are unknown. Continued storage validity across dependency calls is a prerequisite, not a guarantee provided by this operation.

## Behavior

### Entry and dispatch

Entry observes the seed, initial H0, initial XOR state, and initial selector, in that order. Other working state starts at zero, including the pointer.

Before every action, the operation observes G, then Q, then a dispatch-offset word. The first two observations are retained for that dispatch. Adding the selector and offset as unsigned 32-bit values chooses the action.

Results 0–15 have explicit actions. Every other result performs action 3. Action 14 still follows the dispatch observations.

No established control-flow guarantee ensures pointer initialization before a pointer-using action.

### Helper conventions

F-870c4fcf7295 receives two arguments. Its second argument is always 2,147,483,647. Transitions consume a **secondary result**, not an assumed primary return value.

A helper call separates observations before and after it. Configuration observed afterward may reflect dependency-induced changes. The repeated second argument does not establish division or modular reduction.

### Initialization and helper-state maintenance

**Action 2** determines its next selector before calling the helper: it XORs the entry-retained seed with a fresh configuration word. The helper receives H0 plus a fresh adjustment.

Afterward, configuration initializes these states in order:

H1, R1, H2, R2, R3, R4, R5, H4, H3, H5, R6, R0.

The working pointer then becomes the supplied pointer. Another word is captured, H0 receives the helper’s secondary result, and the XOR state is updated with that captured word. Dispatch proceeds using the selector determined before the call. Indices, accumulator, intermediate, and retained products are not reset.

**Action 0** resets only the first index and makes six helper calls:

| Call | First argument | Observations following the call |
|---|---|---|
| 1 | H0 minus a fresh adjustment | Capture an eventual XOR-state operand |
| 2 | H1 minus a fresh adjustment | XOR-update R0, then R1 |
| 3 | H2 plus a fresh adjustment | XOR-update R2, then R3 |
| 4 | H3 minus a fresh adjustment | XOR-update R4, then R5 |
| 5 | H4 plus a fresh adjustment | No intervening transition-state update |
| 6 | H5 plus a fresh adjustment | XOR-update R6 |

Every update operand is fresh. The first call’s secondary result becomes the next selector. Secondary results from calls 2–6 are retained and committed to H1–H5 after the calls. H0 remains unchanged. Finally, the XOR state is updated with the operand captured after call 1.

Resetting the first index does not refresh A.

### Branch decisions and retained products

Actions 3, 6, and 7 use width-sensitive arithmetic predicates. Their established construction combines adjusted products, bitwise masks, complements, and a signed high-bit extraction, then converts the sign outcome into zero or one. No ordinary comparison, loop bound, or higher-level invariant is established as equivalent.

**Action 3, including the default action:**

- Refresh D from J and the second index.
- Derive a mixing word by XORing a fresh J observation with the arithmetic-right-shifted value of another J observation; the shift count is 27.
- Refresh C and B by multiplying that mixing word by the third and second indices, respectively.
- Compute the branch predicate using D, the second index, another fresh J, and action-specific adjustment and mask observations.
- Call F-870c4fcf7295 twice, each time using H3 minus a separately observed adjustment.

Both calls occur regardless of the predicate. Predicate one selects the first secondary result; predicate zero selects the second.

**Action 7** refreshes E from J and the third index, then refreshes F by clearing in the third index the bits selected by W. Its predicate uses E, a V-weighted third index, and separate tuning observations. One tuning location is observed both as an arithmetic-right-shifted signed word, shifted by 29, and later as an unshifted mask.

Two helper calls use H2 minus separate adjustments. Both occur; predicate one selects the first secondary result, and zero selects the second.

**Action 6** computes its predicate from the first index and separate V observations. One V observation is shifted right unsigned by one before participating in the arithmetic. After computing the predicate, this action refreshes A from S and the first index.

It then observes two continuation operands, first and second. Each is XORed with the retained XOR state. Predicate zero selects the first-observed alternative; predicate one selects the second. No helper is called.

**Action 4** forms an unsigned high-bit predicate from the third index, two separate W observations, retained B, D, E, F, and a fresh S. It refreshes none of those retained products.

It observes two continuation operands in order and XORs each with R5. Predicate zero selects the first-observed alternative; predicate one selects the second.

### Remaining control transitions

| Action | State change and continuation |
|---|---|
| 1 | Reset the third index; continue with R1 XOR a fresh word. C, E, and F remain retained. |
| 5 | Replace the intermediate with the product of K, the accumulator, and S. Call the helper with H5 minus an adjustment; select its secondary result. |
| 9 | Continue with R0 XOR a fresh word. |
| 10 | Reset the second index and accumulator; continue with R3 XOR a fresh word. Associated products are not refreshed. |
| 11 | Call the helper with H1 minus an adjustment, then XOR-update R0 using a fresh word. Continue with the retained secondary result. |
| 13 | Subtract Δ from the second index and multiply by K and S. Separately replace the accumulator with the intermediate multiplied by fresh S and K observations. Continue with R6 XOR a fresh word. |
| 14 | Return without a value. |
| 15 | Subtract Δ from the first index and multiply by K and S. Continue with R2 XOR a fresh word. A remains unchanged. |

### Shared sample representation

The arithmetic actions repeatedly construct a low-word sample by multiplying an element or intermediate by a coefficient and retaining 16 bits.

A corresponding **complement mask** is made by multiplying the sample by a Q coefficient at 32-bit width, complementing that product, and forcing the low 16 bits to ones. Its own complement therefore has zero low bits.

This is an established bit-level representation, not evidence of saturation, normalization, or a known numerical encoding.

### Action 12: intermediate computation

F-d2d6758a9b1b receives the working pointer and two low-word integer arguments. Those arguments multiply fresh Q and I observations by the third and second indices, respectively. Each argument has its own Q and I observations; argument evaluation order is not established.

The primary return is multiplied by fresh R and narrowed to 16 bits. Its complement mask uses **dispatch-retained Q**.

Next, a fresh G is interpreted as signed and shifted right arithmetically by one. The resulting half-coefficient is retained for this action.

Two elements are then processed in order:

1. Offset \(3B + S \times \text{first index} + 1\).
2. Offset \(3S \times \text{third index} + A + 1\).

Each address uses a separate S observation. Each element has its own L multiplication and low-word retention, followed by a complement mask using dispatch-retained Q. These accesses occur after the pointer-taking dependency and can observe its effects.

The intermediate is replaced by a coefficient-weighted accumulation. Its established organization is:

- A returned-value contribution combines the narrowed return, its mask, and the retained half-coefficient, including a correction involving a fresh G and four times that half-coefficient.
- An element-difference contribution adds the first processed element’s weighted sample and mask terms and subtracts the corresponding terms for the second.
- Their product is scaled, added to the existing accumulator, and scaled again using the retained half-coefficient.

The two S factors are separate observations. Every unretained U and G occurrence is fresh; dispatch-retained G is not substituted here.

Finally, F-870c4fcf7295 receives H4 minus a fresh adjustment. Its secondary result becomes the selector. This action directly stores no buffer element, and replacing the intermediate does not replace the accumulator.

### Action 8: three-store transformation

This action retains dispatch-observed G for later coefficient mixing. It also makes fresh G observations where stated.

**Initial reads and pointer capture.** The anchor at offset zero is multiplied by L and narrowed. The first index multiplied by fresh S is retained for addressing. Define the common displacement as three times retained C plus that retained address product.

The anchor’s complement mask uses fresh Q. The first auxiliary pointer addresses displacement plus 10, and its element is transformed using fresh L. The second auxiliary pointer, at displacement plus 1, is established before computing the first auxiliary sample’s complement mask. The second auxiliary element is then transformed and complemented. Each complement has its own fresh Q observation.

Both auxiliary pointers exist before any dependency call. A fresh G is then interpreted as signed and shifted arithmetically right by one; this half-coefficient remains retained throughout the action.

**Initial dependency.** F-531bdc62ac41 receives 5, fresh Y, and a scale-weighted mixed word built from the three processed samples and their masks. The mix includes U-weighted sample contributions, G-weighted mask contributions, a three-times-G contribution, and complement-mask contributions weighted by the retained half-coefficient or K. Its four G occurrences are fresh, not dispatch-retained.

The primary result is multiplied by fresh N and narrowed, then complemented using fresh Q. The anchor is reread using fresh L and complemented using another fresh Q.

**First two-argument dependency and first store.** F-d7be9cfb4000 receives fresh Y and a scaled mixture containing the accumulator, the processed primary result, the reread anchor, and their masks. This mixture uses dispatch-retained G, the retained half-coefficient, and fresh U observations. A correction combines retained G with a U observation shifted by 29.

After the call, U is captured before the returned secondary result is multiplied by fresh N and O. The low 16 bits of that full-width product are stored at displacement plus 19.

The following observations are strictly interleaved:

1. Reread the first auxiliary element using fresh L and retain its low-word sample.
2. Observe fresh Q and compute that sample’s complement mask.
3. Observe fresh L and transform the **full preceding product**, retaining its low word.
4. Observe another fresh Q and compute this second sample’s complement mask.

Thus the order is **first sample → its complement → second sample → its complement**.

**Second call and second store.** F-d7be9cfb4000 receives fresh Y and a scaled mixture of these two sample/mask pairs. It uses retained G and the half-coefficient, subtracts fresh Z, weights the first sample with the previously captured U, and weights the second with fresh U.

Afterward, L is captured first and U second. The secondary result is multiplied by fresh O and N, and its low word is stored through the first auxiliary pointer.

Processing then occurs in this order:

1. Read the second auxiliary element using the captured L.
2. Observe fresh Q and compute its complement mask.
3. Transform the low word of the preceding product using fresh, post-store L.
4. Observe another fresh Q and compute its complement mask.

The captured L is not refreshed by the store. Again, each sample is complemented before the next sample is transformed.

**Third call and final store.** A third F-d7be9cfb4000 call uses the second call’s mixing pattern with the new samples and captured U, plus fresh unretained configuration.

After it returns, a full-width O observation is captured and interpreted as signed. The final store writes through the second auxiliary pointer. Its value retains the low 16 bits of a product whose factors are independently narrowed to signed 16-bit values: fresh O, the dependency’s secondary result, and fresh N.

After that store, the third index is updated using the retained half-coefficient, fresh post-store S, and an adjustment derived from the captured full-width O arithmetic-shifted right by 16. A continuation word is then observed and XORed with R4.

## Side effects

A completed action 8 directly stores at these offsets, in order:

1. Common displacement plus 19.
2. Common displacement plus 10.
3. Common displacement plus 1.

These destinations are distinct under valid pointer arithmetic. The anchor can coincide with one destination when the displacement is −19, −10, or −1. Negative offsets require valid surrounding storage; no independent check is established.

Configuration may alias writable storage. An earlier store can affect later configuration observations, but not retained captures. Dependencies may modify buffer contents or configuration; preservation is not assumed.

There is no established transaction, rollback, explicit error result, recovery, synchronization, or cancellation mechanism. A failure after a store need not leave storage unchanged.

## Dependencies

| Opaque ID | Established use | Unresolved behavior |
|---|---|---|
| F-531bdc62ac41 | Three arguments; first is 5; primary result feeds action 8 | Meaning, range, effects, failures |
| F-d7be9cfb4000 | Three ordered two-argument calls per completed action 8; secondary results feed stores | Arithmetic contract, secondary-result ABI, effects, failures |
| F-870c4fcf7295 | Transition helper with fixed second argument | Meaning, secondary-result ABI, preservation, failures |
| F-d2d6758a9b1b | Working pointer and two low-word arguments; primary result feeds action 12 | Access extent, ownership, retention, release, mutation, failures |

## Acceptance examples

These are conditional static examples, not executed tests.

- **Immediate return:** If the first dispatch selects 14, entry and dispatch observations occur, but no dependency call or buffer access occurs. Null input is not dereferenced on that path.
- **Initialization capture:** Changing seed configuration during action 2’s helper call does not change the selector already computed from the entry-retained seed.
- **Continuation ordering:** In action 4 or 6, if the first-observed alternative is 11 and the second is 23, predicate zero selects 11 and predicate one selects 23.
- **Unconditional calls:** In action 3 or 7, predicate zero selects the second secondary result, but both helper calls still occur.
- **Stale product:** Action 15 changes the first index without refreshing A. Action 12’s second address continues to use retained A until action 6 refreshes it.
- **Post-first-store interleaving:** The Q observation for the reread first auxiliary sample precedes the L observation used to transform the preceding full-width product.
- **Post-second-store interleaving:** The second auxiliary sample uses captured L; its Q complement observation precedes the fresh L observation for processing the preceding product’s low word.
- **Aliased configuration:** If the second store changes L, the next auxiliary load still uses captured L, while the subsequent product transformation uses the later observation.
- **Dependency mutation:** An anchor change made by F-531bdc62ac41 can be observed by the later anchor reread.
- **Narrowing:** A stored result with low word 65,535 writes that unsigned element value, not a saturated signed maximum.

## Uncertainties

The following remain **documented_only**:

- Configuration definitions, actual values, external mutability, and aliasing.
- Reachability, initialization-before-access, allocation extent, iteration count, and termination.
- Target treatment of signed overflow and out-of-range conversions.
- Extension behavior of the uncast U right shift by 29 in action 8.
- Dependency meanings, secondary-result ABI, ownership, storage lifetime, and failures.
- Concurrency, reentrancy, and application-level numerical meaning.

The mixed-word and predicate constructions are established as width-sensitive arithmetic. Their interpretation as familiar numerical operations or comparisons is unresolved. This prose contract does not provide a complete numeric oracle for those constructions and does not authorize algebraic simplification.

## Evidence

**Established static evidence:** the supplied representation supports the interface, widths, dispatch structure, observation boundaries, retained-state behavior, dependency ordering, addressing relationships, interleaved sample/complement processing, and three-store order described above.

**Inferred intent:** an iterative numerical buffer transformation. No stronger domain interpretation is established.

**Unavailable evidence:** configuration definitions, dependency implementations or contracts, target ABI documentation, caller allocation requirements, and runtime observations.

No runtime tests have been performed. No concrete transformed-buffer oracle, termination guarantee, or dependency-preservation guarantee is claimed.
