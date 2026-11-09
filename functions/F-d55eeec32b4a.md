<!-- Trusted provenance: F-d55eeec32b4a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d55eeec32b4a

## Purpose

F-d55eeec32b4a describes an ordered sequence of narrow memory writes, context-word updates, global zero assignments, and external invocations.

The supplied static representation establishes only a visible execution prefix. It reports invalid instruction data and ends at a truncated-control-flow boundary. It does not establish a normal return, an intentional processor halt, or the behavior of any continuation.

No application-level purpose is established. Repeated writes and a reset-associated global name do not prove initialization, reset handling, device programming, or decoding.

This contract distinguishes:

- **Established static evidence:** behavior explicitly represented, including access widths, call targets, argument construction, and ordering.
- **Inferred intent:** possible interpretations that are not acceptance requirements.
- **documented_only unknowns:** behavior for which the supplied material is insufficient.

“Established” here means supported by the supplied representation, not independently validated against machine execution.

## Interface

### Explicit inputs

The represented interface has two 32-bit input slots.

| Input | Visible use |
|---|---|
| First input | No explicit use in the visible prefix. |
| Second input | Supplied as the second argument of the initial external invocation. |

The unused first slot is not proven irrelevant to actual execution. The recovered interface and calling convention have not been independently verified.

### Implicit entry state

Four additional incoming values influence the visible behavior:

| Semantic role | Width | Use |
|---|---:|---|
| Entry address | 32 bits | Initial store destination, source of stored bits, arithmetic operand, and call-argument source |
| Context base | 32 bits | Locates a context word at byte offset `0x14` |
| Context value | 32 bits | Written unchanged and later shifted left by eight bits |
| Link value | 32 bits | Determines the initial call target and contributes to its third argument |

These names describe roles in this contract; they are not additional declared parameters.

The prefix also assigns zero to a reset-associated global location. Its address, storage width, and overlap with other symbols are unresolved. The supplied representation warns that some global symbols overlap smaller symbols.

### Outputs and results

No explicit output value is established. The represented interface has no return value, and the visible prefix does not reach a verified return.

Six external invocations produce values represented as 64 bits. For each:

- The **lower word** is bits 0–31.
- The **upper word** is bits 32–63.

The caller’s use of those bits is established. Their semantic type and underlying return-register convention are not.

Other external invocations have no explicitly consumed result. This does not establish that they produce no result.

### Widths and arithmetic

The represented memory operations comprise:

- 16-bit stores through computed addresses.
- 32-bit stores at the context base plus `0x14`.
- Zero assignments to the reset-associated global, whose access width is unknown.

Address arithmetic and word arithmetic are described as 32-bit bit-vector operations: overflow retains the low 32 bits. This convention captures the represented machine-width calculations; it is not a claim about portable signed-language arithmetic.

Right shifts used for result extraction are logical. Halfword values retain only 16 bits. Byte order is unknown, so this contract specifies stored values rather than their byte-level layout.

### Preconditions for completing the prefix

Completion requires every reached memory access and call target to be supported by the execution environment. Every preceding dependency must return control and leave state compatible with subsequent operations.

These are conditional requirements, not guarantees supplied by the dependencies. In particular, the contract assumes neither disjoint memory regions nor dependency preservation of globals, resources, or memory.

## Behavior

### Initial writes and invocation

Before any external invocation, three halfword stores occur in order:

- The low 16 bits of the entry address are stored at the entry address.
- The identical value is stored again at the identical address.
- The same halfword is stored at twice the numeric entry address, using 32-bit wrapping arithmetic.

The third destination is not the entry address plus two bytes.

A 32-bit store then writes the unshifted context value at the context base plus `0x14`.

The initial invocation, locally identified as F-000000000001, follows those writes. Its represented target is the link value plus `0xFFE`, reduced to 32 bits. Its three explicit arguments, in order, are:

1. The entry address shifted left by five bits, reduced to 32 bits.
2. The second explicit input.
3. The sum of the entry address and link value, reduced to 32 bits.

No result from this invocation is explicitly consumed.

### Result-derived destinations and values

Each of the next six result-producing invocations is followed by a pair of identical halfword stores and a zero assignment to the reset-associated global.

Two transformations occur.

**Format A**

For lower word \(L\), the destination is:

\[
D_A = (L \mathbin{\&} \texttt{0x07FFFFFF}) \gg 3
\]

The stored halfword is bits 3–18 of \(L\):

\[
V_A = (L \gg 3) \mathbin{\&} \texttt{0xFFFF}
\]

The represented calculation uses a 32-bit left shift followed by a logical right shift and 16-bit truncation. The expression above gives the same final halfword.

**Format B**

For lower word \(L\), the destination is:

\[
D_B = L \gg 8
\]

The stored halfword is bits 8–23 of the 64-bit result:

\[
V_B = (L \gg 8) \mathbin{\&} \texttt{0xFFFF}
\]

Those bits lie entirely within the lower word. The upper word therefore affects neither the destination nor the stored halfword.

For either format:

- The destination lies between `0x00000000` and `0x00FFFFFF`, inclusive.
- The stored halfword equals the low 16 bits of the destination.
- The destination need not be aligned to two bytes.
- A zero destination is possible and is not rejected by any visible validation.

The formats remain distinct because they select different result bits.

### Six-stage ordering

The following table gives the ordered stages. Dependency IDs are local call-site handles whose exact target bindings appear under **Dependencies**.

| Stage | Result-producing invocation | Format | Context write after global zero assignment | Following transition |
|---|---|---|---|---|
| 1 | F-000000000002 | A | Unshifted context value | F-000000000003 |
| 2 | F-000000000004 | A | Unshifted context value | F-000000000005 |
| 3 | F-000000000006 | B | Context value shifted left eight bits | F-000000000007 |
| 4 | F-000000000008 | A | Same shifted value | F-000000000009 |
| 5 | F-000000000010 | A | Same shifted value | F-000000000011 |
| 6 | F-000000000012 | B | None shown | No transition of this form |

Within each stage, the result-producing invocation returns before its result is used. The first halfword store precedes the identical second store. The global zero assignment follows both stores.

In stages 1–5, the listed context write follows the global assignment. The transition invocation follows the context write and precedes the next stage.

The shifted context value is formed after the stage-3 result is obtained and before that stage’s stores. It retains the low 32 bits of the context value shifted left eight bits. The same represented value supplies the context stores in stages 3, 4, and 5.

Each transition has three explicit arguments:

1. Zero.
2. The upper word of the immediately preceding stage result.
3. The corresponding constant below.

| Transition | Third-argument bit pattern |
|---|---|
| F-000000000003 | `0x089101B5` |
| F-000000000005 | `0x0891021D` |
| F-000000000007 | `0x08910285` |
| F-000000000009 | `0x089102ED` |
| F-000000000011 | `0x08910355` |

These constants are arguments, not the transition call targets. No transition result is explicitly consumed.

The six result-producing invocations have no explicit arguments in the representation. This does not establish an absence of implicit inputs.

### Final invocations and boundary

After stage 6’s second halfword store and global zero assignment, F-000000000013 is invoked with four explicit arguments:

1. `0xFFFFFF1C`, the 32-bit bit pattern for −228.
2. The stage-6 upper word minus `0x264`, reduced to 32 bits.
3. `0x0000004B`, or 75.
4. Zero.

No context write is shown between stage 6’s global assignment and this invocation.

Four further invocations follow, in order:

- F-000000000014.
- F-000000000015.
- F-000000000016.
- F-000000000017.

No explicit arguments or consumed results are shown for these four calls.

The representation then terminates at the invalid-data boundary. Neither a return nor any particular runtime effect at that boundary is established.

## Side effects

### Caller-visible writes

Conditional on reaching every operation, the visible prefix contains:

| Operation | Count |
|---|---:|
| 16-bit stores | 15 |
| 32-bit context stores | 6 |
| Global zero assignments of unresolved width | 6 |
| External invocations | 17 |

The halfword stores comprise three initial stores and twelve result-derived stores. The context stores comprise three writes of the unshifted value and three writes of the shifted value.

These are operation counts, not distinct-location counts. Initial destinations, result-derived destinations, the context word, and the global may overlap. Later writes may overwrite earlier writes partially or completely.

Duplicate stores remain separate represented operations. Their distinction may matter for faults, device memory, or observation mechanisms. The representation does not independently establish physical bus transactions, memory-ordering barriers, or volatility rules.

### Reads and ordering

No explicit read of an old destination value precedes a caller-side store. In particular, the duplicate halfword stores are not represented as read-modify-write operations.

The caller consumes entry state and returned result bits. Dependencies may perform additional reads or writes. Their internal ordering and effects are unknown.

The ordering specified here is the visible caller sequence. It does not imply transactional execution or isolation from other agents.

### Ownership and lifetime

No explicit allocation, release, reference-count adjustment, or ownership transfer appears in the caller sequence.

That absence does not constrain dependency behavior. Dependencies are not known to preserve globals, retain no addresses, release no resources, or leave previously valid destinations usable.

### Errors and partial progress

No explicit validation, status check, rollback, cleanup branch, or retry appears.

A fault or nonreturning dependency may prevent later operations. Earlier completed writes are not visibly undone. Potential failure points include unsupported destinations, alignment restrictions, invalid executable targets, and dependency-internal failures.

The execution environment determines whether a particular access or target fails. The supplied representation establishes no error code, exception type, or recovery behavior.

## Dependencies

The private dependency map establishes F-d55eeec32b4a as the subject function’s opaque identity. It does not establish opaque identities for the callees.

The IDs below are therefore **local reference handles for call sites**, retained to express ordering. They are not provenance-backed dependency identities. Their represented target bindings are nevertheless known and are part of this contract.

| Local handle | Represented target | Caller-side role |
|---|---|---|
| F-000000000001 | Link value plus `0xFFE`, modulo \(2^{32}\) | Initial three-argument invocation |
| F-000000000002 | `0x089111B1` | Stage-1 result |
| F-000000000003 | `0x089111B3` | Transition after stage 1 |
| F-000000000004 | `0x08911219` | Stage-2 result |
| F-000000000005 | `0x0891121B` | Transition after stage 2 |
| F-000000000006 | `0x08911281` | Stage-3 result |
| F-000000000007 | `0x08911283` | Transition after stage 3 |
| F-000000000008 | `0x089112E9` | Stage-4 result |
| F-000000000009 | `0x089112EB` | Transition after stage 4 |
| F-000000000010 | `0x08911351` | Stage-5 result |
| F-000000000011 | `0x08911353` | Transition after stage 5 |
| F-000000000012 | `0x089113B9` | Stage-6 result |
| F-000000000013 | `0x08910BB3` | Final four-argument invocation |
| F-000000000014 | `0x08910CD9` | First subsequent invocation |
| F-000000000015 | `0x0850C4DE` | Second subsequent invocation |
| F-000000000016 | `0x08112CF0` | Third subsequent invocation |
| F-000000000017 | `0x08D2551A` | Last invocation before the boundary |

These are the exact numeric targets represented, including their low bits. No target normalization is assumed. Instruction-state selection, address interpretation, and executable validity are **documented_only**.

Unknown dependency behavior does not make target selection unknown. Conversely, a known target does not establish a dependency’s interface, preservation obligations, ownership behavior, failure modes, or termination.

The dynamic initial target may coincide with another target for some link values. Distinct call-site handles do not prove distinct underlying implementations.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes execution reaches the relevant operation and the environment permits its accesses.

### Initial writes and address arithmetic

For entry address `0x00001234`:

- Two consecutive halfword stores write `0x1234` at `0x00001234`.
- The third writes `0x1234` at `0x00002468`.
- The initial context store follows all three.

A third destination of `0x00001236` would not satisfy the represented arithmetic.

For entry address `0x80000002`, the doubled destination wraps to `0x00000004`, while the stored halfword is `0x0002`. The first call’s entry-derived first argument is `0x00000040`.

### Fixed targets versus argument constants

After the initial invocation returns, the stage-1 result is requested from target `0x089111B1`.

Following stage 1’s writes, global assignment, and context update, the transition targets `0x089111B3`. Its third argument is `0x089101B5`.

Using that argument constant as the transition target would violate the target binding.

### Format A

For lower word `0xF8123458`:

- Applying the destination mask yields `0x00123458`.
- The destination is `0x0002468B`.
- The stored halfword is `0x468B`.
- Both stores precede the global zero assignment.

This destination is odd. Successful completion depends on support for the resulting halfword access.

### Format B

For lower word `0x12345678`:

- The destination is `0x00123456`.
- The stored halfword is `0x3456`.

Changing only the upper word changes neither derived value. It can change the upper-word argument supplied to the following transition or, for stage 6, the argument supplied to F-000000000013.

### Context truncation

For context value `0xAB123456`:

- The initial context store and those following stages 1 and 2 write `0xAB123456`.
- The stores following stages 3, 4, and 5 write `0x12345600`.

No stage-6 context store is shown. This is not a guarantee that dependencies leave the context word unchanged.

### Final subtraction and target order

For stage-6 upper word `0x00000100`, the second argument to F-000000000013 is `0xFFFFFE9C`.

That invocation targets `0x08910BB3`. If all subsequent calls return, the remaining targets occur in this order: `0x08910CD9`, `0x0850C4DE`, `0x08112CF0`, and `0x08D2551A`.

No underflow check or saturation is represented.

### Interrupted progress

If a stage’s first halfword store completes and its second store faults, the first write belongs to the completed prefix. That stage’s later global assignment and context write have not yet occurred.

If a transition does not return, the following result-producing invocation is not reached.

## Uncertainties

The following remain **documented_only**:

- Whether the represented entry point and interface describe a genuine callable routine.
- Whether invalid instruction data affected earlier recovered behavior as well as the terminal boundary.
- Actual instruction set, calling convention, byte order, alignment rules, and memory map.
- The global location’s address, width, overlap, and hardware significance.
- Dependency internals, implicit inputs, memory effects, resource ownership, and preservation obligations.
- Whether represented values remain available across calls under the actual machine convention.
- Whether destinations are ordinary memory, device memory, or another kind of addressable state.
- Whether the terminal boundary corresponds to an exception, an intentional stop, executable continuation, or misinterpreted data.
- Architectural interpretation of the exact call-target bit patterns.

A packed-address or device-command interpretation is plausible because returned bits determine both destination and stored value. It remains inferred intent, not an acceptance requirement.

## Evidence

This contract uses only the supplied static representation. That material includes warnings about invalid instruction data and overlapping global symbols.

The represented operations support the access widths, ordering, argument bit patterns, six result transformations, and fixed target bindings recorded above. Arithmetic simplifications and example values follow from the represented widths.

The local call-site handles do not add dependency identities to the supplied map. Fixed target addresses are preserved independently of unresolved callee behavior.

No dependency bodies, independently verified interface declarations, memory-map definitions, or instruction-level validation were supplied. No runtime tests have been performed.
