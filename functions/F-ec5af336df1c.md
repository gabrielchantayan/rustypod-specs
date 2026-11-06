<!-- Trusted provenance: F-ec5af336df1c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ec5af336df1c

## Purpose

F-ec5af336df1c obtains a primary service, invokes a preparation operation, and requests an initial operation. The low 32 bits of the initial operation’s result determine whether processing stops or continues. Continued processing uses a caller-associated state byte to select follow-up calls, then invokes a final primary-service operation.

The established normal return values are:

| Return value | Condition |
|---|---|
| 9 | The initial operation’s low 32 result bits are zero. |
| 82 | Those low bits are nonzero, processing reaches finalization, and finalization returns zero. |
| 0 | Those low bits are nonzero, processing reaches finalization, and finalization returns nonzero. |

These are numeric outcomes. Their application-level meanings are not established.

An inferred interpretation is that the function prepares a service, attempts an operation, performs state-dependent follow-up, and obtains a final outcome. No business meaning is assigned to the individual calls or constants.

## Interface

The function accepts three arguments and returns a 32-bit value on normal completion.

| Argument | Established role |
|---|---|
| First | A 32-bit address-bearing value. Only after a nonzero low initial result, the function reads one byte at byte offset 748 from this address. |
| Second | A 32-bit value unused by the visible body. It is neither read for a decision nor forwarded to a dependency. |
| Third | A 32-bit value passed unchanged to the primary service’s initial operation. Its meaning is unknown. |

There is no explicit caller-supplied output pointer. Dependencies can produce effects through objects, globals, or other storage accessible to them.

### Memory and dispatch model

The supplied representation establishes a 32-bit address and dispatch model:

- A service object begins with a 32-bit dispatch-table pointer.
- Callable entries are selected by byte offset within that table.
- The caller-associated state field is one byte at offset 748, or `0x2EC`.
- One global supplies a 32-bit value used to initialize a local word.
- Another global supplies an address whose field at byte offset 28, or `0x1C`, is read as a 32-bit integer.

The function initializes two distinct locals:

- A four-byte word, initialized from the first global.
- A separate one-byte local, initialized to zero.

Only the four-byte word’s address is explicitly passed to a dependency.

**The evidence does not establish that these locals are adjacent, their relative address order, or that they form a five-byte object.** The separate zero-valued byte is not established to be accessible through the word’s address. In particular, it cannot be specified as an appended terminator.

The word initialization establishes four initialized bytes at the passed address. It does not establish the dependency’s permitted access extent, an enclosing structure, an encoding, or permission to access additional bytes.

### Preconditions

For the visible accesses and calls to be valid:

- The global supplying the initial word must be readable.
- F-b7af80e829ec must return a usable primary-service pointer.
- Each selected object, dispatch table, and dispatch entry must remain usable when accessed.
- The configuration-address global must identify storage with a readable 32-bit field at offset 28 when the initial operation’s arguments are prepared.
- If the low initial result is nonzero, the first argument must support the one-byte read at offset 748.
- If F-cdd33263ee89 returns nonnull on the special-state path, that returned object must support its dispatch entry at offset `0x104`.

The function does not visibly validate these conditions. In particular, it does not test the primary-service pointer for null.

The offsets describe byte-addressed accesses, not scaled array indexing. Behavior for invalid addresses or address calculations outside valid storage is not established; wrapping behavior is not a contract guarantee.

## Behavior

### Initialization and acquisition

Before calling any dependency, the function initializes the local four-byte word from its global source, establishes a provisional return value of 9, and initializes the separate one-byte local to zero.

The word therefore captures the global value before primary-service acquisition. If a later dependency changes that global, the visible function does not reload it into the word. This is not a guarantee that dependencies leave the word itself unchanged.

The function calls F-b7af80e829ec without explicit arguments and retains the returned pointer as its primary-service pointer.

It then obtains the primary object’s dispatch entry at byte offset `0xA8` and invokes it with the primary-service pointer as its sole visible argument. This operation is labeled F-u01 below. Its return value, if any, does not affect visible control flow.

### Initial operation

After F-u01 returns, the function invokes the primary dispatch entry at byte offset `0x20`, labeled F-u02.

The visible arguments, in order, are:

1. The retained primary-service pointer.
2. The address of the initialized four-byte local word.
3. The third argument to F-ec5af336df1c, unchanged.
4. A Boolean indicating whether the 32-bit configuration field at offset 28 equals 4.
5. The integer constant 3.
6. The integer constant 3.
7. The integer constant 0.

The configuration pointer and field are read for this operation after F-u01 has returned. They are not an entry-time configuration snapshot. Changes made by primary acquisition or F-u01 can therefore affect the comparison.

The representation does not establish the relative evaluation order of the dispatch lookup and configuration expression during preparation of this call. It also does not establish the Boolean argument’s physical calling-convention representation.

F-u02 supplies a result represented as 64 bits. The function extracts:

- The low 32 bits for a zero/nonzero decision.
- The high 32 bits as an intermediate candidate follow-up value.

These are bit portions of the result, not a claim about memory byte order. The decision does not use signed magnitude or ordering.

### Zero low-result path

If the low 32 bits are zero, the function returns 9.

On this path, it does not:

- Read the first argument’s state byte.
- Obtain or invoke the primary dispatch entry at `0x28`.
- Call F-cdd33263ee89.
- Call F-a817d8d58c10.
- Invoke the final primary dispatch entry at `0xA4`.

The high 32 bits do not affect this return.

### Nonzero low-result path

If the low 32 bits are nonzero, the function reads the one-byte field at offset 748 from the first argument and tests whether it equals 2.

This is a single state sample for the branch decision. Later dependency effects do not cause the function to reread the byte or reconsider that decision.

After the sample, the function obtains the primary dispatch entry at byte offset `0x28`, labeled F-u03. The entry is obtained before either state-specific sequence invokes a follow-up operation.

#### State other than 2

For every byte value other than 2, the function replaces the candidate high-result value with zero.

It invokes F-u03 with:

1. The primary-service pointer.
2. The 32-bit value 0.

The return value is ignored. Processing then proceeds to finalization.

The branch is based only on equality with 2. A byte whose bit pattern is `0xFF`, for example, selects this ordinary branch regardless of any signed interpretation of a one-byte value.

#### State equal to 2

For state 2, the function first invokes the already obtained F-u03 entry with:

1. The primary-service pointer.
2. The integer constant 1.

Its return value is ignored.

Only after this invocation returns does the function call F-cdd33263ee89 without explicit arguments.

If that dependency returns null, the function calls F-a817d8d58c10. The supplied static representation identifies F-a817d8d58c10 as nonreturning. Under that annotation, this path does not reach finalization and has no normal return value from F-ec5af336df1c.

If acquisition returns nonnull, the function obtains the returned object’s dispatch entry at byte offset `0x104`, labeled F-u04, and invokes it with:

1. The returned secondary-service pointer.
2. The 32-bit value 2.

Its return value is ignored.

The secondary pointer need not differ from the retained primary pointer. Even if they alias, the `0x104` entry is obtained from the object returned by F-cdd33263ee89.

The initial result’s high 32 bits are not forwarded on this path. The primary follow-up receives 1, and the secondary follow-up receives 2.

### Finalization

After either state-specific sequence completes normally, the function obtains the retained primary object’s dispatch entry at byte offset `0xA4`, labeled F-u05.

It invokes F-u05 with:

1. The retained primary-service pointer.
2. The 32-bit all-ones value, `0xFFFFFFFF`.
3. The integer constant 0.
4. The integer constant 0.
5. The integer constant 0.

The all-ones value has the signed 32-bit interpretation −1, but its semantic interpretation by the dependency is unknown.

The dispatch lookup occurs after the intervening follow-up calls. The function does not use a finalization target captured before those calls. Consequently, it does not require those calls to preserve the primary object’s dispatch-table pointer or entries, although the eventual lookup and call must still be valid.

The finalization result is treated as a 32-bit integer:

- Zero selects return value 82.
- Every nonzero bit pattern selects return value 0.

Neither the magnitude nor the sign of a nonzero result changes that choice.

## Side effects

### Direct memory activity

The function directly performs the following relevant accesses:

- Reads the global 32-bit initialization value.
- Writes the local four-byte word.
- Writes the separate one-byte local.
- Reads service-object dispatch pointers and selected dispatch entries.
- Reads the configuration-address global and its 32-bit field after preparation.
- Reads the caller-associated state byte only on the nonzero low-result path.

The local writes do not establish adjacency or a combined five-byte region.

No direct write to caller-associated storage, either global location, or a service-object field is visible. This does not guarantee that those locations remain unchanged: dependencies may modify them.

### Call ordering

Subject to each preceding operation returning normally, dependency invocation order is:

1. F-b7af80e829ec.
2. F-u01.
3. F-u02.
4. If the low initial result is nonzero, F-u03.
5. For sampled state 2 only, F-cdd33263ee89, followed by either F-a817d8d58c10 on null or F-u04 on nonnull.
6. F-u05 if the nonzero-result path continues normally.

No retry, rollback, compensating action, or explicit cleanup call is visible.

### Ownership and lifetime

The function contains no explicit allocation or release of either service pointer and no explicit ownership-transfer operation. Acquisition semantics alone do not establish ownership.

The four-byte word is local to this invocation. Its address is exposed to F-u02, but the evidence does not establish whether that dependency reads it, modifies it, retains the address, or accesses a larger region. The visible function performs no subsequent explicit read of the word.

The separate one-byte local is initialized but is not explicitly passed by address or subsequently read. Its initialization does not establish that F-u02 receives access to it.

No dependency is assumed to preserve globals, preserve object contents, own memory, release memory, or provide a safe lifetime for a retained local address.

## Dependencies

F-u01 through F-u05 are contract-local opaque labels for unresolved dispatch operations. They identify invocation roles, not recovered concrete function identities; different roles could resolve to the same implementation.

| Dependency | Established use | Unknowns |
|---|---|---|
| F-b7af80e829ec | Called once without explicit arguments to obtain the primary pointer. | Nullability guarantee, ownership, lifetime, effects, and failure behavior. |
| F-cdd33263ee89 | Called only after a nonzero low initial result and sampled state 2, following F-u03. | Acquisition semantics, ownership, lifetime, aliasing, and effects. |
| F-a817d8d58c10 | Called if secondary acquisition returns null; statically annotated nonreturning. | Mechanism of nonreturn, including whether it terminates or transfers control elsewhere. |
| F-u01 | Primary dispatch offset `0xA8`; receives the primary pointer. | Meaning, return convention, and effects. |
| F-u02 | Primary dispatch offset `0x20`; receives seven specified arguments and supplies the 64-bit initial result. | Storage interpretation and accessible extent, result semantics, and effects. |
| F-u03 | Primary dispatch offset `0x28`; receives 0 for ordinary state or 1 for state 2. | Meaning, effects, and return semantics. |
| F-u04 | Returned secondary object’s dispatch offset `0x104`; receives 2. | Meaning, effects, and return semantics. |
| F-u05 | Primary dispatch offset `0xA4`; receives all ones and three zeros; its 32-bit zero/nonzero result selects the final return. | Meaning and effects. |

## Acceptance examples

These are conditional acceptance examples derived from static evidence, not executed tests. Each assumes valid required accesses and normally returning dependencies except where stated otherwise.

1. **Zero low result with nonzero high result.**  
   If F-u02 returns a result whose low 32 bits are zero and high 32 bits are nonzero, the function returns 9. It does not read the state byte or invoke any follow-up or finalization operation.

2. **Ordinary state and zero finalization result.**  
   If the low initial result is nonzero, the sampled state is 0, and F-u05 returns zero, F-u03 receives 0, no secondary service is acquired, and the function returns 82.

3. **Ordinary state at the byte boundary.**  
   If the sampled state has bit pattern `0xFF` and the low initial result is nonzero, F-u03 receives 0. There is no separate negative-state branch.

4. **Special state with successful acquisition.**  
   If the low initial result is nonzero, the sampled state is 2, and secondary acquisition returns nonnull, the order is F-u03 with 1, F-cdd33263ee89, F-u04 with 2, then F-u05. A nonzero F-u05 result produces return value 0.

5. **Special state with null acquisition.**  
   Under the same initial conditions, null secondary acquisition causes F-a817d8d58c10 to be called after F-u03 with 1. Under the nonreturning annotation, neither F-u04 nor F-u05 is reached.

6. **High-result independence.**  
   Holding the low result, sampled state, and subsequent dependency behavior fixed, changing only the high 32 result bits changes no explicitly selected follow-up argument. Ordinary state still passes 0; special state still passes 1 and then 2.

7. **Configuration changed during preparation.**  
   If F-u01 changes the relevant configuration field to 4 and it remains 4 until the comparison, F-u02 receives a true Boolean argument. A value captured before F-u01 would not satisfy this behavior.

8. **State changed after sampling.**  
   If the sampled state is 2 and F-u03 subsequently changes that byte, secondary acquisition still occurs. The decision is not recomputed.

9. **Local-storage initialization without adjacency assumptions.**  
   If dependencies leave the word unchanged before F-u02 receives its address, the word contains the captured global value. Acceptance must not require a fifth zero byte at the next address: only a separate one-byte initialization is established.

10. **Final dispatch changed by a dependency.**  
    If a follow-up dependency changes the primary object’s finalization dispatch entry while leaving it valid, the subsequent F-u05 lookup uses the then-current entry rather than an earlier cached target.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of return values 9, 82, and 0.
- Interpretation of the third argument and reason for the unused second argument.
- Type, encoding, and purpose of the local word.
- Relative addresses, adjacency, padding, or shared-object membership of the two locals.
- The extent of storage F-u02 is permitted or expected to access through the word’s address.
- Whether any dependency retains that address or accesses the separate byte indirectly.
- Concrete calling conventions, Boolean argument representation, and mechanism for returning the 64-bit result.
- Ownership and lifetime guarantees for both service pointers.
- Dependency behavior after invalid memory accesses or other precondition violations.
- Abnormal control flow, blocking, exceptions, and termination behavior of dependencies.
- The exact nonreturning mechanism of F-a817d8d58c10.
- Concurrency, synchronization, reentrancy, atomicity, and memory-ordering requirements.

No unresolved behavior is replaced with an assumed service convention or memory-layout guarantee.

## Evidence

This contract uses only the supplied static function representation and dependency-ID mapping.

Established static evidence includes the explicit widths, access and dispatch offsets, local initializations, call arguments, branch conditions, result-bit extraction, normal return constants, and nonreturning annotation.

The four-byte local and separate one-byte local are independently declared and initialized. Their address relationship is not explicitly established. No five-byte prefix, appended terminator, or broader accessible object is asserted.

Purpose-level interpretation is inferred. Dependency internals, domain semantics, ownership rules, and the remaining layout and failure questions are unresolved.

No runtime tests have been performed.
