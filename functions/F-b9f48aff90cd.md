<!-- Trusted provenance: F-b9f48aff90cd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b9f48aff90cd

## Purpose

F-b9f48aff90cd prepares byte data, selects a chaining-related operand, invokes repeated transformations followed by a final transformation, optionally converts the resulting bytes, and emits them through argument 2.

**Established static behavior:** processing uses four separately declared 16-byte automatic arrays, caller-provided selector words and resources, and global words controlling arithmetic, indexing, and continuation predicates.

**Inferred intent:** the structure resembles a configurable, table-driven block transformation with chaining. The supplied evidence does not establish a named algorithm, encryption direction, standard chaining mode, or padding convention.

The outer displacement advances by 16 after a completed iteration. This does not establish that each iteration processes exactly 16 bytes: starting counters, predicates, and effective addresses depend on unspecified global values.

## Interface

### Arguments

The interface has 25 positions. Scalars and opaque values are 32 bits. Selector arguments point to 32-bit words. Several integer arguments are used as addresses; the evidenced address calculations are 32-bit, not a portable wider-pointer interface.

| Position | Observed role |
|---|---|
| 1 | Source byte-region base. |
| 2 | Destination byte-region base; also read on one chaining path. |
| 3 | Initial-chaining byte-region base. |
| 4 | Scalar used to prepare repeated-call bounds and a retained final-resource index component. |
| 5 | Scalar used in outer-loop setup. Units and supported range are unresolved. |
| 6 | Input-preparation selector pointer. |
| 7 | Input lookup-layout selector pointer. |
| 8 | Selector pointer used during destination-based preparation and again after the final transformation. |
| 9 | Destination-chaining lookup-layout selector pointer. |
| 10 | Initial-chaining preparation selector pointer. |
| 11 | Initial-chaining lookup-layout selector pointer. |
| 12 | Opaque value passed to F-d0d1ab73f88c. |
| 13 | Position-dependent input lookup base. |
| 14 | Input-preparation descriptor passed to F-d2332752ccd0. |
| 15 | Shared input lookup base. |
| 16 | Position-dependent initial-chaining lookup base. |
| 17 | Initial-chaining descriptor passed to F-d2332752ccd0. |
| 18 | Shared initial-chaining lookup base. |
| 19 | Position-dependent destination-chaining lookup base. |
| 20 | Shared destination-chaining lookup base. |
| 21 | Resource base passed to F-d0d1ab73f88c and used for final-resource addressing. |
| 22–23 | Opaque values passed to F-d0d1ab73f88c. |
| 24 | Descriptor for optional in-place F-d2332752ccd0 conversion. |
| 25 | Final position-dependent byte lookup base. |

“Selector,” “descriptor,” and “resource” identify observed uses, not independently established formats.

### Result and prerequisites

The visible ordinary exit returns the 32-bit integer zero. There is no visible processed-byte count, separate error return, or inspection of dependency return values.

Normal return requires valid execution of every selected access and dependency call. In particular:

- Selector storage must support the actual 32-bit reads.
- Byte regions and tables must cover computed addresses.
- Automatic-array indices must remain valid, and bytes read must have valid prior initialization.
- Dependencies must accept the supplied descriptors, addresses, and overlap relationships.
- Predicates must eventually terminate if return is required.

No null, capacity, alignment, overlap, or initialization checks are visible. No fixed minimum buffer size can be derived without the global values and dependency contracts.

## Behavior

### Widths and arithmetic

The body uses 32-bit words, signed sign tests, bitwise operations, right shifts, unsigned byte loads, and byte-width stores. Unsigned arithmetic wraps at 32 bits. Unbounded-integer simplification is not justified for these expressions.

A recurring lookup input is the low eight bits of the source byte plus 134. For byte \(b\), this is \((b+134)\bmod256\). Further arithmetic constructs the lookup address using multiplication, complement, OR, XOR, and global values. The complete column calculation is not established to equal direct indexing by the original byte.

Position-dependent tables use a 256-byte row stride. Final-resource selection uses a 4096-byte stride. Neither stride establishes the number of available rows.

Some intermediates have character width before further arithmetic. Byte truncation is observable; signed character promotion and signed-overflow behavior must not be generalized beyond the represented target.

### Global state and retained values

The following descriptive labels identify global roles, not additional interface arguments:

- **Origin word:** supplies several starting counters and contributes to entry setup.
- **Offset word:** participates in addresses, predicates, repeated-loop capture, and final-resource selection.
- **Repeated-loop seed word:** contributes to repeated-call setup and the retained final-resource index component.
- **Per-call counter-offset word:** contributes to the fourth argument of each executed repeated call.

These labels do not imply that all reads of a word observe one snapshot.

Before outer processing, the function retains an origin-derived complement, origin-derived bound quantities involving argument 5, and a starting outer displacement. The retained complement survives dependency calls and subsequent outer iterations. It is also used in lookup arithmetic and the later postprocessing predicate.

Other reads remain live at their visible evaluation points. In particular, three distinct state relationships are established:

1. **Repeated-loop capture:** every repeated-loop condition evaluation captures the then-current offset word, including the terminating evaluation. The last captured value survives the final transformation call and participates in postprocessing selection.
2. **Final-resource split:** an argument-4-derived index component is calculated before the repeated calls and retained across them; a separate offset-word read occurs when forming the final transformation’s resource argument.
3. **Emission-counter selection:** an origin-word read after the final transformation supplies the destination-emission starting counter if postprocessing is skipped. If postprocessing runs, the terminating condition evaluation of its final lookup loop supplies that counter instead.

Dependency mutation therefore cannot be modeled by either a single entry snapshot or indiscriminate rereading after every call.

### Outer processing

Each outer condition uses the current displacement, retained entry-derived bounds, and live global values. A terminating condition returns zero without performing that iteration’s preparation, transformation, or destination emission.

Before an entered iteration’s input preparation, the function forms and retains an outer-displacement-plus-offset quantity. This value later contributes to chaining-path selection, alongside another offset-word access. Input-preparation dependencies can therefore intervene between the retained contribution and the later live contribution.

After a completed destination-emission loop, the outer displacement advances by 16 in the represented arithmetic. There is no visible tail-padding or partial-block-specific procedure.

### Input preparation

The function evaluates the selector through argument 6 using a signed arithmetic predicate.

On the lookup path, it evaluates argument 7 and selects:

- Argument 15 for shared lookup storage; or
- Argument 13 for position-dependent storage.

Each executed byte step reads the source, computes the transformed lookup input, reads the selected table byte, and writes the input-preparation array. The source address includes the source base, outer displacement, byte counter, and twice the live offset contribution. The prepared-array position includes the counter and an offset contribution. Position-dependent lookup adds 256 times its computed array position.

On the other path, F-d2332752ccd0 receives:

1. A source address derived from argument 1, the current outer displacement, and a live offset contribution.
2. Argument 14.
3. The input-preparation array address.

Argument 7 is not consulted by this dependency path.

Selector expressions contain repeated dereferences and arithmetic, not an established ordinary Boolean test. The representation does not establish one atomic selector snapshot or a left-to-right order among operands within an expression.

### Chaining preparation

A displacement-related arithmetic predicate chooses between initial-region and destination-based preparation.

#### Initial-region path

The function evaluates argument 10. Its lookup outcome additionally evaluates argument 11, choosing shared storage at argument 18 or position-dependent storage at argument 16.

The lookup loop reads bytes relative to argument 3, applies lookup-address arithmetic, and fills the initial-chaining preparation array. Alternatively, F-d2332752ccd0 receives argument 3, argument 17, and that array’s address.

A subsequent loop reads the prepared input and initial-chaining bytes and writes the working array using byte arithmetic and global-dependent bitwise operations.

**Inferred intent:** this combination may implement an XOR-like operation under intended global assignments. Exact XOR behavior is not established without those assignments.

#### Destination-based path

For each executed byte step, the function evaluates argument 8.

On its lookup outcome, it first reads a byte relative to argument 2 using the outer displacement, byte counter, and global offsets. It then evaluates argument 9 to choose shared storage at argument 20 or position-dependent storage at argument 19. The selected lookup result is written to the working array after byte arithmetic whose added constants cancel at byte width.

The position-dependent row uses a counter-derived quantity containing an additional displacement of 16. It must not be replaced with the working-array index without accounting for that difference.

On the other outcome, the byte comes from an address relative to a separate automatic scalar. The working-array value is that byte plus 122, truncated to eight bits. The evidence does not establish this address as a populated byte-buffer region.

The path then reads the initial-chaining preparation array and working array and writes the auxiliary array. It does not visibly initialize the initial-chaining preparation array for the current iteration. The input-preparation array is not visibly consumed by this path.

Consequently, conventional previous-block chaining, valid initialization, and valid fallback scalar-relative accesses are not unconditional properties.

### Repeated transformations and final-resource selection

Before the repeated calls, the function derives bound quantities from argument 4 and the repeated-loop seed word. It also calculates and retains a separate final-resource index component.

The index component uses the seed word’s logical right shift by 30 and subtraction from argument 4. This differs from the signed right shift by 31 used in another part of repeated-loop setup. The two calculations must not be conflated.

Each repeated-loop condition evaluation captures the current offset word before deciding whether another call occurs. A terminating evaluation still updates that retained capture.

Every executed iteration invokes F-d0d1ab73f88c with:

1. Working-array address.
2. Argument 22.
3. Argument 23.
4. Current repeated-loop counter plus a freshly read per-call counter-offset word.
5. Argument 12.
6. Argument 21.

The counter advances by one after a returning call. The per-call counter offset is read separately for each executed call; it is not retained from loop setup. The exact call count cannot be derived from argument 4 alone.

After termination, F-eda87ef38dc8 is called once, provided execution reaches it. Its arguments are the working-array address and the final-resource address.

The resource address combines argument 21 with 4096 times the sum of:

- The index component retained before the repeated calls; and
- An offset-word value read while constructing this final argument.

A repeated dependency’s changes to globals do not retroactively recompute the retained component. They can affect later live reads. F-eda87ef38dc8 can subsequently change globals, but the offset captured by the terminating repeated-loop condition remains the postprocessing predicate’s retained input.

### Postprocessing decision

After F-eda87ef38dc8 returns, the function derives an adjustment from the retained offset capture and reads the origin word for a potential destination-emission starting counter. This origin read precedes the postprocessing decision.

The decision then accesses argument 8 again. It combines:

- Argument-8 selector accesses at this later evaluation point;
- The offset captured during the terminating repeated-loop condition;
- The origin-derived complement retained at entry.

For a stable selector value during this expression, let \(s\) denote that 32-bit word, \(h\) the retained offset capture, and \(c\) the retained entry complement. Let \(d\) be the arithmetic right shift of \(h\) by 30, minus \(h\), using the represented 32-bit operations. Postprocessing is selected when the signed interpretation of

\[
((s-h)+c)\ \mathbin{\mathrm{OR}}\ (d-(s-h))
\]

is nonnegative.

This expression is a description of the decision, not a guarantee that repeated selector dereferences are atomic. In particular, the decision does **not** substitute an offset word freshly read after F-eda87ef38dc8.

### Selected postprocessing and destination emission

When selected, postprocessing performs these phases in order:

1. A loop writes working-byte-derived values to the auxiliary array.
2. F-d2332752ccd0 receives the working-array address, argument 24, and the same working-array address as destination.
3. A final lookup loop replaces working bytes using argument 25.

The preliminary auxiliary writes are not explicitly passed to the conversion dependency and are not an additional caller-visible output.

The final lookup loop starts from an origin-word read after the in-place dependency. During **every** condition evaluation, it reloads the origin word, including when the condition terminates the loop. A continuing iteration still advances from its previously computed counter-plus-one value; the origin reload is not an unconditional restart of lookup traversal.

On termination, that evaluation’s reloaded origin value remains as the starting counter for destination emission. This replaces the earlier origin value obtained after F-eda87ef38dc8.

The final lookup’s row calculation includes a low-three-bit-related mask and a 256-byte stride. Exactly eight rows and simple position-modulo-eight selection are not established.

If postprocessing is skipped, emission instead starts from the origin value read after F-eda87ef38dc8 and before the decision.

Each executed emission step reads a working byte and stores it through argument 2. The working index includes the emission counter and an offset contribution; the destination address includes argument 2, the outer displacement, that counter, and twice an offset contribution. Emission then advances its counter by one.

## Side effects

The body directly writes destination bytes and reads source, selector, chaining, and lookup storage. Destination-based preparation also reads argument 2.

Aliasing is unchecked. Destination stores can change later source bytes, selectors, or tables if caller-provided regions overlap. There is no protective copy of the complete input.

Dependencies receive automatic-array addresses and caller-provided resources. Their mutation scope, global effects, allocation behavior, ownership rules, pointer retention, and failure behavior are unknown. They cannot be assumed to preserve globals or caller storage.

The visible body performs no allocation, deallocation, explicit ownership transfer, or deliberate clearing. Automatic-array pointers are valid only within their enclosing activation. Retention by a dependency would require an independently established lifetime arrangement.

There is no visible rollback or recovery. Earlier destination writes remain observable if later execution fails. Failure during emission can leave a partially written current iteration.

## Dependencies

| Opaque function ID | Established use | Unresolved requirements |
|---|---|---|
| F-d2332752ccd0 | Source, descriptor, destination; used for input preparation, initial-chain preparation, and optional in-place working conversion. | Extent, descriptor format, initialization guarantees, overlap support, ownership, mutation, and failures. |
| F-d0d1ab73f88c | Repeated six-argument transformation with the working array, opaque values, counter-derived argument, and resource base. | Transformation meaning, resource layouts, memory extent, global effects, and failures. |
| F-eda87ef38dc8 | Working array plus a computed resource address with 4096-byte stride. | Transformation meaning, accessible extent, mutation, ownership, and failures. |

No dependency return value is visibly used to select recovery or report an error.

## Acceptance examples

These are conditional static-contract examples, not runtime results.

1. **Initial outer termination.** If the first outer predicate terminates, no iteration preparation, transformation call, or destination emission occurs. Ordinary return is zero.

2. **Input dependency selection.** If argument 6 selects dependency preparation, F-d2332752ccd0 receives the derived source address, argument 14, and the input-preparation array. Argument 7 is not needed for that path.

3. **Immediate repeated-loop termination.** If its first condition terminates, no F-d0d1ab73f88c call occurs. That condition still captures the offset word. F-eda87ef38dc8 remains reachable afterward.

4. **Changing per-call offset.** If two executed repeated calls use counters 5 and 6, and their respective counter-offset reads yield 7 and 9, their fourth arguments are 12 and 15.

5. **Retained resource component.** If the pre-call retained index component is 3 and the final-resource offset read yields 2, the final resource is argument 21 plus 20,480 in the represented address arithmetic. Seed-word changes during repeated calls do not recompute the retained 3.

6. **Final dependency changes the offset word.** If the terminating repeated-loop condition captured one offset value and F-eda87ef38dc8 changes the global afterward, postprocessing still uses the captured value. A result based on the newly written global is not equivalent.

7. **Postprocessing skipped.** Emission starts from the origin read after F-eda87ef38dc8 and before the decision, not from the retained entry origin.

8. **Postprocessing selected.** The in-place conversion precedes final lookup. Emission starts from the origin value reloaded by the final lookup’s terminating condition, even if that lookup executes no body iterations.

9. **Conditional 16-byte emission.** If independently supplied globals make emission visit exactly valid indices 0 through 15 and preceding operations initialize those bytes, those 16 bytes are emitted in loop order. Without these premises, exact 16-byte output is unsupported.

## Uncertainties

The following remain **documented_only**:

- Concrete global values, supported update rules, and concurrency assumptions.
- Exact iteration counts, valid starting offsets, and arguments 4 and 5’s supported ranges.
- Boolean interpretations of selector predicates.
- Simplified lookup-column mappings and minimum table sizes.
- Valid initialization and scalar-relative access on the destination-based path.
- Dependency semantics, overlap support, ownership, failures, and retained-pointer behavior.
- Supported executions’ avoidance of signed-overflow ambiguity and invalid automatic-array accesses.
- Named algorithm, security properties, standard chaining mode, and padding semantics.

These unknowns do not erase the established timing distinctions between retained values, per-call reads, and terminating-condition captures.

## Evidence

This contract uses only the supplied static representation: interface widths, four 16-byte arrays, visible accesses, dependency argument order, arithmetic structure, retained values, selector rereads, loop-condition captures, and ordinary zero return.

The resource-selection snapshot, per-call counter-offset reads, retained postprocessing offset, and emission-counter handoff follow from visible data dependencies across the calls and loop exits.

Algorithmic intent is identified separately from established behavior. No dependency implementations or concrete global assignments were supplied. No runtime tests have been performed.
