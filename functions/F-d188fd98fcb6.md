<!-- Trusted provenance: F-d188fd98fcb6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d188fd98fcb6

## Purpose

F-d188fd98fcb6 conditionally compares an unsigned 32-bit requested quantity against an unsigned 32-bit quantity obtained from object-associated state. If the request does not exceed the comparison quantity, the function returns their difference in the low half of its result. Otherwise, that half is zero.

Before the final comparison, an object-dispatched policy operation may enable replacement of the stored state word. Replacement also clears a separate state word and obtains a new comparison quantity.

The 64-bit result contains two independently meaningful halves:

- Bits 63–32 contain the first word of a local three-word argument area, as it exists at return.
- Bits 31–0 contain an early sentinel result, an unsigned difference, or zero.

The evidence does not establish that callers interpret this result as one conventional 64-bit number.

**Inferred intent:** The comparison, conditional replacement, and fixed increment suggest an allowance, limit, or deadline adjustment. Neither that interpretation nor any unit of measurement is established.

## Interface

### Inputs

The function receives four inputs:

| Input | Established representation and use |
|---|---|
| Object reference | An address used to access dispatch information and associated state. |
| First argument word | An opaque 32-bit value copied into the first local argument word. |
| Second argument word | An opaque 32-bit value copied into the second local argument word. |
| Third argument word | An opaque 32-bit value copied into the third local argument word. |

Before the initial dispatched call, the function initializes a local argument area in this order:

| Relative byte offset | Initial contents |
|---|---|
| 0 | First argument word |
| 4 | Second argument word |
| 8 | Third argument word |

Two dependencies later receive the address of its first word. Their behavior is not supplied. The function does not establish that they leave this area unchanged.

### Object and state layout

The supplied representation uses 32-bit words, including address-bearing fields. The relevant access layout is:

| Location | Established role |
|---|---|
| Object + `0` | Dispatch-table address. |
| Object + `8` | Associated-state address. |
| Dispatch table + `8` | Initial eligibility operation. |
| Dispatch table + `0x194` | Replacement-policy operation. |
| Associated state + `0x10` | Nested-state address. |
| Associated state + `0x90` | Word cleared after replacement. |
| Nested state + `0x24` | Stored word examined, converted, and potentially replaced. |

These offsets describe access requirements, not complete object definitions. Class identity, allocation boundaries, and ownership are unknown. The representation does not establish an independently portable calling convention.

### Output

The output is a 64-bit concatenation of two 32-bit words:

| Portion | Contents |
|---|---|
| High 32 bits | First local argument word at return. |
| Low 32 bits | Selected result word. |

The low word is:

- `0xFFFFFFFF` when the initially examined stored word is the sentinel;
- the final comparison quantity minus the requested quantity when the request is unsigned-less-than-or-equal to that quantity; or
- zero otherwise.

These results are not uniquely tagged. Zero can represent an early rejection, an insufficient comparison quantity, or an exact match. `0xFFFFFFFF` can represent either the initial sentinel or an ordinary unsigned difference.

### Preconditions

All addresses actually accessed must support the corresponding read, write, or dispatched call.

In particular:

- The object and its initial dispatch information must be valid before eligibility can be determined.
- A nonzero eligibility result leads to an associated-state traversal. The associated-state address is not checked for null.
- Only the initially obtained nested-state address is checked for zero.
- Later state traversals do not repeat that null check.
- Every dispatch entry used must be callable with the object reference.
- Replacement requires writable storage for the nested stored word and associated-state clear word.

Callees are not established to preserve dispatch information, state links, stored words, or globals. If they alter reachable state, the addresses subsequently accessed must still be valid.

## Behavior

### Initialization and eligibility

The function first copies all three argument words into the local argument area. This initialization precedes the initial eligibility call and every sentinel check.

It then reads the object’s dispatch information and invokes the operation at dispatch-table offset `8`, passing the object reference.

If that operation returns zero:

- The function does not traverse the associated-state chain.
- None of the four identified dependencies is called.
- The low result is zero.
- The high result is the original first argument word.

If eligibility is nonzero, the function reads the associated-state address at object offset `8`, then the nested-state address at associated-state offset `0x10`.

If the nested-state address is zero, processing ends with a zero low result and the original first argument word in the high result. No stored-word read or identified dependency call occurs on this path.

### Initial sentinel

For a nonzero initial nested-state address, the function reads its word at offset `0x24`.

If that word is `0xFFFFFFFF`, the function returns immediately with:

- low word `0xFFFFFFFF`;
- high word equal to the original first argument word.

This sentinel check occurs **after local argument initialization and the initial eligibility call**. It precedes the call to F-59222578e3f0 and all subsequent comparison-path calls, including the replacement-policy operation.

The sentinel check applies only to this initial stored-word read. A later stored word or dependency result equal to `0xFFFFFFFF` does not trigger the same early-return rule.

### Obtaining the quantities

For an initially nonsentinel stored word, the function performs these operations in order:

1. Calls F-59222578e3f0 with the local argument-area address. Any return value is unused.
2. Calls F-b8274406a146 with that same address and a second argument equal to `1`. Its 32-bit result becomes the requested quantity.
3. Reads the object’s associated-state and nested-state links again, then reads the current nested stored word.
4. Calls F-dd8a773e7b58 with that stored word. Its 32-bit result becomes the comparison quantity.
5. Reads the object’s dispatch information and invokes the operation at dispatch-table offset `0x194`, passing the object reference.

Both quantities are compared as unsigned 32-bit values.

The stored word passed to F-dd8a773e7b58 is obtained after the two local-area calls. It need not equal the word examined by the initial sentinel check.

Likewise, the replacement-policy operation is selected after the first conversion call. Neither its dispatch table nor the state graph is established to be an entry-time snapshot.

### Deciding whether to replace

If the replacement-policy operation returns zero, no further stored-word read is performed for the replacement condition, and replacement is skipped.

If it returns nonzero, the function traverses the current state chain and reads the nested stored word. Replacement occurs if either:

- that newly read stored word is zero; or
- the previously obtained comparison quantity is unsigned-less-than the requested quantity.

The current stored-word read occurs before the quantity comparison in this condition. A zero stored word is sufficient to select replacement, even if the earlier comparison quantity already satisfies the request.

If the current stored word is nonzero and the earlier comparison quantity is at least the request, replacement is skipped.

The comparison quantity used in this decision remains the result of the earlier F-dd8a773e7b58 call. The function does not reconvert the word read after the policy call merely to decide whether replacement is necessary.

### Performing replacement

When replacement is selected, the function:

1. Adds `0x278D00` to the requested quantity using 32-bit unsigned arithmetic.
2. Calls F-8c08b0c651a4 with that wrapped sum.
3. Traverses the object’s state chain and writes the dependency’s 32-bit result to nested-state offset `0x24`.
4. Reads the associated-state address from the object and writes zero at associated-state offset `0x90`.
5. Traverses the state chain again and reads the nested stored word.
6. Calls F-dd8a773e7b58 with that word, replacing the earlier comparison quantity with this new 32-bit result.

The increment is decimal 2,592,000. Its unit is unknown. Addition wraps modulo \(2^{32}\); there is no overflow check or saturation.

The replacement-producing call precedes the destination-state traversal and store in the supplied representation. Consequently, state changes made by F-8c08b0c651a4 can affect the destination selected afterward.

The stored-word write precedes the clear, and the clear precedes the read used for the second conversion. These are distinct state-chain accesses, not operations guaranteed to use one permanently captured associated-state or nested-state object.

The replacement result is not validated. Zero and `0xFFFFFFFF` are both accepted as stored bit patterns. Neither causes an additional sentinel check.

### Final comparison

After replacement is either completed or skipped:

- If the requested quantity is unsigned-less-than-or-equal to the final comparison quantity, the low result is their 32-bit difference.
- Otherwise, the low result is zero.

A difference with its high bit set is preserved. It is not clamped to a signed-positive range.

The high result is taken from the first local argument word at return. On the ordinary comparison path, changes made by either dependency receiving the local area can therefore affect the high result. The return construction does not use a separately saved copy of the original first argument.

## Side effects

### Explicit writes

The function initializes its three-word local argument area on every path.

Its explicit object-associated writes occur only when replacement is selected, in this order:

1. The result of F-8c08b0c651a4 is stored at nested-state offset `0x24`.
2. Zero is stored at associated-state offset `0x90`.

There is no rollback when the subsequent conversion yields a quantity smaller than the request. A zero low result can follow both writes.

Paths without replacement contain no explicit object-associated writes by this function. This does not establish that their callees have no side effects.

### Ownership and lifetime

The three opaque argument words are copied by value. No write back to caller-provided argument storage is shown.

The local area’s address is passed to two dependencies. Their full access extent, pointer-retention behavior, and ownership assumptions are unknown. The storage is local to this invocation; the function provides no mechanism extending its lifetime.

No explicit allocation, deallocation, ownership transfer, or resource release appears in this function. Such behavior must not be assumed absent or present inside dependencies.

### Errors and synchronization

No distinct error channel is established.

The function does not:

- comprehensively validate addresses or dispatch entries;
- distinguish all causes of a zero low result;
- validate conversion or replacement outputs;
- repeat the initial sentinel check;
- undo completed replacement writes; or
- perform explicit synchronization.

Dependency failure, nonreturning behavior, invalid memory access, and concurrent mutation are not assigned defined outcomes by the supplied evidence.

## Dependencies

| Opaque identifier | Established use |
|---|---|
| F-59222578e3f0 | Called once on the ordinary comparison path with the local argument-area address. Its return value is unused. |
| F-b8274406a146 | Called after F-59222578e3f0 with the same address and the word `1`. Supplies the unsigned requested quantity. |
| F-dd8a773e7b58 | Receives a current stored word and supplies the unsigned comparison quantity. Called once on the ordinary path and a second time after replacement. |
| F-8c08b0c651a4 | Receives the wrapped requested quantity plus `0x278D00`. Supplies the word written during replacement. |

The two object-dispatched operations have no supplied opaque identifiers. Their established identities are limited to dispatch-table offsets `8` and `0x194`.

No dependency is established to be pure, preserve globals, preserve the object graph, preserve local arguments, or own or release memory. Their roles above describe only how this function uses their arguments and results.

## Acceptance examples

These are conditional consequences of the static representation, not observed executions. Unless stated otherwise, addresses remain valid, calls return normally, and dependencies leave unrelated state unchanged.

### Eligibility rejection

Given any three argument words, when the initial dispatched operation returns zero:

- All three words have already been copied into the local area.
- No associated-state traversal or identified dependency call follows.
- The low result is zero.
- The high result is the original first argument word.

### Missing initial nested state

When eligibility succeeds but the initially read nested-state address is zero:

- No stored word is read through that address.
- No identified dependency is called.
- The low result is zero.

This example does not permit a null associated-state address: that address is dereferenced to obtain the nested-state address.

### Initial sentinel

When eligibility succeeds and the initial stored word is `0xFFFFFFFF`:

- Local argument initialization has already occurred.
- F-59222578e3f0 and all later comparison-path calls are skipped.
- The low result is `0xFFFFFFFF`.
- The high result is the original first argument word.

### Sufficient quantity without replacement

If the requested quantity is `20`, the first conversion returns `35`, and replacement policy returns zero:

- No replacement-condition stored-word read follows the policy call.
- F-8c08b0c651a4 is not called.
- Neither explicit object-associated write occurs.
- The low result is `15`.

If a local-area dependency changed the first local word, that changed word appears in the high result.

### Insufficient quantity with replacement disabled

If the request is `36`, the first conversion returns `35`, and replacement policy returns zero, the low result is zero without replacement.

### Zero stored word selects replacement

If policy returns nonzero and the subsequent current stored-word read yields zero, replacement occurs even when the first comparison quantity exceeds the request.

The final comparison uses the second conversion result.

### Replacement can leave an insufficient quantity

If the request is `100`, replacement occurs, and the second conversion returns `99`:

- The replacement word has been stored.
- The associated-state word at offset `0x90` has been cleared.
- The low result is zero.

The writes are not reversed.

### Replacement destination follows the producing call

If F-8c08b0c651a4 changes the object’s associated-state link to another valid state chain, the subsequent replacement store follows that changed chain. The function does not require the store to target the nested state examined before that call.

### Addition wraps

If the request is `0xFFFFFFFF` and replacement occurs, F-8c08b0c651a4 receives `0x00278CFF`.

### Result collisions

If request and final comparison quantity are both `35`, the low result is zero despite a successful comparison.

If the request is zero and the final comparison quantity is `0xFFFFFFFF`, the low result is `0xFFFFFFFF` without requiring the initial sentinel path.

## Uncertainties

The following remain **documented_only**:

- Application meaning and units of the quantities, stored word, and increment.
- Semantic identities of the dispatched operations.
- Complete object layouts, allocation sizes, and ownership.
- Dependency behavior beyond the established call contracts.
- Whether the local-area dependencies modify its first word or other contents.
- Whether dependencies mutate globals, dispatch information, or state links.
- Dependency access bounds, pointer retention, allocation, and release behavior.
- Dependency failure, exceptional, or nonreturning behavior.
- Caller interpretation of the packed result.
- Physical result ABI and platform byte order.
- Behavior under concurrent mutation.

The allowance, limit, or deadline interpretation is inferred intent, not an acceptance requirement.

## Evidence

Established behavior comes from static examination of the supplied representation: argument-word initialization, 32-bit memory accesses, dispatch offsets, call order, conditional reads, unsigned comparisons, wrapped addition, explicit stores, and high/low return composition.

In particular, the representation places all three local argument initializations before the initial dispatched call. The initial sentinel check occurs later and precedes F-59222578e3f0 and the remaining comparison-path calls; it does not precede local initialization.

The representation establishes these operations but not broader dependency semantics, ownership rules, application meaning, or runtime safety.

No runtime tests have been performed. The acceptance examples are conditional static expectations, not execution reports.
