<!-- Trusted provenance: F-ff05eb475bc7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ff05eb475bc7

## Purpose

F-ff05eb475bc7 performs three ordered direct dependency calls. After each call, it reads a designated global 32-bit word and writes that observed value to the address returned by the call.

It then performs a branch-selected indirect call, up to two optional indirect calls, and a final indirect call. An address-valued chain argument passes between these stages. On normal completion, the function returns an address 124 bytes before the third direct dependency’s result.

The leading-word replacements and repeated dispatch-table offset suggest lifecycle-related processing across related receivers. This is inferred intent, not an established allocation, construction, destruction, release, or ownership contract.

## Interface

### Inputs and ambient state

The represented interface has no explicit parameters. Behavior depends on:

- An implicit 32-bit condition tested for equality to zero.
- Ambient state consumed or modified by dependencies.
- Three designated global 32-bit storage locations, named **G1**, **G2**, and **G3** in this contract.
- Memory reached through dependency results, receiver fields, and dispatch tables.

The origin and calling convention of the implicit condition are unresolved. It is not established as a conventional caller-supplied argument, nor is its relationship to the preceding calls established.

The globals provide values at distinct observation points:

- **K1**: G1’s value after F-d6fc0dbc11e4 returns.
- **K2**: G2’s value after F-94e041b2e028 returns.
- **K3**: G3’s value after F-7caeaed12e0c returns.

These are not entry-state snapshots or immutable constants. Earlier calls and writes may affect later observations. No dependency is assumed to preserve any global.

### Address and width conventions

The represented data model uses four-byte integer words and 32-bit pointer-bearing quantities. Global reads and replacement writes are 32-bit operations. Receiver-leading words, pointer fields, and dispatch-table entries are represented as 32-bit quantities.

This contract uses the following address names:

| Name | Meaning |
|---|---|
| A | Result of F-d6fc0dbc11e4 |
| B | Result of F-94e041b2e028 |
| P | Result of F-7caeaed12e0c |
| R | Returned address, P − 124 |

All stated offsets are in bytes. Address names do not imply separate objects, allocation boundaries, or ownership. No non-aliasing relationship is established among A, B, P, the globals, or other reached storage.

The optional flag decision depends on the low eight bits of a represented word. The exact physical load width used to obtain those bits is unresolved.

### Output and preconditions

On normal completion, the result is R, equal to P − 124. Indirect-call return values do not determine the final result or the chain argument.

No explicit status or error result is shown.

Normal completion requires valid reached memory accesses, callable indirect targets, and normally returning dependencies. In particular, the representation provides no null-result checks before the three replacement writes. It does not establish portable behavior for invalid, out-of-bounds, or wrapping address adjustments.

## Behavior

### 1. First dependency and replacement

Call F-d6fc0dbc11e4 with no explicit arguments and retain its result as A.

After the call returns:

1. Read the 32-bit word at G1 as K1.
2. Write K1 as a 32-bit word at A.

There is no separate destination-content read or save beyond this specified global read and replacement write. If A aliases G1, the global read reads the existing word at A, and the write writes that observed value back to the same storage. The operation must not be described as necessarily ignoring or changing A’s previous contents.

No null check or destination validation is shown.

After the write, call F-94e041b2e028 with two represented arguments:

- A + 112.
- Zero.

The adjustment is 28 four-byte elements. It does not establish an allocation size.

### 2. Second dependency and replacement

Retain the result of F-94e041b2e028 as B.

After that call returns:

1. Read the 32-bit word at G2 as K2.
2. Write K2 as a 32-bit word at B.

The G2 observation follows the first replacement write and the second direct call. Either may affect the value subsequently read from G2. An entry-time value cannot substitute for this observation.

As in the first stage, there is no separate destination-content read or save. If B aliases G2, this global read also reads B’s existing word, and the replacement writes that value back. No null check is shown.

After the write, call F-7caeaed12e0c with:

- B + 12.
- Zero.

This adjustment is three four-byte elements. The argument uses the actual B returned by the preceding dependency; B need not equal A + 112.

### 3. Third dependency and replacement

Retain the result of F-7caeaed12e0c as P.

After that call returns:

1. Read the 32-bit word at G3 as K3.
2. Write K3 as a 32-bit word at P.

This global observation follows all three direct calls and both preceding replacement writes. Neither those calls nor those writes are assumed to preserve G3.

There is no separate destination-content read or save, and no null check is shown. If P aliases G3, the specified global read reads P’s existing word and the write writes it back.

All subsequent fixed-offset receiver addresses and the returned address are based on this actual P.

### 4. Branch-selected initial dispatch

Test the implicit condition for equality to zero. Exactly one initial branch executes on normal progression.

**When the condition is zero:**

- Obtain a dispatch-table address from the word at P − 12.
- Obtain the indirect target from byte offset 28 in that table.
- Obtain a 32-bit argument value from P − 120.
- Invoke the target with receiver P − 12 and that argument value.
- After normal return, set the chain argument to P − 12.

The representation does not independently establish the relative evaluation order between the argument-field read and the dispatch-target lookup. Both supply the same call.

**When the condition is nonzero:**

- Obtain a dispatch-table address from the word at P − 104.
- Obtain the indirect target from byte offset 28 in that table.
- Invoke the target with P − 104 as its sole represented argument.
- After normal return, set the chain argument to P − 104.

Every nonzero condition selects the same branch, regardless of sign or magnitude.

In either branch, the selected receiver becomes the chain argument. The indirect call’s result is not used for that purpose.

### 5. Optional dispatch through P

After the initial dispatch returns, examine the low eight bits of the represented word at P − 28.

If those bits are zero, skip this call and retain the chain argument.

If they are nonzero:

- Read the dispatch-table address from the word at P.
- Read the indirect target at byte offset 28 in that table.
- Invoke the target with P and the current chain argument.
- After normal return, replace the chain argument with P.

The flag is observed after the initial indirect call. That call can therefore affect whether this optional dispatch occurs.

The dispatch-table word at P is also observed at this later stage. Although K3 was written there earlier, an intervening call may have changed it. An enabled dispatch must not be specified as necessarily using K3.

Only the low eight bits control the decision. Their signed or unsigned interpretation does not alter the zero-versus-nonzero result.

### 6. Optional dispatch through a referenced receiver

After the preceding stage, read the pointer-bearing word at P − 64.

If it is null, skip this stage and retain the chain argument. This skipped path performs neither the indirect call nor its post-call field reload.

If it is non-null:

1. Use the observed pointer as the receiver.
2. Read that receiver’s leading dispatch-table word.
3. Read the indirect target at byte offset 28 in that table.
4. Invoke the target with the receiver and the current chain argument.
5. After normal return, read the pointer-bearing word at P − 64 again.
6. Replace the chain argument with this newly observed pointer.

The reload is behaviorally significant. The replacement chain argument need not equal the receiver just called; it may be another address or null.

No second null check is shown before the reloaded value is passed to the final dispatch.

### 7. Final dispatch and return

After both optional stages:

- Read the dispatch-table address from the word at P − 116.
- Read the indirect target at byte offset 28 in that table.
- Invoke the target with receiver P − 116 and the current chain argument.

This call occurs on every normally progressing path. Its table word is observed after the earlier indirect calls, so their effects may change the selected target.

After this call returns normally, return P − 124. The return uses the retained P, not the final target’s result.

## Side effects

### Ordered reads and writes

The represented replacement operations are:

| Stage | Required preceding call | Global read | Following write |
|---|---|---|---|
| First | F-d6fc0dbc11e4 | G1 → K1 | K1 at A |
| Second | F-94e041b2e028 | G2 → K2 | K2 at B |
| Third | F-7caeaed12e0c | G3 → K3 | K3 at P |

No further call separates a listed global read from its replacement write.

The function explicitly represents three replacement writes. It does not separately save destination contents. This does not imply that destination contents are never read: a specified global read also reads the destination’s existing word when those locations alias.

Aliasing can also make an earlier replacement alter a later global source or make multiple replacement writes affect the same storage. Even when source and destination coincide and the value is unchanged, the specified read and write remain part of the represented behavior.

Calls may perform additional writes or other side effects. Their preservation of globals and reachable memory is not established.

### Relative layout

The later accesses have these relationships to R:

| Byte offset from R | Established use |
|---:|---|
| +4 | 32-bit second argument for the zero-condition initial dispatch |
| +8 | Final receiver and location of its dispatch-table word |
| +20 | Nonzero-condition initial receiver and its dispatch-table word |
| +60 | Nullable pointer field, reloaded after its optional call |
| +96 | Word whose low eight bits control dispatch through P |
| +112 | Zero-condition initial receiver and its dispatch-table word |
| +124 | Third replacement destination and optional receiver P |

These relationships do not establish a complete object layout, declared field types, alignment guarantees, or allocation extent.

### Call ordering and abnormal completion

Normal progression includes three direct calls and between two and four indirect calls:

- One branch-selected initial call.
- Zero or one call through P.
- Zero or one call through the referenced receiver.
- One final call.

Later observations are not general entry-state snapshots. This applies to globals, the optional flag, pointer-field reads, and dispatch-table words.

No rollback, error-handling branch, or recovery mechanism is shown. Earlier writes and call effects are not explicitly reversed if a later operation fails or does not return normally.

### Ownership and lifetime

No allocation, reference-count operation, ownership transfer, or release is directly established.

Returning R does not establish caller ownership. Passing an address to a dependency or target does not establish whether that callee retains, preserves, or releases it.

Normal completion requires that earlier operations leave subsequently accessed storage usable. The available evidence does not guarantee this property.

## Dependencies

| Opaque function ID | Represented arguments | Result use |
|---|---|---|
| F-d6fc0dbc11e4 | None explicit | Supplies A |
| F-94e041b2e028 | A + 112, zero | Supplies B |
| F-7caeaed12e0c | B + 12, zero | Supplies P |

G1, G2, and G3 are global data dependencies, observed after the corresponding calls. Their concrete values, addresses, storage properties, and alias relationships are unresolved.

Each indirect target is obtained from byte offset 28 of the table identified by the relevant receiver’s leading word. Target identities and implementations are unavailable; no supplied opaque function IDs identify them.

No dependency is assumed to preserve globals, fields, dispatch tables, or object lifetimes. No unknown callee is assigned ownership or release semantics.

## Acceptance examples

These are conditional static expectations, not executed tests. Each assumes valid reached accesses and normally returning calls.

### A replacement destination aliases its own source

Suppose A identifies the same four bytes as G1, and those bytes contain a particular word when the first dependency returns.

The specified G1 read obtains that existing word at A. The following write writes the same observed value at A. There is no additional destination-content read or separate saved copy.

The corresponding rule applies when B aliases G2 or P aliases G3.

### A dependency changes its replacement source

If F-d6fc0dbc11e4 changes G1 before returning, the first replacement uses G1’s post-call value, not its entry value.

The same timing rule applies to G2 after F-94e041b2e028 and G3 after F-7caeaed12e0c.

### An earlier destination aliases a later source

Suppose A identifies G2’s four bytes. The first replacement writes K1 into G2.

If F-94e041b2e028 leaves those bytes unchanged and no other relevant change intervenes, the subsequent G2 read yields K1, which is written at B. If that dependency instead changes G2, the replacement at B uses the newly observed value.

### Zero condition with both optional stages skipped

If the condition is zero, the flag’s low byte is zero when examined, and the pointer field is null when examined:

- The initial call receives P − 12 and the word from P − 120.
- Neither optional call occurs.
- The final call receives P − 116 and P − 12.
- The returned address is P − 124.

### Nonzero condition with both optional stages skipped

For any nonzero condition under the same optional-stage observations:

- The initial call receives only P − 104.
- The final call receives P − 116 and P − 104.
- The returned address is P − 124.

### Only the low byte controls dispatch through P

A represented flag word of 256 has a zero low byte, so this dispatch is skipped.

A represented flag word of 128 has a nonzero low byte, so it occurs. If the pointer stage is then skipped, the final call’s second argument is P, regardless of the optional target’s return value.

### The referenced receiver changes during its call

Suppose the first pointer-field read yields a non-null address Q. The optional call receives Q and the current chain argument.

If that call changes the field to S, the post-call reload makes S the final call’s second argument. If the field becomes null, the final call receives null. It does not automatically receive Q.

### Earlier calls change later dispatch state

If the initial call changes the flag’s low byte from zero to nonzero before its observation, dispatch through P occurs.

If an intervening call changes P’s leading word, an enabled P dispatch uses the later word rather than an assumed saved K3.

If an earlier call changes the final receiver’s leading word, the final dispatch uses that later word.

### Dependency results differ from their arguments

If F-94e041b2e028 returns B different from A + 112, the next dependency still receives B + 12.

If F-7caeaed12e0c returns P unrelated to its first argument, subsequent receiver offsets and the return remain based on that actual P.

## Uncertainties

The following remain **documented_only**:

- The implicit condition’s origin, calling convention, and relationship to preceding calls.
- Concrete global addresses, values, storage properties, and alias relationships.
- Whether any dependency actually changes a global or other subsequently read location.
- Indirect-target identities and semantics.
- Allocation boundaries, alignment guarantees, ownership, and lifetime rules.
- Whether dependency results identify interior addresses, aliases, or separate objects.
- Dependency effects on globals, fields, dispatch tables, and memory validity.
- Exception, trap, nonlocal-exit, and other abnormal-completion behavior.
- The physical load width used for the low-byte condition.
- Arithmetic behavior for invalid or wrapping address adjustments.

The absence of represented null checks, target validation, or recovery establishes only that those checks and paths are not shown. It does not establish a particular fault mechanism.

## Evidence

**Established static evidence:** The supplied representation shows three direct calls, each followed by a 32-bit global read and replacement write; four-byte-scaled address adjustments; a zero/nonzero initial branch; two optional dispatch stages; a post-call pointer-field reload; a final indirect call; and a return 124 bytes before P.

The specified global reads can also read destination contents when source and destination alias. There is no additional destination-content read or save represented by the replacement operations.

The observation points establish that globals and later dispatch state cannot be treated as immutable entry-state values. The stated byte offsets follow from the represented four-byte elements.

**Inferred intent:** Leading-word replacement and dispatch through a shared table offset suggest lifecycle-related processing. Stronger lifecycle or ownership semantics are not established.

**Unknowns:** Dependency implementations, actual runtime values, memory validity, concrete aliasing, ownership, and abnormal-completion behavior remain unavailable.

No runtime tests have been performed.
