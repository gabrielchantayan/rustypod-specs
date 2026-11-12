<!-- Trusted provenance: F-cbcfbe70f640; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cbcfbe70f640

## Purpose

F-cbcfbe70f640 accepts two opaque 32-bit inputs and conditionally returns an unsigned 32-bit result. Its established behavior is a state-driven sequence of arithmetic operations, dependency calls, and possible byte writes into a 16-byte local buffer.

The evidence does not establish an application-level purpose such as decoding, validation, resource acquisition, or identifier generation. The meaning of the inputs, buffer contents, global arithmetic values, and dependency results remains unknown.

The contract below describes individual dispatch actions and their observable ordering. It does not assert that every action is reachable, that actions execute in numerical order, or that execution terminates. Global values and dependency-produced secondary results determine those properties.

## Interface

### Inputs

The routine receives two 32-bit values in a fixed order. Both are saved unchanged during entry initialization.

Their signedness and interpretation are unspecified. In particular, the evidence does not identify either value as a pointer, handle, length, status, or ownership-bearing resource.

A separate pair of 32-bit argument slots starts at zero. Action 4 copies the saved inputs into those slots, preserving their order. Action 0 passes their current contents to F-ab4565f926b1. Forwarding the original inputs therefore depends on action 4 having executed before that call.

### Result

Action 1 returns the current unsigned 32-bit accumulator.

The accumulator starts at zero and can subsequently hold:

- A transformed dependency result.
- A transformed saved accumulator value.
- A byte-write index.
- An intermediate arithmetic and bitwise result.

Consequently, the returned value is not necessarily a dependency’s original result. No success convention, error code, sentinel, or exception protocol is established.

### Local storage

The routine provides a contiguous 16-byte buffer with automatic local lifetime. It does not initialize the buffer as a whole. Its contents before a dependency call depend on any preceding dependency writes or byte-write actions.

A separate 32-bit arithmetic slot starts at zero and is assigned 16 by action 9. The slot is not part of an established buffer structure. Action 9 incorporates its address into a calculation through a signed 32-bit representation; it does not pass that address to a dependency.

A buffer-destination pointer starts null. Action 0 assigns it the local buffer address. Action 2 writes through this pointer, whereas actions 0 and 7 pass the local buffer address directly.

Other local storage holds saved inputs, argument slots, arithmetic state, saved accumulator values, control values, and secondary dependency results. No externally consumable layout for this working storage is established.

### Preconditions and limitations

No visible check ensures that the byte-write destination is valid or that its index lies within the 16-byte allocation.

For action 2 to write within that allocation, its destination and index must identify an existing byte of the buffer. The visible body does not prove this condition. An invalid write is not associated with a defined error return.

The environment must also support the secondary-result convention used after F-870c4fcf7295. The source representation exposes those results, but their ABI meaning is not supplied.

## Behavior

### Entry and dispatch

Entry initializes the accumulator, argument slots, destination pointer, and several arithmetic and control values to zero. It also reads several 32-bit global values to establish initial arithmetic and control state, including the initial dispatch value. The original inputs are then retained in saved slots.

Each dispatch selects an action using a 32-bit difference between the current dispatch value and a global dispatch base. Explicit actions exist for selector values 0 through 9. Selector 6 and all values outside that range execute the same default action.

The dispatch base is consulted at dispatch time. Global operands used within actions are likewise not established as immutable snapshots. Dependency calls may change globals; therefore, a global read before a call must not be replaced by a read after it without additional preservation guarantees.

The action numbers below identify selector values, not an execution schedule.

### Action 0: process the current argument slots

F-ab4565f926b1 receives the current first argument slot, current second argument slot, and local buffer address, in that order.

After that call, its primary result is transformed using global arithmetic operands and stored in the accumulator. The destination pointer is then assigned the local buffer address.

Two calls to F-870c4fcf7295 follow. Each receives a varying 32-bit first argument and the fixed second argument 2,147,483,647.

The first call uses a transformed arithmetic-state value. After it completes, a control value is XOR-updated using a global mask. The second call then uses another arithmetic-state value combined with a global operand.

After the second call, the action samples additional globals and updates working state. The first call’s secondary result supplies the next dispatch value; the second call’s secondary result supplies arithmetic working state. Two further control values are XOR-updated with globally obtained masks before dispatch resumes.

The primary returns of the two F-870c4fcf7295 calls are not visibly consumed. Their calls, secondary results, and possible side effects remain part of the behavior.

### Action 1: return

The routine returns the current accumulator immediately.

This action performs no dependency call, buffer inspection, or explicit cleanup.

### Action 2: calculate an index and write one byte

This action derives character-width intermediate values from byte-transformation state and a global operand. It also updates the separate 32-bit arithmetic slot through a chain of multiplication, addition, and subtraction using global operands.

A further arithmetic calculation produces a 32-bit unsigned index and stores that index in the accumulator. The action writes one byte through the current destination pointer at that index.

The written byte is derived from the character-width intermediates. Its construction includes multiplication, addition, negation, and a bitwise OR. The final store retains eight bits. Plain-character signedness, intermediate narrowing, and promotion behavior require target-level confirmation before the entire transformation can be given a portable arithmetic interpretation.

After the store, the action updates another arithmetic value and computes the next dispatch value by XOR-combining control state with a global operand.

Established boundaries:

- The index is not visibly constrained to 0 through 15.
- The destination is not checked for null.
- The destination byte is not read before being overwritten.
- The accumulator becomes the calculated index.
- Repeated visits can perform repeated writes; no iteration count is established.

### Action 3: transform a saved value

The action transforms a saved 32-bit value using global multiplication, addition, and subtraction operands. The transformed result replaces the accumulator.

It then derives the next dispatch value from a control value and a global XOR operand. No dependency call or buffer access is visible.

The saved value used here can be populated by actions 5 and 7.

### Action 4: prepare the original inputs

This action copies the saved first and second inputs into their corresponding argument slots.

It computes the next dispatch value **before either helper call**. Two calls to F-870c4fcf7295 then execute, both with second argument 2,147,483,647.

The first call’s secondary result is saved into arithmetic working state before the second call. The second call uses a different arithmetic-state value and a separately sampled global operand.

Only after the second call does the action perform the additional control-value XOR update and replace another control value from a global value. It then saves the second call’s secondary result into arithmetic working state and resumes dispatch.

Thus:

- Computing the next dispatch value does not bypass either call.
- The operation between the calls saves the first secondary result; it is not the later control-value XOR or replacement.
- The global operands for the post-call control changes are read after the second call.

### Action 5: transform a saved accumulator

The action transforms an accumulator value previously saved by action 9. The result replaces the accumulator and is also saved for use by action 3.

It then calls F-870c4fcf7295 with a transformed arithmetic-state value and fixed second argument 2,147,483,647. That call’s secondary result becomes the next dispatch value.

### Action 6 and default: choose between two secondary results

The action computes a 32-bit expression involving arithmetic and bitwise operations, stores its result in the accumulator, and extracts its most significant bit by unsigned right shift through 31 positions. The resulting predicate is zero or one.

It also updates the state from which action 2 derives its character-width values.

Two ordered calls to F-870c4fcf7295 follow, each with second argument 2,147,483,647. Their first arguments are derived from the same arithmetic-state value using different global operands. The first secondary result is saved before the second call.

After both calls:

- Predicate one selects the first call’s secondary result as the next dispatch value.
- Predicate zero selects the second call’s secondary result.

Both calls execute regardless of the predicate. This is selection between already-produced results, not conditional suppression of a call.

### Action 7: invoke the buffer dependency and select a continuation

F-6f0d307cf172 receives, in order:

1. The fixed 32-bit value `0x14000001`.
2. Zero.
3. The local buffer address.

Its primary result undergoes arithmetic transformations using global operands. A related 32-bit bitwise expression produces a predicate by unsigned right shift through 31 positions. The predicate is zero or one.

The action updates the accumulator and saves a related transformed result for action 3. It then makes three ordered calls to F-870c4fcf7295, each with second argument 2,147,483,647.

After the first helper call, the action reads globals to replace arithmetic and control values, performs control XOR updates, and saves the first secondary result into arithmetic working state. These updates precede the second helper call.

The second helper call produces a candidate secondary result, which is saved before the third call. After the third call:

- Predicate one selects the second helper call’s secondary result.
- Predicate zero selects the third helper call’s secondary result.

All three helper calls execute. Global operands for the later calls are sampled at their respective call sites, not necessarily before the first call.

This action passes the local buffer directly and does not initialize the destination pointer used by action 2.

### Action 8: update arithmetic working state

The action computes the next dispatch value from control state and a global XOR operand before invoking F-870c4fcf7295.

The helper receives a transformed arithmetic-state value and fixed second argument 2,147,483,647. Its secondary result replaces arithmetic working state.

The already-computed dispatch value is used afterward. A helper-side change to the global XOR operand does not retroactively change that computed value.

### Action 9: initialize arithmetic state and capture call-separated masks

The action assigns 16 to the separate 32-bit arithmetic slot. It derives another arithmetic value using the slot’s address represented through a signed 32-bit quantity and a chain of global arithmetic operands.

Before either helper call, it computes the next dispatch value and saves the current accumulator for action 5.

Two ordered calls to F-870c4fcf7295 follow, both with second argument 2,147,483,647. Their secondary results supply two arithmetic working-state updates.

The control-mask sampling order is significant:

- After the first helper returns, but **before the second helper is invoked**, the action reads and captures a global mask for one later control XOR update.
- The second helper then executes. Its first argument uses another arithmetic-state value and a global operand sampled for that call.
- After the second helper returns, the action reads a different global mask for another control XOR update.
- The secondary results are incorporated into arithmetic working state, and both control XOR updates occur afterward.

The first XOR therefore uses the mask value captured before the second call, even if that call changes the mask’s global storage. The other XOR uses its distinct mask sampled after the second call. Sampling both masks after both calls would not preserve this behavior.

### Arithmetic and termination

Unsigned 32-bit arithmetic has a modulo-\(2^{32}\) interpretation. The predicate-extraction shifts operate on unsigned 32-bit values.

Some intermediate values and expressions are represented as signed 32-bit quantities. The supplied evidence does not establish portable signed-overflow semantics or all conversion details. Address-derived arithmetic and character-width transformations particularly require target confirmation.

No visible iteration bound exists. Normal return requires eventual selection of action 1; that selection is not proved for all inputs or environments.

## Side effects

Directly visible effects include local working-state writes, passing the local buffer address to two dependencies, and one byte store for each execution of action 2.

No direct global assignment is visible. This does not establish global preservation: any dependency may affect globals, external state, or memory accessible through its arguments.

The buffer has automatic lifetime. No explicit allocation, deallocation, ownership transfer, or resource-release operation appears. Passing its address does not establish permission to retain it after return, nor does it establish that a dependency owns or releases it.

The dependencies’ buffer-access extents are unknown. The evidence does not show whether they read existing contents, initialize all 16 bytes, modify only part of the buffer, or access beyond it.

There is no established rollback or cleanup path for dependency failure, an invalid write, or nontermination.

## Dependencies

| Opaque ID | Established interface use | Unresolved contract |
|---|---|---|
| F-ab4565f926b1 | Two current 32-bit argument-slot values followed by the local buffer address; primary result contributes to the accumulator. | Input meanings, result meaning, buffer access extent, failure behavior, ownership, pointer retention, global effects. |
| F-6f0d307cf172 | Fixed arguments `0x14000001`, zero, and local buffer address; primary result contributes to arithmetic state and a dispatch predicate. | Operation selected by the constants, buffer format, status semantics, memory effects, ownership, global effects. |
| F-870c4fcf7295 | Varying 32-bit first argument and fixed second argument 2,147,483,647; secondary results drive arithmetic state and dispatch. | Mathematical operation, secondary-result ABI, errors, global preservation, other effects. |

The fixed second argument does not by itself establish division, remainder, random generation, or range reduction.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. They do not establish reachability.

1. **Immediate return:** If the initial selector is 1, the routine returns zero without invoking a dependency.

2. **Prepared forwarding:** If action 4 precedes action 0 and the argument slots remain unchanged, F-ab4565f926b1 receives the original inputs in their original order.

3. **Unprepared forwarding:** If action 0 occurs before action 4 and no other effect has changed the slots, F-ab4565f926b1 receives zero and zero.

4. **Action 4 ordering:** A trace of action 4 must show dispatch computation before both helper calls, storage of the first secondary result between them, and the additional control XOR and replacement after the second call.

5. **Action 9 captured mask:** Suppose the first mask reads as value A after the first helper, and the second helper changes its storage to B. The later corresponding XOR uses A, not B.

6. **Action 9 later mask:** If the second helper changes the different mask read after that call, the other control XOR uses the post-call value. Both XOR updates still occur after the second call.

7. **Fixed buffer call:** Every action 7 invocation calls F-6f0d307cf172 with `0x14000001`, zero, and the local buffer address, in that order.

8. **Predicate does not suppress calls:** In action 6, predicate zero selects the second secondary result, but both helper calls execute. In action 7, predicate one selects the second of three helper secondary results, but all three calls execute.

9. **Last in-range byte:** If action 2 uses the local buffer as destination and calculates index 15, it overwrites the final byte without first reading that byte. The accumulator becomes 15.

10. **Unrejected unsafe index:** With that same destination and index 16, no visible bounds rejection intervenes. This is an unresolved memory-safety condition, not a specified error result.

11. **Default selector:** A selector outside 0 through 9 executes action 6/default rather than directly returning failure.

## Uncertainties

The following remain **documented_only**:

- Global values, mutability, and the resulting reachable state graph.
- Intended initial action and guaranteed preparation ordering.
- Validity of destination pointers and byte indices.
- Exact character signedness, promotions, and narrowing behavior.
- Target-level signed arithmetic and address-conversion semantics.
- F-870c4fcf7295’s secondary-result ABI.
- Dependency memory access, ownership, retained pointers, global effects, and failures.
- Termination and the caller-visible meaning of returned values.
- Any application-level interpretation of the buffer or fixed arguments.

An apparent intent to coordinate dependency calls and byte transformation through arithmetic control state is an inference only.

## Evidence

This contract uses the supplied static body and opaque dependency mapping. The prior draft and reviewer feedback are not independent evidence.

Established observations include the two 32-bit inputs, unsigned 32-bit result, 16-byte local buffer, initialization, dispatch alternatives, call arguments, byte store, predicate selection, and call-relative global reads. In particular, action 4 places its additional control changes after the second helper, while action 9 captures one mask between helpers and reads another afterward.

Dependency implementations, global contents, target-level arithmetic confirmation, caller guarantees, and execution traces were not supplied. No runtime tests have been performed.
