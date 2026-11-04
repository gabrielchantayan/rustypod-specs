<!-- Trusted provenance: F-5a7627bb8aba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5a7627bb8aba

## Purpose

F-5a7627bb8aba selects between two behaviors using an unsigned 32-bit selector:

- For selectors 0 through 22, it obtains an initial object from F-9d0ecc5f6adf and traverses an object chain. Each visited object receives a dynamically dispatched callback with two captured values and a pointer to mutable result storage. Traversal stops when a callback returns nonzero or the chain becomes null. The function returns the result-storage value, not the callback’s return value.
- For all other selectors, it returns a value selected from two global sources without invoking F-9d0ecc5f6adf or traversing objects.

The first input has no direct use in the supplied representation.

This contract describes statically visible behavior. It does not establish the application-level meaning of the selector, objects, captured values, or returned result.

## Interface

### Inputs and return value

| Element | Established representation | Contract |
|---|---|---|
| First input | One 32-bit value | Not read or forwarded by this function’s visible body. No semantic role is established. |
| Selector | Unsigned 32-bit integer | Determines the branch and, in the traversal branch, selects a table entry. |
| Return value | Signed 32-bit integer | Returns either the mutable result word or the selected global value. No success/error convention is established. |

The selector comparison is unsigned. A 32-bit pattern representing a negative number under a signed interpretation is therefore outside the traversal range.

The supplied representation uses 32-bit object fields and address-bearing words. Layout offsets below describe that represented model; they are not a portability guarantee for a wider-pointer environment.

### Referenced state

For specification purposes, the function accesses these distinct roles:

- **Captured-value global:** a 32-bit value read after F-9d0ecc5f6adf returns.
- **Selection-table base:** a global address used to locate a selector-dependent 32-bit value.
- **Primary-return base:** a global address whose byte offset 4 supplies the primary return value in the alternate branch.
- **Fallback-return global:** a 32-bit value read conditionally in that alternate branch.
- **Object chain:** objects returned by F-9d0ecc5f6adf and reached through a link field.
- **Result storage:** local, mutable storage whose first 32-bit word supplies the traversal branch’s return value.

These labels identify access roles, not guaranteed distinct allocations or nonaliasing regions.

### Required object layout

Each object that is actually visited must support the following accesses:

| Location | Use |
|---|---|
| Object byte offset 0 | Address of a dispatch table |
| Dispatch-table byte offset 100 | Callable address for the per-object callback |
| Object byte offset 20 | Address of the next object, read only after the callback returns zero |

The callback is invoked with four arguments, in this order:

1. The current object address.
2. The captured-value global’s sampled 32-bit value.
3. The selected table entry’s sampled 32-bit value.
4. An address identifying local result storage.

The exact callback calling convention and a separate opaque identity for the callback are not supplied.

## Behavior

### Branch selection

The selector is compared with 23 before either branch’s specific work begins.

- Values 0–22 take the traversal branch.
- Values 23–4,294,967,295 take the global-return branch.

There is no visible validation, normalization, or adjustment of either input.

### Traversal branch: setup and read order

The traversal branch performs the following observable sequence:

1. Calls F-9d0ecc5f6adf without explicit arguments and retains its returned address as the initial current object.
2. Reads the captured-value global.
3. Reads the selection-table base and obtains the 32-bit word at byte displacement \(8s + 4\), where \(s\) is the selector.
4. Initializes the first word of local result storage to zero.
5. Tests whether the current object is null before attempting any dispatch through it.

The global and table reads occur after F-9d0ecc5f6adf returns. Changes that dependency makes before returning can therefore affect the captured values and table lookup.

The two values passed to callbacks are sampled once during setup. This function does not reload them between callbacks. Later global changes by callbacks do not, by themselves, change the argument values already captured for subsequent callbacks.

The selected table word is read even when the initial object is null. A null initial object avoids object dispatch, but it does not bypass the captured-value and table accesses.

### Table layout and arithmetic

The selection table has a visible stride of 8 bytes. Only the 32-bit word at offset 4 within the selected stride is consumed by this function. No meaning or access is established for the other word.

For the accepted selector range:

- Selector 0 reads bytes 4–7 relative to the table base.
- Selector 22 reads bytes 180–183 relative to the table base.
- The displacement expression ranges from 4 through 180.

The selector multiplication and displacement addition cannot overflow a 32-bit unsigned integer within this branch. Validity of the resulting address still depends on the table base and accessible memory. Address validity and address-wrap behavior are not checked by the function.

The representation is consistent with 23 eight-byte table positions, but this function does not establish that an entire 184-byte region is valid or readable. Each execution directly requires only its selected word.

### Per-object processing

For each nonnull current object, the function obtains the callback through the object’s dispatch-table address and the dispatch slot at byte offset 100. It then invokes that callback with the four arguments described above.

After a callback returns:

- **Nonzero return:** traversal stops immediately. The next-object field is not read by this function for that iteration.
- **Zero return:** the function reads the current object’s next-object field at byte offset 20 and makes that value the current object for the next iteration.

The next-object field is read after the callback returns. A callback that changes that field can therefore alter the continuation path. No next-object address is visibly saved before the call.

Each visited object has its dispatch information resolved when that object is processed. The function does not visibly reuse the previous object’s callback address.

### Result accumulation and termination

The first result word is initialized to zero exactly once. It is not reset between callbacks.

The callback’s return value controls traversal only. The final function return comes from the first result word. Consequently:

- A nonzero callback return does not imply a nonzero function return.
- A zero callback return does not discard a result-word change.
- If later callbacks leave the result word unchanged, an earlier write remains available as the final result.
- Exhausting the chain returns the current result word, whether zero or nonzero.
- An initially null chain returns zero after completing setup.

There is no visible cycle detection, visit limit, or timeout. A reachable cycle whose callbacks keep returning zero can prevent termination.

### Global-return branch

This branch does not call F-9d0ecc5f6adf, read the selection table, initialize callback result storage, or traverse objects.

The supplied representation shows two reads of the word at byte offset 4 from the primary-return base:

1. A first read supplies a candidate return value.
2. A second read supplies the zero/nonzero condition.

If the condition read is zero, the function reads and returns the fallback-return global. Otherwise, it returns the value captured by the first read.

With stable memory, this is equivalent to returning the primary word when it is nonzero and the fallback word otherwise. The displayed read sequence is more specific than that shorthand: if the two primary observations differ, the condition comes from the second observation while the retained candidate comes from the first.

The fallback global is read only when the condition read is zero. No check requires the fallback value to be nonzero.

Whether the two displayed reads correspond to two distinct machine-level memory transactions, especially under volatile or concurrent access, is not independently established.

## Side effects

### Direct effects

The visible body:

- Writes zero to the first local result word.
- Updates its local traversal state.
- Reads globals, table storage, object fields, and dispatch entries.
- Invokes F-9d0ecc5f6adf and zero or more indirect callbacks.

No direct global write, allocation, deallocation, object-field write, or ownership transfer is visible.

The local result area is represented as two adjacent 32-bit words. Only its first word is explicitly initialized and used as the function’s return source. The second word has no visible initialization or result role. A valid initial value for that second word must not be assumed.

### Dependency-mediated effects

F-9d0ecc5f6adf and the callbacks may have effects not shown here. This contract does not assume they preserve globals, avoid allocation, release memory, or leave objects unchanged.

The function performs no visible release of objects returned by F-9d0ecc5f6adf or reached through the chain. This does not establish that those objects are owned, borrowed, reference-counted, or leaked; the ownership contract is unknown.

The result-storage address refers to storage local to this invocation. No transfer of that storage’s ownership is visible. Its validity after this function returns is not established.

### Safety preconditions and failures

For ordinary completion, all actually accessed addresses must be valid for their respective reads, and each resolved callback must be callable under the expected convention.

A callback returning zero must leave the current object sufficiently valid for the subsequent next-link read. If it destroys or invalidates that object, the caller has no visible protection before accessing byte offset 20.

No local exception handler, recovery branch, or explicit error conversion is shown. Invalid pointers, invalid dispatch entries, callback failures, and nonlocal exits are not converted into a defined status by the visible body.

## Dependencies

### F-9d0ecc5f6adf

- Called exactly once in the traversal branch.
- Not called in the global-return branch.
- Receives no explicit arguments.
- Supplies the initial object address; null is handled without an object dereference.
- Runs before the captured global and selected table word are read.

Its implementation, effects, ownership rules, and failure behavior are unknown.

### Indirect callback dependency

No opaque function ID is supplied for the indirect callback target. Its identity is selected dynamically from each current object’s dispatch table.

The visible caller establishes:

- The dispatch slot’s byte offset is 100.
- Four arguments are supplied.
- A zero return requests continuation through the current object’s post-call next link.
- Any nonzero return stops traversal.
- The callback can communicate the eventual function result through the first result word.

The caller does not establish what the callback’s nonzero values mean, whether it writes result storage, or what other state it may change.

## Acceptance examples

These are conditional acceptance criteria derived from the supplied static representation, not executed tests.

1. **Null initial object still requires setup reads.**  
   Given selector 0, valid captured-value and table storage, and a null return from F-9d0ecc5f6adf, the function reads the table word at displacement 4, invokes no object callback, and returns zero.

2. **Highest traversal selector uses the final visible displacement.**  
   Given selector 22, the selected callback argument comes from the 32-bit word at displacement 180. Selection alone does not read either neighboring word.

3. **Boundary selector bypasses traversal.**  
   Given selector 23 and a stable nonzero primary word of 41, the function returns 41 without invoking F-9d0ecc5f6adf or an object callback.

4. **Unsigned high-bit selector takes the global branch.**  
   Given selector bit pattern `0xFFFFFFFF`, primary word zero, and fallback word −7, the function returns −7. It does not treat that selector as a negative in-range index.

5. **Callback status is not the returned result.**  
   Given one object whose callback leaves the result word unchanged and returns 9, the function returns zero. It does not return 9 and does not read that object’s next link afterward.

6. **A result survives continued traversal.**  
   Given a first callback that writes 17 and returns zero, followed by a second callback that leaves the result word unchanged and returns nonzero, the function returns 17.

7. **Chain exhaustion retains the result.**  
   Given a callback that writes −3, returns zero, and leaves a null next link, the function returns −3.

8. **A callback can redirect continuation.**  
   Given a callback that changes its current object’s next link and returns zero, the next iteration uses the post-call link value, provided the object remains valid.

9. **Captured arguments remain sampled.**  
   Given a callback that changes the captured-value global or selected table word, later callbacks still receive the originally sampled argument values. The function performs no visible reload of those values.

10. **Differing primary observations affect selection.**  
    Conditional on the two displayed primary reads being distinct observable accesses, if the first yields 12 and the second yields zero, the fallback is returned. If the first yields zero and the second yields 12, the retained return value is zero and the fallback is not read.

## Uncertainties

The following are **documented_only**:

- The semantic purpose of the selector, captured values, result word, and global return sources.
- The significance, if any, of the unused first input.
- Dependency ownership, allocation, release, and global-preservation rules.
- The indirect callback’s complete interface contract and exact calling convention.
- Any legitimate callback use of the second local word or retention of the result-storage address.
- Whether globals or object fields are volatile, atomic, concurrently mutable, or externally synchronized.
- Whether all nominal table positions exist in valid contiguous storage.
- Machine-level access ordering beyond what the supplied representation shows.
- Behavior after invalid memory access, invalid dispatch, or dependency failure.
- Any higher-level interpretation of zero or nonzero returned results.

An inferred intent is a first-accepting-object search with a separate result channel. That interpretation is not required: the established behavior is ordered callback traversal controlled by callback status, with a separately returned mutable word.

## Evidence

The supplied static function representation supports:

- An unsigned selector boundary of 23.
- An eight-byte table stride and four-byte within-position displacement.
- A setup call to F-9d0ecc5f6adf before captured-value reads.
- A dispatch-table slot at byte offset 100.
- A next-object field at byte offset 20.
- One zero-initialized 32-bit result word shared across callback invocations.
- Continuation on callback zero and stopping on callback nonzero.
- A separate global-return branch with a conditional fallback.

No dependency implementations, runtime traces, executable observations, or external interface documentation were supplied. No runtime tests have been performed.
