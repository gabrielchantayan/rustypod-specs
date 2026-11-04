<!-- Trusted provenance: F-9a9fe7f8c7a8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9a9fe7f8c7a8

## Purpose

F-9a9fe7f8c7a8 conditionally performs two callback invocations, evaluates a range predicate between them, and invokes F-1e9b48c57a00 only if all three gates pass. Successful completion writes two fields in the supplied context.

The function unconditionally clears a caller-supplied four-byte location before dispatching any callback. It does not directly populate that location with another value afterward.

Its 64-bit return combines a four-byte payload in the upper half with a status in the lower half:

- Status zero identifies completion of the visible success path.
- Status one identifies failure of a callback-result comparison or the range predicate.

The payload initially contains the output address’s 32-bit representation. A dependency can subsequently change it through the local storage supplied to the second callback or helper. Consequently, the payload is not necessarily transferred data, particularly on failure.

These are static observations. Interpreting the operation as reading a short record and deriving a context field is plausible intent, not an established format or protocol contract.

## Interface

### Arguments

| Argument | Visible representation | Established use |
|---|---|---|
| Context address | 32-bit address-like value | Provides access to a callback-associated object and two success-path destination fields. |
| Output address | Address of a writable four-byte location | Receives an initial zero; its address bits also initialize the return-payload storage. |
| Auxiliary word | Four-byte value | Initializes the second word of the local transfer storage. |
| Initial candidate | Four-byte integer | Initializes the destination supplied to the first callback. |

The visible context accesses and address storage use a 32-bit address model. This does not establish the storage width of every external datum referenced by the function.

### Context and callback layout

| Location | Access width | Established role |
|---|---:|---|
| Context + 4 | 4 bytes | Callback-associated object address. |
| Context + 8 | 4 bytes | Destination for the helper result increased by ten. |
| Context + 12 | 4 bytes | Destination cleared on success. |
| Callback object + 0 | 4 bytes | Dispatch-table address. |
| Dispatch table + 16 | 4 bytes | Callable entry selected for each callback invocation. |

No interpretation is established for other context fields, object members, or dispatch-table entries.

The callback receives four arguments, in this order:

1. The callback-associated object address.
2. A writable destination address.
3. A requested count.
4. A numeric selector.

The first invocation uses count four and selector one. The second uses count six and selector two. Their semantic meanings, including whether the return value conventionally reports bytes transferred, remain unspecified.

### Local storage

Two local regions are exposed to dependencies:

- A four-byte candidate region, initialized from the initial-candidate argument.
- An eight-byte transfer region, initialized with the output address in its first four bytes and the auxiliary word in its next four bytes.

The second callback receives the beginning of the transfer region. F-1e9b48c57a00 receives an address two bytes into that region.

These addresses refer to function-local storage. Passing them establishes neither ownership transfer nor permission to retain and use them after the invocation ends. The supplied body does not establish whether dependencies retain addresses, allocate memory, release memory, or obey particular lifetime conventions.

### Return layout

Let \(P\) be the four-byte bit pattern in the transfer region’s first word when the return is constructed. The 64-bit return is:

\[
R = P \times 2^{32} + S
\]

where \(S\) is zero on success and one on a failed gate. The upper word is placed without sign extension.

The return’s bit layout is established; its caller-side interpretation as a signed or unsigned quantity is not.

### Preconditions

Normal execution requires:

- A valid four-byte write through the output address immediately on entry.
- Valid context, object, dispatch-table, and callable accesses when reached.
- Writable context fields at offsets eight and twelve if the success path is reached.
- Dependencies compatible with the supplied arguments and local storage.
- Normal dependency returns for subsequent behavior to occur.

There are no visible null checks, validity checks, recovery handlers, or destination-capacity checks. The body does not establish a non-aliasing requirement between the output location and reachable context or dispatch storage.

## Behavior

### Initialization and first observable write

The first visible memory effect is a four-byte zero write through the output address.

The function then initializes the transfer region from the output address and auxiliary word, and initializes the candidate region from the initial candidate. The cleared output contents and the address stored in the transfer region are distinct: clearing the output does not clear the copied address bits.

This ordering matters when the output location aliases other reachable storage. For example, clearing a location that overlaps the context’s object reference can change or invalidate the subsequent dispatch. No special handling for such overlap is visible.

### First callback and result gate

After initialization, the function resolves the callback through the context’s object reference, the object’s dispatch table, and the entry at byte offset sixteen.

The first invocation receives the object, the candidate region’s address, count four, and selector one.

The candidate region already contains the initial candidate. A return value of four does not itself prove that the callback wrote four bytes, or wrote anything. The subsequent predicate uses the candidate region’s contents after the callback returns.

The callback’s four-byte integer result must equal exactly four. Every other result takes the failure path. There is no distinct treatment for negative values, zero, or results greater than four.

On this failure path:

- The external-addend expression is not evaluated.
- The second callback is not invoked.
- The helper is not invoked.
- Neither success-path context assignment is performed.

The initial output clear and any callback effects remain in place.

### Candidate and external-addend gate

If the first callback returns exactly four, the function adds the current candidate to an externally referenced addend, converts the resulting expression to an unsigned 32-bit value, and compares that converted value strictly below 255.

Let \(S_{\text{sum}}\) denote the evaluated sum expression and let \(U\) denote its conversion to an unsigned 32-bit value. The established gate is:

\[
U = \operatorname{unsigned32}(S_{\text{sum}}), \qquad U < 255
\]

The accepted converted values are therefore zero through 254 inclusive. A converted value of 255 fails.

For an integral sum that is successfully evaluated, conversion to unsigned 32-bit form retains its residue modulo \(2^{32}\). This describes the conversion and comparison, not the width or signedness of the addition before conversion.

In particular, the external addend’s declaration is absent. Its storage width, read width, signedness, and effects on arithmetic promotion are **documented_only**. The unsigned 32-bit conversion of the sum does not establish that the addend itself occupies or is read as four bytes. It also does not, by itself, prove that the preceding addition is a 32-bit modular operation or that all potential overflow cases have defined behavior.

The addend is consulted after the first callback. Changes made by that callback can therefore affect the predicate. Its value must not be treated as an entry-time snapshot.

If the predicate fails, the function returns failure without invoking the second callback or helper.

### Second callback and result gate

After the first two gates pass, the function resolves the callback again through the context-associated object and dispatch table. The second call is not guaranteed to use the same object or callable as the first: the first callback may have changed reachable dispatch state.

The second invocation receives the currently selected object, the beginning of the transfer region, count six, and selector two.

Its returned four-byte integer must equal exactly six. Any other result takes the failure path without calling the helper or performing the success-path context assignments.

A failing result does not undo destination writes. If the callback changes the transfer region’s first word before returning a value other than six, the changed word contributes to the failure return.

If the callback writes exactly six contiguous bytes beginning at its destination, those writes replace:

- All four bytes of the transfer region’s first word.
- The first two bytes, in increasing address order, of its second word.

The remaining two bytes retain their initialization unless another dependency effect changes them. This conditional layout statement requires no byte-order assumption. A reported result of six is not independent evidence that those writes occurred.

### Helper and successful completion

If the second callback returns exactly six, F-1e9b48c57a00 is invoked with an address two bytes into the transfer region.

Under the exact-six-byte-write condition above, this address identifies the beginning of the final four bytes of the transferred sequence. The helper’s actual access extent, encoding, alignment requirements, and mutation behavior are not established here.

After the helper returns, the function performs these operations in order:

1. Increases the helper’s four-byte integer result by ten and stores the four-byte result at context offset eight.
2. Stores four zero bytes at context offset twelve.
3. Constructs a success return from the transfer region’s current first word and a zero lower word.

No helper-result validation or additional failure gate is visible. A helper result that denotes an error under some separate contract is not visibly recognized as such here.

The result adjustment and destination are four-byte operations, with no visible overflow check or wider stored result. The low-word arithmetic description does not establish general source-language guarantees for signed overflow.

The helper precedes both context assignments and the final payload read. Its effects must not be assumed absent. The success payload may differ from the word present immediately after the second callback if the helper mutates overlapping local storage.

## Side effects

### Direct effects

The function directly performs:

- The unconditional four-byte output clear.
- Initialization of the two local regions.
- On success, the four-byte context-offset-eight assignment.
- On success, the four-byte context-offset-twelve clear.

It has no visible explicit allocation or release operation.

Failure does not imply unchanged context. The initial output write may alias the context, and dependencies may modify it. Likewise, the output location is not guaranteed to contain zero at return: only the function’s own initial zero write is established.

### Ordering and dependency effects

The maximum dependency sequence is:

1. First callback: count four, selector one.
2. Second callback: count six, selector two.
3. F-1e9b48c57a00.

Each later call requires all preceding gates to pass. The external-addend expression is evaluated between the two callbacks, and the context assignments follow the helper.

No rollback, transaction boundary, synchronization, or dependency-state preservation is established. A dependency may affect globals, reachable memory, or dispatch state; such effects cannot be excluded from the contract merely because they are absent from this body.

If a dependency faults, does not return, or otherwise prevents normal continuation, this body establishes no normal-return outcome for that execution.

## Dependencies

| Dependency | Established interaction | Unresolved contract |
|---|---|---|
| F-1e9b48c57a00 | Called once after the second callback returns six; receives the transfer-region address plus two; its four-byte result feeds the context-offset-eight assignment after addition of ten. | Encoding, access extent, mutations, error conventions, global effects, ownership, and retention behavior. |
| Indirect callback target | Resolved through the documented object and dispatch-table offsets for each invocation; receives the documented destination, count, and selector. | Target identity, destination-write guarantees, return-value semantics, state preservation, ownership, and other effects. |
| External addend | Participates in the sum expression after the first callback and before the second. | Declaration, storage width, read width, signedness, value, initialization, mutability, and arithmetic-promotion consequences. |

No opaque function ID is supplied for the indirect target. The two resolutions use the same dispatch-table offset but need not select the same target.

## Acceptance examples

These are conditional static expectations, not executed tests. They assume valid accesses and normal dependency returns. Any absence of additional mutation is an explicit condition, not a dependency guarantee.

1. **First callback fails its result gate.**  
   The first callback returns three. No addend evaluation, second callback, helper call, or success-path context assignment follows. If the transfer region remains unchanged, the return’s upper word is the output address representation and its lower word is one.

2. **Successful result without a candidate write.**  
   The first callback returns four but leaves its destination untouched. The predicate uses the caller’s initial candidate together with the subsequently observed external addend.

3. **Lower range boundary.**  
   The first callback returns four, and evaluation followed by unsigned 32-bit conversion produces zero. The predicate passes and the second callback is attempted.

4. **Strict upper boundary.**  
   A converted sum of 254 permits the second callback. A converted sum of 255 prevents both the second callback and the helper.

5. **Conversion is distinct from operand width.**  
   If the actual operand types permit the sum \(2^{32}\) to be evaluated without invalid overflow, conversion produces zero and the predicate passes. This expectation does not require a four-byte addend and does not establish whether that sum is evaluable for the missing declaration.

6. **Second callback fails after writing.**  
   The first two gates pass. The second callback changes the transfer region’s first word to \(P\), then returns five. With no further mutation, the return is \(P \times 2^{32} + 1\). The helper and success-path context assignments do not occur.

7. **Ordinary successful completion.**  
   The callback results are four and six, and the range predicate passes. The helper receives the transfer-region address plus two and returns 32. Context offset eight receives 42, context offset twelve receives zero, and the return’s lower word is zero.

8. **Negative helper result without overflow.**  
   On the success path, a helper result of negative one produces nine after addition of ten. There is no visible rejection of the negative helper result.

9. **Dispatch changes after the first call.**  
   The first callback replaces the context’s object reference with another valid object and satisfies the first gate. If the range predicate also passes, the second dispatch uses the subsequently observed reference.

10. **Helper changes the payload.**  
    The helper modifies bytes overlapping the transfer region’s first word and returns normally. The success return uses the word present after that call, not a guaranteed snapshot from before it.

## Uncertainties

The following remain **documented_only**:

- The external addend’s declaration, storage width, read width, signedness, value, and initialization.
- The arithmetic type and overflow behavior of the sum before unsigned 32-bit conversion.
- Whether callback results conventionally count bytes and whether reported counts guarantee writes.
- The domain meanings of selectors one and two.
- Byte order, platform alignment rules, and dependency-specific access requirements.
- The helper’s encoding, access extent, mutations, and error conventions.
- Dependency guarantees about global state, caller memory, ownership, release, and address retention.
- Semantic names for the context fields.
- Caller-side interpretation and detailed calling-convention treatment of the 64-bit return.
- Concurrency, synchronization, and atomicity.

A record-reading interpretation remains inferred intent. It does not justify adding record-format validation, memory-management rules, or callback guarantees absent from the supplied body.

## Evidence

The supplied static body establishes the visible argument uses, local four-byte values, context and dispatch offsets, callback arguments, short-circuit gates, call ordering, direct context writes, and 64-bit return construction.

It establishes an unsigned 32-bit conversion of the candidate-plus-addend expression followed by comparison with 255. It does **not** establish the external addend’s storage or read width, because its declaration is not supplied. Access-width claims above are limited to accesses whose widths are visible; they do not extend to that undeclared external datum.

No dependency implementation, addend declaration, platform byte-order declaration, or caller behavior was supplied. No runtime tests have been performed. The acceptance examples describe conditional consequences of the static body, not observed executions.
