<!-- Trusted provenance: F-4181fe6ae4fd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4181fe6ae4fd

## Purpose

F-4181fe6ae4fd defines an ordered, alias-sensitive memory transformation followed by two dependency calls and a terminal continuation. It also exposes a partially initialized stack region to the first dependency.

The established behavior is conditional: each memory access must complete, and each dependency must return, for later effects to occur. No ordinary return is established for the complete displayed path.

Application intent is unknown. Repeated offsets, address-valued payloads, and self-address stores do not establish an object format, collection operation, ownership protocol, or valid allocation boundary.

## Interface

### Explicit inputs

The modeled interface has two 32-bit integer arguments:

| Argument | Established use |
|---|---|
| Primary argument | Base for word reads at byte offsets 20, 100, and 116; operand of left shifts by 4 and 8 |
| Secondary argument | Base for a word store at offset 84; participant in an address calculation with the ambient displacement |

Neither argument is validated before use.

### Ambient inputs

The behavior additionally requires:

- An **entry base**, used both as an address and as a word payload.
- A **displacement**, used as an address base, an address displacement, and a word payload.
- An **auxiliary value**, used as an address and as a word payload.
- An indirect call target for F-indirect-01.
- A word-valued **global source**, evaluated twice during the memory transformation.
- A **final argument source**, evaluated for the call to F-cdec09f5d432.
- Accessible stack storage containing the region exposed to F-indirect-01.

The mechanism supplying these inputs is unresolved. They are not additional proven formal parameters.

### Access widths and arithmetic

All offsets in this contract are byte offsets.

- A byte access is 8 bits.
- A halfword access is 16 bits.
- A word access and the modeled integer values are 32 bits.
- Both address-bearing halfword reads are signed. Their values are sign-extended before subsequent arithmetic.
- Byte stores retain the low eight bits; the halfword store retains the low sixteen bits.

Byte order, actual pointer width, alignment requirements, and the arithmetic treatment of overflow are not established. In particular, this contract does not impose modular 32-bit addressing or a portable result for overflowing multiplication or signed left shifts.

### Outputs and failure conditions

No normal result value is established. Observable outputs are memory mutations, dependency invocations, and the terminal continuation.

There is no displayed bounds check, null check, alignment check, error result, rollback, or cleanup path. An access failure or non-returning dependency can prevent all later effects while leaving earlier mutations in place.

## Behavior

### Interpretation of the memory contract

The requirements below describe ordered observations and mutations, not merely a final memory image.

Captured values remain the values obtained at their capture points. Conversely, an expressly fresh observation must not be replaced by an earlier capture when intervening writes could change the addressed memory.

The contract does not assume that distinct role names denote distinct addresses. Externally derived addresses may overlap one another, global storage, or the exposed stack region. Equal addresses therefore do not justify removing repeated events.

### Captured values and derived roles

The following names describe values needed to state the behavior; they do not imply data types or object relationships beyond the listed accesses.

| Role | Established derivation and capture boundary |
|---|---|
| Saved byte | Low byte of the initial word read at primary argument plus 100 |
| Initial link | Initial word read at entry base plus 68 |
| Signed locator | Signed halfword read at entry base plus displacement |
| Branch base | Word read at signed locator plus 20, after the first three stores described below |
| Displacement snapshot | Word read at displacement plus 68, after the early halfword store |
| Global base | First evaluation of the global source |
| Traversal seed | Word read at branch base plus 68, after the global-based word update and saved-byte update |
| Signed factor | Signed halfword read at branch base plus global base, immediately after capture of the traversal seed |
| Replacement base | Result of a three-read traversal: offset 68 from the product of signed factor and traversal seed; offset 116 through that result; then offset 20 through the next result |
| Previous link | Word read at replacement base plus 20 before that location is replaced |
| Working product | Signed factor multiplied by a fresh word read at previous link plus 20, after the replacement |
| Retained word | Word read at working product plus 116 |
| Expanded product | Signed factor multiplied by the working product |
| Final destination base | Word read at primary argument plus 20 near the end of the external-memory transformation |

These roles are conditional on the necessary arithmetic and reads completing. Multiplication does not establish indexing, element sizes, or allocation membership.

### Memory-update obligations

The table groups effects by destination rather than presenting an implementation recipe. Ordering constraints follow separately.

| Destination | Required mutation |
|---|---|
| Secondary argument plus 84 | Word containing the entry base |
| Entry base plus 17 | Saved byte |
| Initial link plus 68 | First the displacement; later the value obtained from the second evaluation of the global source |
| Initial link plus 84 | Word containing the initial link itself |
| Displacement plus secondary argument | Word containing the auxiliary value |
| Displacement plus 20 | Word containing the initial link |
| Displacement plus 13 | Low byte of a fresh word read at auxiliary value plus 68 |
| Branch base plus 13 | Low byte of the initial link |
| Signed locator plus displacement | Low halfword of the initial link |
| Branch base plus 84 | Word containing the branch base itself |
| Global base plus 20 | Displacement snapshot |
| Branch base plus 17 | Saved byte |
| Signed factor plus 84 | Five separate word stores, each containing the signed factor itself |
| Global base plus 17 | Four byte stores: twice the low byte of the branch base, then the low byte of the previous link, then the low byte of the retained word |
| Replacement base plus 20 | Branch base shifted left by one bit |
| Global base plus 100 | Retained word |
| Global base plus 9 | Low byte of the earlier final-stage capture at expanded product plus 68 |
| Expanded product plus 17 | Low byte of a later, fresh word read at expanded product plus 68 |
| Twice the expanded product, plus 17 | Low byte of the retained word |
| Retained word plus 1 | Low byte of the final-stage word capture at primary argument plus 116 |
| Final destination base plus 17 | Low byte of the final product-based word capture described below |
| Final destination base plus 9 | Low byte of twice the expanded product |

The retained word serves as both a stored value and an address base. Its suitability for either role does not establish suitability for the other.

### Required ordering and freshness boundaries

**Initial observations precede all displayed stores.** The initial word at primary argument plus 100 is read first, the initial link second, and the signed locator third. The first mutations are, in order, the word at secondary argument plus 84, the byte at entry base plus 17, and the displacement at initial link plus 68.

**Branch discovery divides the early updates.** The branch-base read follows those three stores. It precedes, in order, the initial-link self-address store, the auxiliary-value store at displacement plus secondary argument, and the initial-link store at displacement plus 20. Only then is auxiliary value plus 68 read for the byte update at displacement plus 13. That byte update precedes the byte update at branch base plus 13, which precedes the halfword update at signed locator plus displacement.

**The global source is observed twice, after a potentially aliasing store.** The displacement snapshot is captured before the branch-base self-address store. That store precedes both global-source evaluations. The first evaluation establishes the global base; the second supplies the replacement payload at initial link plus 68. No displayed memory store separates those two evaluations.

The second global observation is not specified to reuse the first. They may agree in stable ordinary memory, but that equality is not an unconditional requirement.

After the update at initial link plus 68, the displacement snapshot is written at global base plus 20, followed by the saved byte at branch base plus 17. The traversal seed and signed factor are then captured, in that order.

**Repeated stores are separate events.** Before the three-read traversal, two identical pairs occur. Each pair consists of a signed-factor self-address word store followed by the branch-base byte store at global base plus 17. The observable order is word, byte, word, byte—not a single equivalent pair.

**Replacement precedes the dependent reread.** After traversal resolves the replacement base, its offset-20 word is captured as the previous link and replaced with the shifted branch base. The read at previous link plus 20 used for the working product occurs after that replacement. It can therefore observe the replacement when addresses overlap.

The third signed-factor self-address store follows computation of the working product and precedes the previous-link byte store at global base plus 17. The retained-word read follows that pair. Its assignment to stack offset 264 occurs at this point, before the remaining external-memory mutations.

The fourth signed-factor self-address store follows formation of the expanded product. The retained-word byte store at global base plus 17 follows it.

**The final stage contains two distinct expanded-product observations.** First, the word at expanded product plus 68 is captured; next, the word at primary argument plus 116 is captured. The retained word is then written at global base plus 100, followed by the low byte of the first capture at global base plus 9.

Only after those stores is expanded product plus 68 read again. Its current low byte supplies the store at expanded product plus 17. The fifth signed-factor self-address store follows, then the retained-word byte store at twice the expanded product plus 17.

The final product-based capture is a word read at offset 68 from the product of signed factor and expanded product. It precedes capture of the final destination base. The closing byte stores occur in this order:

1. At retained word plus 1, using the earlier primary-argument-plus-116 capture.
2. At final destination base plus 17, using the final product-based capture.
3. At final destination base plus 9, using twice the expanded product.

### Exposed stack layout

The pointer passed to F-indirect-01 defines stack-region offset zero. Every listed assignment is word-sized.

| Offset or ordered offsets | Assigned value | Timing |
|---|---|---|
| 264 | Retained word | At its capture, before the remaining external-memory updates |
| 16 | Primary argument shifted left by 4 | First stack preparation after the external-memory updates |
| 52 | Primary argument shifted left by 8 | Next |
| 228, 96, 168, 440 | Zero | In the listed order |
| 36 | Copy of the shift-by-4 result | Next |
| 76 | Copy of the shift-by-8 result | Next |
| 296 | Copy of the retained-word value | Next |
| 316 | Copy of the shift-by-8 result | Next |
| 380 | Copy of the retained-word value | Next |
| 392 | Copy of the shift-by-8 result | Last before F-indirect-01 |

This is partial initialization, not a complete record definition. Unlisted bytes have no established value.

The table specifies assignments, not an unconditional call-entry memory image. In particular, the early assignment at offset 264 may be affected by later aliased writes. The representation does not establish enough physical storage detail to resolve every consequence of aliasing with modeled local storage.

### Calls and continuation

F-indirect-01 receives the exposed stack-region pointer followed by two zero arguments. Its target comes from ambient state.

If that call returns, five word assignments occur at stack offsets 8, 412, 256, 288, and 320, in that order. Their values are respectively zero, 0x08400E1C, zero, zero, and zero.

The final argument source is evaluated after these assignments. F-cdec09f5d432 receives that current value followed by three zeros. An entry-time snapshot is not an equivalent substitute.

If F-cdec09f5d432 returns, F-terminal-01 is invoked with operands zero and 0x08401154. Its resulting target is then invoked with no displayed arguments. The displayed terminal sequence is marked non-returning; no portable exception class, exit status, or recovery behavior is established.

## Side effects

The established effects include narrow and word-sized writes, replacement of an existing word, repeated self-address stores, partial stack initialization, and dependency calls.

No allocation, release, reference-count operation, or ownership transfer is displayed. Passing stack storage to F-indirect-01 establishes exposure, not permission for indefinite retention. It also does not prove that the dependency refrains from retaining the pointer.

Neither ordinary dependency is known to preserve global storage, stack storage, or other reachable memory. Their possible memory-management effects remain unresolved.

No atomicity, synchronization, transaction boundary, or restoration of earlier memory is established.

## Dependencies

| Opaque ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-indirect-01 | Ambient indirect target; receives stack-region pointer and two zeros | Identity, calling convention, memory access, retention, ownership, errors, and return behavior |
| F-cdec09f5d432 | Receives the freshly evaluated final argument source and three zeros | Argument interpretation, memory effects, ownership effects, errors, and return behavior |
| F-terminal-01 | Terminal mechanism with two operands, followed by invocation of its produced target | Target production, runtime handling, fault behavior, and recoverability |

F-indirect-01 and F-terminal-01 are documentation identifiers, not independently identified implementations.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. No runtime tests have been performed.

### Capture stability

If the initial primary-argument-plus-100 word is 0x123456AB, both saved-byte updates use 0xAB. A later change to the original word does not change that captured byte. Subsequent overlapping writes may nevertheless change either destination.

### Signed halfword interpretation

If the signed locator’s stored bit pattern is 0xFFFF, its modeled value is −1. Under ordinary non-overflowing arithmetic, the branch-base read is therefore addressed at 19, not 65,555. Readability of address 19 is a separate prerequisite.

The later signed-factor capture has the same sign-extension requirement.

### Store before global observation

If branch base plus 84 designates the global source, the self-address store precedes both evaluations of that source. Under stable ordinary-memory semantics, both evaluations observe the newly stored branch base.

Omitting the store is unacceptable even when a non-aliasing execution would hide the difference.

### Independent global observations

If the execution environment permits the global source to yield different values on its two evaluations, the first establishes the global base and the second supplies the payload at initial link plus 68. Reusing the first value for both roles does not satisfy that observation pattern.

### Replacement-sensitive traversal

If previous link plus 20 aliases replacement base plus 20, the fresh read used for the working product occurs after the shifted branch base has been stored there. Using the previous contents would violate the required ordering.

### Fresh final-stage read

If the store at global base plus 100 overlaps expanded product plus 68, the later read from expanded product plus 68 uses memory after that store. Its result need not equal the earlier capture. The exact result depends on overlap and byte order.

### Repeated effects

Where individual memory events are observable, five signed-factor self-address stores are required. The first two must alternate with the two branch-base byte stores. Matching only the final memory image is insufficient.

### Stack and dependency mutation

Absent aliasing interference, stack offset 264 receives the retained word before the final external-memory updates, not merely during the later stack preparation.

If F-indirect-01 changes the final argument source and returns, and the post-call stack assignments do not alter it again, F-cdec09f5d432 receives the changed value.

### Complete continuation

If all required accesses complete and both ordinary dependencies return, execution reaches F-terminal-01. An ordinary successful return to the original caller is not an accepted outcome for that complete displayed continuation.

## Uncertainties

### Established static evidence

The supplied representation establishes modeled access widths, signed halfword conversions, capture boundaries, repeated mutations, stack assignment locations and order, call argument shapes, and the terminal continuation.

### Inferred intent

The repeated field offsets may suggest structured storage, and the exposed stack region may be intended as a dependency input record. Neither interpretation is a behavioral requirement. The fixed post-call word may have continuation-related significance, but that significance is unproven.

### documented_only

The following remain unresolved:

- Whether the apparent entry is a complete callable boundary.
- The provenance and calling-convention placement of ambient inputs.
- Actual object boundaries, lifetimes, alignment, and permitted aliasing.
- Byte order, pointer representation, overflow, shifting, and address wrapping.
- Whether memory is ordinary, concurrent, device-backed, read-sensitive, or faulting.
- Physical realization of modeled local values and aliasing effects on stack-based copies.
- Dependency preservation, ownership, retention, error, and return contracts.
- Runtime handling of access failures and the terminal mechanism.
- The significance of two reported exclusions of unreachable regions.

These unknowns must not be replaced with conventional ABI, pointer-validity, or memory-management assumptions.

## Evidence

This contract is based solely on the supplied static representation. The earlier draft and review feedback are not independent evidence of runtime behavior.

No dependency bodies, runtime traces, implementation access, or executed tests were available. The resulting contract describes the displayed path conditionally; it does not establish application intent, runtime feasibility, or a complete external calling convention.
