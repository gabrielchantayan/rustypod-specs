<!-- Trusted provenance: F-6bd9dee8da0b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6bd9dee8da0b

## Purpose

F-6bd9dee8da0b selects between an immediate return, a direct operation followed by a one-byte state write, and an indirect operation through an object-like result. Its selection depends on two unsigned 32-bit inputs. A separate 32-bit payload is forwarded to dependencies and, on the direct path, truncated for the byte write.

One selector value, 3, has special precedence: it bypasses the ordinary equality-based return and may trigger a preliminary dependency call before the indirect path.

These are established control-flow properties. The application meanings of the payload, selectors, dependency arguments, and writable byte are unknown. Describing them as a mode change, notification, device operation, or lifecycle transition would be inferred intent rather than an established contract.

## Interface

### Inputs

The function receives three inputs, described here by role rather than source identifiers:

| Input | Recovered representation | Established use |
|---|---|---|
| Payload | Opaque 32-bit value | Forwarded to selected dependencies; its low eight bits are written on the direct path |
| Primary selector | Unsigned 32-bit integer | Compared with 3 and the secondary selector; bit zero selects the direct path outside the special case |
| Secondary selector | Unsigned 32-bit integer | Compared with 1 in the special case and with the primary selector otherwise |

The payload has no established signedness or semantic type. It is not directly dereferenced by this function. Dependencies may interpret it in ways not visible here.

Selector equality is full-width unsigned equality. The parity decision examines only the primary selector’s least significant bit. No range validation or normalization is visible.

### Output

The recovered interface has no return value. Dependency return values are not returned to the caller.

Observable outcomes consist of:

- An immediate return without a visible dependency call or explicit global access.
- A preliminary call on certain special-case inputs.
- A direct-path dependency call followed by a byte write.
- Two calls to the same dependency, followed by an indirect call using the second result.
- Any additional effects, errors, or non-returning behavior of those dependencies.

The function has no visible status result or local error-reporting mechanism.

### Global state and indirect layout

Four global roles are visible:

- **Dispatch argument:** a 32-bit value saved before the chosen nontrivial path runs. The saved value is supplied to the eventual indirect call.
- **Direct argument:** a global value supplied as the first argument of the direct-path dependency.
- **Byte destination:** a globally supplied address used for a one-byte write after the direct dependency returns.
- **Indirect receiver:** a result obtained from the second of two consecutive calls to F-77128790625c.

The indirect receiver is treated as the address of a structure whose first recovered integer-sized field provides a dispatch-table base. The indirect target is obtained from a slot at byte offset 16 from that base.

The supplied representation supports a 32-bit dispatch-layout interpretation. The table offset is explicitly 16 bytes, not a source-level method number. The wider object layout, pointer validity rules, and calling-convention details are not established.

## Behavior

### Selection precedence

The primary selector is first checked against 3.

When the primary selector is 3:

- If the secondary selector is not 1, F-684bbd54b443 is called with the payload and a zero argument.
- If that call returns, execution proceeds to the indirect path.
- If the secondary selector is 1, the preliminary call is skipped and execution proceeds directly to the indirect path.

Ordinary selector equality does not apply in this branch. In particular, primary 3 and secondary 3 do not cause an immediate return.

When the primary selector is not 3:

- Equal selectors cause an immediate return.
- Otherwise, an odd primary selector selects the direct path.
- Otherwise, an even primary selector selects the indirect path.

The equality check therefore precedes the parity decision outside the special case. An odd primary selector does not select the direct path when the selectors are equal.

### Shared read before either operation path

For every execution that reaches the direct or indirect path, the dispatch argument is read and saved first.

This read also occurs on the direct path, even though that path does not visibly use the saved value afterward. The contract does not eliminate that read merely because its value is unused on that path.

For the primary-selector-3 case with secondary selector other than 1, the preliminary F-684bbd54b443 call occurs before this read. Consequently, the saved value may reflect changes made by that dependency.

No claim is made about whether this global is volatile, memory-mapped, synchronized, or concurrently modified. The stated ordering is the order visible in the supplied representation.

### Direct path

The direct path is selected exactly when all of the following hold:

- The primary selector is not 3.
- The selectors differ.
- The primary selector is odd.

After saving the dispatch argument, the function calls F-be23f56df795 with:

1. The direct argument obtained from global state.
2. Zero.
3. The payload.

If that call returns, the function uses the byte destination from global state and writes the payload truncated to one byte. It then returns.

The byte written contains the low eight bits of the payload. The source expresses a character-sized conversion; any signed interpretation of that stored byte is not established here. The observable stored bit pattern is the relevant behavior.

The byte destination is used after F-be23f56df795 returns. It must not be assumed to be the same address that global state identified before the call. No visible earlier snapshot fixes the destination address, and the dependency’s global-state effects are unknown.

The byte write is not performed before the dependency call. If that call does not return normally, this function does not visibly reach the write.

The direct path contains no visible call to F-77128790625c and no visible indirect dispatch.

### Indirect path

After saving the dispatch argument, the function calls F-77128790625c twice in sequence.

The first result is discarded. That does not establish that the first call is unnecessary: the dependency may have side effects, and those effects are unknown.

The second result becomes the indirect receiver. The function then:

1. Reads the receiver’s first recovered integer-sized field to obtain a dispatch-table base.
2. Resolves the indirect call target at byte offset 16 from that base.
3. Invokes the resolved target with the receiver, the previously saved dispatch argument, zero, and the payload, in that order.

The indirect call’s result, if any, is not visibly used. The function returns if the call returns normally.

The saved dispatch argument is obtained before either F-77128790625c call. Changes made by those calls do not replace that saved argument. In contrast, the receiver’s dispatch metadata is read after the second call has produced the receiver.

There is no visible byte-destination write on this path. Dependencies might independently write the same state, but that is outside the established local behavior.

### Exceptional and invalid-state behavior

There are no visible checks for:

- A null or invalid byte destination.
- A null or invalid second result from F-77128790625c.
- An invalid dispatch-table base.
- A missing or invalid target at offset 16.
- Dependency failure.

For defined normal completion, the addresses actually accessed must support the indicated reads or write, and the indirect target must support the supplied call shape. Violations have no established handled outcome. A specific fault, exception, or recovery behavior cannot be promised from this evidence.

## Side effects

The function’s explicit state mutation is limited to the direct path’s one-byte write. It does not visibly allocate, copy, release, or transfer ownership of a memory region.

That observation does not establish ownership behavior inside dependencies. In particular:

- The direct dependency may modify global state before the byte destination is used.
- The preliminary dependency may modify the dispatch argument before it is saved.
- Either F-77128790625c call may alter state.
- The indirect target may modify or release the receiver or other resources.

No retain, release, or cleanup operation is visibly performed on the indirect receiver by this function. This is not evidence that the receiver is borrowed, newly allocated, permanent, or leaked; its ownership contract is unknown.

For normally returning dependencies, the visible order is:

| Selected outcome | Observable sequence |
|---|---|
| Equal selectors, primary not 3 | Immediate return |
| Direct path | Save dispatch argument; obtain direct argument and call F-be23f56df795; use byte destination and write one byte; return |
| Indirect path without preliminary call | Save dispatch argument; call F-77128790625c twice; resolve receiver dispatch metadata; invoke indirect target; return |
| Special case requiring preliminary call | Call F-684bbd54b443; then perform the indirect-path sequence |

These sequences do not impose behavior on concurrent actors or on the internals of dependencies.

## Dependencies

### F-684bbd54b443

Called only when the primary selector is 3 and the secondary selector is not 1.

Its arguments are the payload and zero. Its return value is not used. It runs before the dispatch argument is saved.

Its purpose, state effects, failure behavior, and ownership effects are undocumented by the supplied evidence.

### F-be23f56df795

Called only on the direct path.

Its arguments are the current direct argument, zero, and the payload. Its result is not used. Normal return precedes the byte write.

No global-preservation or memory-ownership guarantee is established.

### F-77128790625c

Called twice on every indirect-path execution that reaches both calls normally.

The calls have no explicit arguments in the supplied representation. The first result is discarded; the second is used as the indirect receiver. The two results are not established to be equal or interchangeable.

### Unidentified indirect dependency

The final target has no supplied opaque function ID. It is identified only by the receiver-derived dispatch slot at byte offset 16.

It receives four arguments: receiver, saved dispatch argument, zero, and payload. Its return value is ignored. Its semantics and ownership effects remain documented_only unknowns.

## Acceptance examples

These examples are conditional acceptance criteria derived from static evidence, not executed tests. They assume accessed addresses are valid and each invoked dependency returns normally.

### Ordinary equality suppresses all operation paths

With primary selector 2 and secondary selector 2, the function returns immediately. It performs no visible dependency call, dispatch-argument read, or byte write.

The same holds for equal selectors of any value other than 3, including 1 and the maximum unsigned 32-bit value.

### Special equality does not suppress processing

With primary selector 3 and secondary selector 3:

- F-684bbd54b443 receives the payload and zero.
- The dispatch argument is saved after that call.
- F-77128790625c is called twice.
- The indirect target is invoked using the second receiver.
- No explicit direct-path byte write occurs.

An implementation that returned merely because these selectors match would violate the observed precedence.

### Special secondary value suppresses only the preliminary call

With primary selector 3 and secondary selector 1, F-684bbd54b443 is not called. The indirect path still runs, including both F-77128790625c calls.

### Odd, unequal, nonspecial selectors use the direct path

With primary selector 5, secondary selector 0, and payload bit pattern 0x123456AB:

- The dispatch argument is read first.
- F-be23f56df795 receives the direct argument, zero, and the full payload.
- After it returns, the byte destination receives bit pattern 0xAB.
- F-77128790625c is not visibly called.

If F-be23f56df795 changes the global byte destination, the subsequent write uses the destination selected after that call, subject to the supplied representation’s memory model.

### Zero remains an ordinary even selector

With primary selector 0 and secondary selector 1, the indirect path runs without a preliminary call. With both selectors 0, the function instead returns immediately.

### Full-width selector values are not narrowed

With primary selector 0xFFFFFFFF and secondary selector 0, the direct path runs: the primary selector is odd and is not 3. With both selectors 0xFFFFFFFF, equality causes an immediate return.

### Saved dispatch argument survives later global changes

Suppose the dispatch argument is saved as value A, and an F-77128790625c call subsequently changes its global source to B. The indirect target still receives A.

Conversely, a change made by the preliminary F-684bbd54b443 call occurs before the save and may affect the argument ultimately passed.

### The second receiver determines dispatch

If the two F-77128790625c calls return distinct valid receivers, dispatch is resolved through the second receiver. The first result is not used for dispatch or explicitly released by this function.

## Uncertainties

The following remain documented_only:

- The application-level meaning of all three inputs.
- The meanings and initial values of the global roles.
- Whether the otherwise-unused direct-path read has externally observable significance.
- Exact machine calling conventions, pointer representation, and signed integer-to-pointer interpretation.
- The complete receiver and dispatch-table layouts.
- Whether dependencies return stable, borrowed, newly allocated, or otherwise managed objects.
- Whether dependencies preserve any global state or release memory.
- Whether the indirect target or any dependency can fail, throw, block, terminate execution, or re-enter this function.
- Concurrency, synchronization, and volatile-access semantics.

No runtime tests have been performed. No claim of operational safety, successful invocation, or observed state transition is made.

## Evidence

The established behavior comes solely from the supplied static function representation: selector comparisons, the low-bit decision, ordered calls, a saved 32-bit global value, a character-sized write, and receiver-based indirect dispatch at byte offset 16.

The indirect-call reconstruction is explicitly uncertain in the supplied material. Accordingly, this contract records the visible target-resolution expression and argument order without assigning an application role, method name, or stronger ABI guarantee.

Comments and naming do not establish dependency semantics. The contract distinguishes visible local behavior from inferred representation details and leaves unavailable callee behavior unknown.
