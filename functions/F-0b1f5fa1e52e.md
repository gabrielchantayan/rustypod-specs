<!-- Trusted provenance: F-0b1f5fa1e52e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0b1f5fa1e52e

## Purpose

F-0b1f5fa1e52e coordinates a conditional state update involving a context, an input record, and an integer-valued local slot whose contents are subsequently used as object addresses.

Its established behavior is to save an input word, request an initial operation, overwrite that input word after initial success, select mode-dependent preparation, and conditionally perform resolution, object processing, completion handling, and a final operation.

Zero permits continuation at each status gate. A nonzero status stops further processing, except that failures after successful resolution can still require completion handling. The value-producing dependency used during preparation is different: its nonzero result becomes an argument rather than an immediate error.

The operation is not established to be transactional. Its direct input-record update precedes several possible failures and is not directly reversed.

“Preparation,” “resolution,” “processing,” and “completion handling” describe call positions and data flow. Their domain meanings and ownership implications are not established.

## Interface

### Arguments

| Position | Role | Established use |
|---|---|---|
| 1 | Context address | Supplies a four-byte word at offset zero for the initial operation. The address itself is forwarded to resolution and preliminary or final operations. |
| 2 | Input-record address | Supplies four-byte words at offsets 68, 72, and 76. Offset 76 is overwritten after initial success. |
| 3 | Mode | Compared exactly with 1 and 5; forwarded unchanged to object processing and the final operation when reached. |
| 4 | Original selector | Initializes the local resolution slot and is independently forwarded to resolution and the final operation. |
| 5 | Incoming word | A four-byte value passed to dependencies and stored at input-record offset 76. |

The field offsets are byte offsets:

- **68, or 0x44:** read only for modes other than 1 and 5, after the initial operation succeeds and the direct store occurs.
- **72, or 0x48:** read from the input record for the initial operation.
- **76, or 0x4c:** read and saved before the initial operation; overwritten with the incoming word after initial success.

A successful resolution path also reads a four-byte word at offset 72 from an address obtained from the local slot. This does not establish that the resolved object and input record have the same layout.

### Widths and address representation

The field accesses and incoming word are explicitly four bytes wide. The saved word therefore retains a 32-bit value. This function does not perform signed comparisons, numerical transformations, or explicit narrowing on those words.

The supplied representation uses integer-typed quantities for the mode, selector, result, input-record address, and local resolution slot. In particular, **the resolution slot is an integer-typed local, not an established pointer-width storage location**. Its address is passed to the resolver; its contents are later used in address formation and as dependency arguments.

The excerpt does not independently define the width of its integer type or recover the original pointer ABI. A 32-bit integer/address interpretation is consistent with the representation, but equal integer and pointer widths are not an established interface guarantee.

Fixed offsets are added to address-bearing values. No explicit overflow, null, alignment, or bounds checks appear. The original address-conversion rules, overflow behavior, and fault behavior remain documented_only.

### Return value and local output

The function returns zero or an unchanged status obtained from a dependency. Nonzero statuses are stopping conditions, not established members of a shared error namespace.

The resolver receives writable access to the local resolution slot, initially containing the original selector. That slot is not a caller-supplied output parameter of F-0b1f5fa1e52e.

Successful resolution need not visibly replace the slot: the function uses whatever value the slot contains afterward. Later uses reload the slot after intervening calls; they are not established to use one stable address.

### Preconditions

For an ordinary, normally returning execution:

- Input-record offset 76 must be readable on entry.
- The context’s first four-byte word and input-record offset 72 must be readable for the initial operation.
- Input-record offset 76 must be writable after initial success.
- Input-record offset 68 must be readable if the mode is neither 1 nor 5 and that preparation path is reached.
- After successful resolution, the current slot value must support the four-byte read at offset 72.
- Arguments passed to dependencies must satisfy those dependencies’ requirements, which are not supplied.

There is no established non-aliasing requirement. The context, input record, and addresses obtained from the slot may overlap.

## Behavior

### Entry snapshot and initial operation

Before any dependency call, the function reads and retains the four-byte word at input-record offset 76. It also initializes the local resolution slot from the original selector.

It then calls **F-c9d1236e0e6b** with:

1. The context’s first four-byte word.
2. The input record’s four-byte word at offset 72.
3. The incoming word.

The two argument-producing reads occur before that dependency is entered. Their relative evaluation order is not established by the supplied expression.

A nonzero result is returned immediately. No direct store to input-record offset 76 and no later dependency call occurs on this path. The initial dependency may nevertheless have modified memory or other state.

After a zero result, the function writes the incoming four-byte word to input-record offset 76. This overwrites any intervening change to that location made by the initial dependency. The retained word remains the entry-time value; it is not refreshed.

### Mode-dependent preparation

For either exact mode value **1** or **5**, the function calls **F-38cbe115d53b** with the input-record address.

- Nonzero: return that result immediately.
- Zero with mode 1: return zero immediately.
- Zero with mode 5: continue to resolution.

Consequently, mode 1 never reaches resolution. Its successful preparation returns after the direct input-record update.

For every other mode, the function reads input-record offset 68 and passes that four-byte value to **F-d1e995cfb2d3**.

- If the result is zero, skip the preliminary operation and continue to resolution.
- If the result is nonzero, call **F-97f36d1a1364** with the context address, that result, literal **4**, and the incoming word.

A nonzero result from this preliminary F-97f36d1a1364 call is returned immediately. A zero result permits resolution.

Thus, a nonzero result from F-d1e995cfb2d3 is a forwarded value, not an immediate failure. There is no further mode validation: every representable mode other than 1 and 5 takes this preparation branch.

### Resolution and the first slot read

The function calls **F-14f40f321d22** with:

1. The context address.
2. The original selector.
3. The address of the writable local resolution slot.
4. Literal **0**.

The slot was initialized before the initial dependency call. It is not reinitialized immediately before resolution.

If resolution returns nonzero, the function returns that result. It does not read object offset 72 or invoke completion handling, even if the resolver changed the slot before failing.

If resolution returns zero, the function reads the slot’s current integer value, uses it to form an address at offset 72, reads four bytes there, and passes that word to **F-81c906bf6e3a**.

This is the value used for the resolved-object field read. It is not established to remain the slot’s value after F-81c906bf6e3a returns.

### Gate failure and completion handling

If F-81c906bf6e3a returns nonzero, the function:

- Reads the local slot again, after that call has returned.
- Passes this newly read value to **F-6cc778185718**.
- Returns the nonzero result from F-81c906bf6e3a.

It does not call F-cdc87186e9b7 or perform the final operation.

The completion argument is therefore not necessarily the address used for the preceding offset-72 read. The resolver received the slot’s address, and whether that address escaped is unknown. An intervening dependency may modify the slot through a retained reference.

### Gate success, processing, and another slot read

If F-81c906bf6e3a returns zero, the function reads the slot again and calls **F-cdc87186e9b7** with:

1. The newly read slot value.
2. The entry-time saved word from input-record offset 76.
3. The incoming word.
4. The mode.

After F-cdc87186e9b7 returns, the function reads the slot again and passes that value to **F-6cc778185718**.

Completion handling occurs whether processing returned zero or nonzero. Its return value, if any, is ignored.

These are separate slot reads across separate dependency calls. Accordingly:

- The address used for the gate’s field read can differ from the processing argument.
- The processing argument can differ from the completion argument.
- Completion handling is not established to receive the object that was checked or processed.

If processing returned nonzero, that result is returned after completion handling. There is no final operation on this path.

### Final operation

If both F-81c906bf6e3a and F-cdc87186e9b7 returned zero, and completion handling returns normally, the function calls **F-97f36d1a1364** with:

1. The context address.
2. The incoming word.
3. The mode.
4. The original selector.

The fourth argument is the original selector, not any version of the resolution slot.

The result of this final call becomes the function’s result unchanged.

## Side effects

### Direct writes and retained values

The sole explicit caller-record store writes the incoming four-byte word to input-record offset 76 after initial success.

The function also initializes the local resolution slot and exposes its address to the resolver. No direct allocation, release, global write, or rollback appears in the supplied body.

The entry-time saved word and original selector have distinct roles from subsequently observed memory. Changes to input-record offset 76 do not replace the saved-word argument. Changes to the resolution slot do not replace the original selector forwarded to the final operation.

### Observable ordering

For a path that reaches the final operation, the established sequence is:

1. Save input-record offset 76 and initialize the local slot.
2. Obtain the initial call’s field arguments and invoke the initial dependency.
3. Store the incoming word at input-record offset 76.
4. Perform mode-dependent preparation.
5. Invoke resolution with writable access to the slot.
6. Read the slot, then read offset 72 through its current value.
7. Invoke the gate with that four-byte field value.
8. Read the slot again and invoke processing.
9. After processing returns, read the slot again and invoke completion handling.
10. Invoke the final operation.

On gate failure, step 8 is replaced by a fresh slot read and completion handling; processing and the final operation are absent.

Reads after calls observe then-current state, not entry snapshots. For example, the offset-68 read occurs after the initial dependency and the direct offset-76 store.

### Ownership and escaped references

The resolver’s access to the local slot is established; whether it retains that address is unknown. While the invocation remains active, the contract must allow intervening dependencies to change the slot through an escaped reference. This possibility explains why the separate reads cannot be collapsed into one stable resolved-address guarantee.

No dependency is assumed to preserve globals or pointed-to memory. No supplied contract establishes ownership acquisition on successful resolution.

F-6cc778185718 occupies a completion position, but its name-independent call placement does not prove release, deallocation, unlocking, or invalidation. In particular, the changing slot value prevents an unconditional claim that it releases the object used earlier.

On ordinary paths, completion handling is invoked once after a successful resolution and a returned gate call. It is not invoked after resolution failure. Exceptions, faults, nonlocal exits, and nontermination are outside these ordinary-path guarantees.

## Dependencies

| Opaque ID | Established use |
|---|---|
| **F-c9d1236e0e6b** | Initial status gate using two four-byte field values and the incoming word. Nonzero stops before the direct record store. |
| **F-38cbe115d53b** | Preparation for modes 1 and 5. Zero completes mode 1 or permits mode 5 to continue. |
| **F-d1e995cfb2d3** | Produces a value from input-record offset 68. Zero skips the preliminary operation; nonzero is forwarded to it. |
| **F-97f36d1a1364** | Optional preliminary operation with literal 4, and a separate final operation with the mode and original selector. |
| **F-14f40f321d22** | Resolution status gate receiving the original selector and writable access to the integer-typed local slot. |
| **F-81c906bf6e3a** | Gate receiving the four-byte word read at offset 72 through the post-resolution slot value. |
| **F-cdc87186e9b7** | Processing call receiving a fresh slot value, the saved entry word, incoming word, and mode. |
| **F-6cc778185718** | Completion call receiving another fresh slot value after the gate-failure or processing call. Its result is ignored. |

Internal effects, retained references, ownership rules, failure atomicity, and additional preconditions of these dependencies remain documented_only.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. They assume valid reached accesses and normally returning dependencies.

### Initial rejection

If F-c9d1236e0e6b returns 9:

- The function returns 9.
- It performs no direct offset-76 store.
- No later dependency is called.
- Effects performed by the initial dependency remain possible.

### Successful mode 1

Given mode 1, an entry-time offset-76 value of 17, an incoming word of 23, and zero results from the initial dependency and preparation:

- Offset 76 receives 23 before preparation.
- The function returns zero.
- Resolution, processing, completion handling, and the final operation are absent.

### Preparation failure after the store

For mode 5, if initial processing succeeds but F-38cbe115d53b returns 7:

- The function returns 7 without resolution.
- The direct incoming-word store has already occurred.
- No direct restoration of the saved word occurs.
- Preparation itself may have changed that location again.

### Other mode with a nonzero preparation value

For mode 0, suppose F-d1e995cfb2d3 returns 12.

The preliminary F-97f36d1a1364 receives the context address, 12, literal 4, and the incoming word. Its nonzero result stops processing; zero permits resolution. The value 12 is not itself returned as an error.

### Resolution failure after slot modification

If resolution changes the slot and then returns 8, the function returns 8 without reading offset 72 through that value and without completion handling.

### Distinct addresses across successful gate and processing

Suppose resolution leaves address A in the slot. The function reads offset 72 at A and invokes the gate.

If the gate changes the slot to B through an escaped reference and returns zero, processing receives B. If processing changes the slot to C, completion receives C.

This behavior does not require A, B, and C to be equal. If processing returns zero, the final operation still receives the original selector, not A, B, or C.

### Gate failure after changing the slot

Suppose the field read uses address A, but the gate changes the slot to B and returns 6.

Completion receives B. The function then returns 6 regardless of completion’s return value. Processing and the final operation are absent.

### Saved word survives intervening record changes

If offset 76 contains 17 on entry and the incoming word is 23, an initial-dependency change of that field to 99 is overwritten by the direct store of 23.

If processing is reached, its saved-word argument is 17, not 99 or 23.

### Processing failure still invokes completion

If processing returns 11, the function reloads the slot and invokes completion before returning 11. The final operation is absent, and no direct rollback occurs.

## Uncertainties

The following remain documented_only:

- Domain meanings of the context, records, selector, modes, and words.
- Original integer and pointer ABI, slot storage width, address conversions, alignment, and overflow behavior.
- Whether resolution must replace the slot on success.
- Whether dependencies retain or share the slot’s address, and which dependencies can modify it.
- Ownership or resource effects of resolution and completion handling.
- Dependency side effects on records, globals, external state, or aliased memory.
- Behavior under invalid addresses, concurrent mutation, exceptions, nonlocal exits, or nontermination.

No thread-safety, atomicity, stable resolved-object identity, rollback, or dependency-purity guarantee is established.

## Evidence

The supplied function body establishes the explicit four-byte accesses, saved entry value, direct store, mode split, call arguments, status gates, and return selection. The supplied dependency mapping establishes the opaque IDs used here.

Separate uses of the integer-typed resolution slot establish separate reads: before the offset-72 access, after a successful gate for processing, and after processing for completion. The gate-failure branch likewise reads the slot after the gate returns. The body does not establish one cached address shared across these operations.

Escaped-reference mutation is a permitted possibility because the resolver receives the slot’s address and no retention or preservation contract is supplied. Actual retention, mutation, domain semantics, and ownership effects are not established.

No runtime tests have been performed.
