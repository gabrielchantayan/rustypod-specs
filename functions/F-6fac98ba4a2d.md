<!-- Trusted provenance: F-6fac98ba4a2d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6fac98ba4a2d

## Purpose

F-6fac98ba4a2d advances a counter-driven transition over two handle slots in a caller-supplied object. It applies entry gates, consumes a request flag, conditionally updates both handles, alternates the selected handle, and invokes a final dependency on a shortcut or completion path.

An alternating visual transition is **inferred intent**, not an established rendering contract. The supplied static evidence does not establish what the handles represent, whether dependency calls produce visible output, or whether completion releases resources.

This contract distinguishes represented behavior from unresolved implementation and dependency details. No runtime tests have been performed.

## Interface

The function accepts four arguments and has no represented return value.

| Argument | Represented width | Observed role |
|---|---:|---|
| Object address | 32 bits | Base of mutable state and handle slots. |
| Unused argument | 32 bits | Neither read nor forwarded. |
| Initial first coordinate | 32-bit signed integer | Initializes the first temporary coordinate value. |
| Initial second coordinate | 32-bit signed integer | Initializes the second temporary coordinate value. |

The supplied representation uses 32-bit signed and unsigned integers, 32-bit addresses, and 8-bit character storage. Exact calling conventions and external dependency parameter types are not established.

Object-relative layout:

| Offset | Width | Observed role |
|---|---:|---|
| `0x168` | 8 bits | Source for entry-state replacement. |
| `0x169` | 8 bits | Entry gate and replacement destination. |
| `0x1A0` | 32 bits | First handle slot. |
| `0x1A4` | 32 bits | Second handle slot. |
| `0x1A8` | 32-bit signed integer | First coordinate displacement. |
| `0x1AC` | 32-bit signed integer | Second coordinate displacement. |
| `0x1B0` | 32 bits | Handle selector; signed in alternation, unsigned in completion indexing. |
| `0x1B8` | 32 bits | Branch control, tested against zero. |
| `0x1DC` | 32 bits | Pause selection, compared with one. |
| `0x1E0` | 32 bits | Main counter; unsigned for range tests, signed for increment. |
| `0x1E4` | 32 bits | Subcounter; signed for increment and as an arithmetic-expression operand. |
| `0x1EC` | 8 bits | Request flag, consumed by clearing. |
| `0x1ED` | 8 bits | Pause flag. |
| `0x1F0` | 32 bits | Shortcut selection, compared with one. |

F-6ac70711bbb5 returns an address whose byte at offset `0x68` supplies the initial gate. Coordinate processing additionally reads signed 32-bit values at offsets `0x30` and `0x34` from the first handle.

Every reached access requires valid, appropriately accessible storage. No explicit null check protects the primary object or the initial dependency result. Handle nonzero checks do not establish handle validity.

The apparent selector domain is zero or one, but no bounds check enforces it. Safe execution also requires that reached signed arithmetic and address calculations remain defined.

## Behavior

### Entry gates

F-6ac70711bbb5 is called first, without explicit arguments. Its result is used to read the gate byte at offset `0x68`.

If that byte is zero, the function returns without an explicit primary-object access. This does not constrain the dependency’s own accesses or effects.

Otherwise, the object’s byte at `0x169` must be zero or two. The zero comparison occurs first; the comparison with two requires another read if the first comparison does not admit processing. Other values cause return.

After admission:

1. The byte at `0x168` is copied to `0x169`.
2. The pause flag at `0x1ED` is read.
3. A nonzero pause flag causes return.

The entry-state copy therefore occurs even when the pause flag stops processing.

The coordinate arguments then initialize two temporary signed 32-bit values. Every represented call that receives their storage occurs after both values have been replaced by computed coordinates. Their initial values have no demonstrated effect on persistent state or dependency arguments.

### Early shortcut

When branch control at `0x1B8` is zero, the first handle is read.

- A zero first handle selects the shortcut without reading shortcut selection.
- A nonzero first handle causes the field at `0x1F0` to be read; a value of one selects the shortcut.

The shortcut calls F-363f884d7ae3 with the object, then F-a037f41af662 with the object, and returns.

This path precedes request consumption, counter increments, and handle alternation. A nonzero branch-control field bypasses the shortcut.

### Request consumption

A local coordinate-suppression condition initially is false.

If the request byte at `0x1EC` is nonzero, the function, in order:

1. Reads the main counter as unsigned and records whether it is below 90.
2. Writes 90 to the main counter.
3. Clears the request byte.

A request with an original unsigned counter below 90 therefore suppresses coordinate processing later in this invocation. A request at 90 or above still replaces the counter with 90 but does not enable suppression.

### Threshold-dependent handle updates

Branch control is read again; its value need not equal the value used for the shortcut decision.

- With zero branch control, an unsigned main counter below 90 skips threshold processing.
- With zero branch control and a counter of at least 90, F-363f884d7ae3 receives the object. The main counter is then reread.
- With nonzero branch control, the function directly tests the unsigned main counter against 90.

When the applicable test finds at least 90, the first handle is read. The second handle is read only if the first is nonzero. Both must be nonzero to continue.

F-d7be9cfb4000 then receives an arithmetic value and the integer 20. Let \(M\) be the unsigned main-counter operand and \(S\) the signed subcounter operand read for this expression. Its represented 32-bit unsigned value is:

\[
\bigl((S \bmod 2^{32}) + 2M - 180\bigr)\times255
\pmod{2^{32}}.
\]

The unsigned main-counter multiplication makes the subsequent addition, subtraction, and final multiplication unsigned. These operations have modulo-\(2^{32}\) behavior in the represented type model. This is distinct from the signed counter increments described below.

The dependency result is converted to an 8-bit character value. F-e694578895f8 receives the first captured handle and negative one minus that character value. It is then called with a newly read second handle and the same derived value.

Character signedness and out-of-range conversion behavior are unresolved. For an unsigned character value, the derived integer ranges from −256 through −1. For a signed character in the range −128 through 127, it ranges from −128 through 127. The contract does not establish clamping or the eventual parameter conversion performed by F-e694578895f8.

The first handle is captured before F-d7be9cfb4000. The second handle used by the second update is reread after the first update. Dependency mutation can therefore change the second recipient.

### Subcounter and main-counter advancement

The subcounter is read as a signed 32-bit integer, and one is added **before** conversion to unsigned. The unsigned result is written to the subcounter.

If that result is below two, the function returns. Otherwise, the subcounter is written as zero, and the main counter undergoes the same signed-read, signed-addition, unsigned-conversion sequence. The resulting unsigned main counter is written back.

Consequences:

- Subcounter zero becomes one and returns.
- Subcounter one becomes two, is reset to zero, and permits main-counter advancement.
- Other defined increment results of at least two also cause reset and advancement.
- Subcounter `0xFFFFFFFF` is read as −1; adding one gives zero without overflow. Zero is written, and the function returns.
- An increment operand of `0x7FFFFFFF` causes signed overflow. The supplied representation establishes no defined result for that execution.

These increments are **not universally wrapping unsigned increments**. The same signed-overflow limitation applies to the main-counter increment when reached.

Threshold-dependent calls occur before the subcounter increment. They can occur on an invocation that subsequently returns without advancing the main counter.

### Alternation below 101

When the newly written unsigned main counter is below 101, the function selects a slot at object offset `0x1A0` plus four times the signed selector.

A zero selected handle skips alternation. A nonzero handle causes this ordered sequence:

1. F-df6da5bd408b receives the captured handle and one.
2. The selector and corresponding handle are reread; F-6af9d7af5ff8 receives that handle.
3. Conditional coordinate processing occurs.
4. The selector is reread; one minus its signed value is written back.
5. The handle indexed by that replacement value is passed to F-df6da5bd408b with zero.
6. The selector and corresponding handle are reread; F-9b8f3181789f receives that handle, one, and zero.

Only the handle admitting this sequence has an explicit nonzero check. Later handles are passed without additional checks.

Dependencies need not preserve the selector or slots. Consequently, successive calls need not address the same pair of handles. Selector subtraction, signed indexing, and address formation are not established as safe for arbitrary selector values.

### Conditional coordinate processing

After F-6af9d7af5ff8, coordinate work requires:

- The freshly read selector equals one.
- The first handle is nonzero.
- The second handle, read only after a nonzero first handle, is nonzero.
- Request-based coordinate suppression is false.

The first handle is captured as the coordinate source. The first temporary value is replaced with its signed value at offset `0x30` plus the object’s signed displacement at `0x1A8`. The second temporary value is then replaced using offsets `0x34` and `0x1AC`.

Both additions are signed 32-bit additions. Results outside the signed range are not established as wrapping.

F-a19ace379809 is called with a freshly read first handle and the address of the first temporary coordinate. It is then called with a freshly read second handle and the same address.

The representation suggests a two-coordinate temporary layout, but does not independently establish adjacency of the two separately represented local values or the dependency’s readable extent. Only the address of the first value is explicitly passed. No length is supplied.

The first dependency call may modify the referenced storage or object state. The second call therefore is not guaranteed to receive unchanged coordinate contents or the previously checked second handle.

### Pause and completion

After a nonzero-handle alternation, the main counter is reread and compared with exactly 90.

If it equals 90, the pause-selection field is read. A value of one causes the pause flag to be written as one. The function returns either way. This exact-90 return does not occur when the initial selected handle was zero.

Otherwise, on the below-101 path, the unsigned main counter is reread. A value below 100 causes return.

Completion is reached when that reread is at least 100, or directly when the newly incremented unsigned counter is at least 101.

At completion:

1. The selector is read as unsigned to select a handle slot.
2. If that handle is nonzero, the selector is read again, masked to its low eight bits, and passed with the captured handle to F-e694578895f8.
3. F-a037f41af662 receives the object unconditionally.

With stable state and defined arithmetic, advancement to 100 permits alternation before completion; advancement to 101 skips alternation. No explicit completed-state guard prevents later invocations from completing again.

## Side effects

Explicit persistent writes comprise:

- Entry-state replacement.
- Main-counter replacement with 90 and request clearing.
- Subcounter increment and possible reset.
- Main-counter increment.
- Selector replacement.
- Pause-flag setting on the eligible exact-90 path.

The function also writes temporary coordinate storage and invokes dependencies in the order described.

No explicit allocation, deallocation, or ownership transfer appears. Completion is not evidence of resource release.

The coordinate address refers to invocation-local storage. Retention beyond its lifetime has no established safety guarantee. Whether F-a19ace379809 copies, modifies, or retains that storage is `documented_only`.

No dependency result is checked as an error status. No explicit rollback, synchronization, exception handling, or recovery mechanism is represented. Ordinary continuation assumes dependencies return normally; their failure behavior is unknown.

## Dependencies

| Opaque ID | Observed use |
|---|---|
| F-6ac70711bbb5 | Supplies the address containing the initial gate byte. |
| F-363f884d7ae3 | Receives the object on the shortcut or eligible zero-control threshold path. |
| F-d7be9cfb4000 | Receives the unsigned arithmetic expression and 20; its result undergoes character conversion. |
| F-e694578895f8 | Receives a handle and character-derived value, or a completion handle and selector low byte. |
| F-df6da5bd408b | Receives a selected handle with one before alternation and zero afterward. |
| F-6af9d7af5ff8 | Receives a reread selected handle before coordinate processing. |
| F-a19ace379809 | Receives a reread handle and temporary-coordinate address, twice when eligible. |
| F-9b8f3181789f | Receives a reread selected handle, one, and zero after selector replacement. |
| F-a037f41af662 | Receives the object on shortcut or normal completion. |

These call-site contracts do not imply preservation of globals, object fields, handles, or referenced temporary storage.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Dependencies return normally and preserve relevant state unless an example explicitly states otherwise. Reached memory accesses and arithmetic must be valid and defined.

1. **Disabled initial gate.** A zero gate byte causes return after F-6ac70711bbb5, without explicit primary-object access.

2. **Rejected entry state.** With the initial gate enabled and entry-state byte three, the function returns without copying the source byte.

3. **Paused after admission.** With the initial gate enabled, entry-state byte zero or two, and a nonzero pause flag, the source byte is copied before return. This body does not change counters.

4. **Shortcut.** With entry admitted, pause clear, branch control zero, and first handle zero, F-363f884d7ae3 precedes F-a037f41af662. A request remains unconsumed. The same call sequence occurs with a nonzero first handle and shortcut selection one.

**Shared baseline for examples 5–11:** the initial gate is nonzero; the entry-state byte initially is zero or two; the pause flag is zero; branch control is nonzero, avoiding the shortcut. The selector is zero or one, handles and reached memory are valid, and no request is pending unless stated. Dependencies preserve relevant state.

5. **One substep.** Main counter ten and subcounter zero leave the main counter and selector unchanged. The subcounter becomes one, then the function returns.

6. **Pause boundary.** Main counter 89, subcounter one, both handles nonzero, and pause selection one produce main counter 90, alternation, pause flag one, and return without F-a037f41af662.

7. **Request suppression.** A request with main counter ten, subcounter one, selector one, and both handles nonzero sets the main counter to 90, clears the request, then advances the counter to 91. Coordinate calls are suppressed; threshold-dependent handle updates remain eligible.

8. **Completion after alternation.** Main counter 99, subcounter one, selector zero, and both handles nonzero produce main counter 100 and selector one. Completion passes the second handle and one to F-e694578895f8, then calls F-a037f41af662.

9. **Defined all-ones subcounter increment.** Subcounter `0xFFFFFFFF` increments from signed −1 to zero. Zero is written, and the function returns without a main-counter increment. Earlier threshold calls remain possible.

10. **Defined unsigned threshold arithmetic.** With main counter `0x80000000`, subcounter zero, and both handles nonzero, the first arithmetic argument to F-d7be9cfb4000 is `0xFFFF4CB4` as an unsigned expression. The subcounter subsequently becomes one and causes return, so the main counter is not incremented.

11. **Coordinate update.** Main counter ten, subcounter one, selector one, and both handles nonzero reach coordinate processing. Source coordinates 20 and −5 with displacements 3 and 7 produce temporary values 23 and 2 before the two F-a19ace379809 calls.

12. **Signed-overflow boundary.** Under the shared entry baseline, a subcounter of `0x7FFFFFFF` reaches signed overflow during increment. No post-increment state or return behavior is guaranteed. The same limitation applies to a main counter of `0x7FFFFFFF` when its increment is reached.

## Uncertainties

- **`documented_only`: character conversion.** Character signedness, out-of-range conversion, and dependency argument conversions are unresolved.
- **`documented_only`: signed overflow and addressing.** Signed counter increments, coordinate additions, selector subtraction, and signed address calculations do not acquire unsigned wrapping semantics merely because some destinations are unsigned.
- **`documented_only`: temporary layout.** A two-coordinate arrangement is suggested, but adjacency and dependency access extent are not independently established.
- **`documented_only`: dependency semantics.** Rendering, ownership, retention, failure handling, and preservation guarantees are unknown.
- **`documented_only`: concurrency and reentrancy.** Repeated reads can produce different values; no synchronization or coherent-snapshot guarantee is represented.
- **Inferred intent:** two-handle alternation and paired updates suggest a transition effect, not a guaranteed domain behavior.

## Evidence

The supplied static representation establishes the access widths, object offsets, branch structure, constants, explicit write sequencing, dependency-call order, and signed versus unsigned arithmetic distinctions described here.

Reads within an individual arithmetic expression are not assigned a stricter evaluation order than the representation establishes. Explicit sequencing between statements and dependency calls is distinguished from such operand-order uncertainty.

The threshold expression has represented modulo-\(2^{32}\) arithmetic. Counter increments instead perform signed addition before unsigned conversion. Dependency internals, exact ABI details, temporary-storage layout, and runtime outcomes remain unestablished.

No runtime tests, execution traces, or independent implementation checks have been performed.
