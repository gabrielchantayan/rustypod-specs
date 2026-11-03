<!-- Trusted provenance: F-9c6e06f5ecb8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9c6e06f5ecb8

## Purpose

F-9c6e06f5ecb8 selects between a stored default result and a dependency-mediated lookup. The choice depends on whether an optional descriptor is present and whether its control word contains any set bit above bit zero.

The function also writes an output word identifying the selected input:

- On the default path, it writes the supplied selector.
- On the lookup path, it writes the descriptor address, including when the lookup subsequently fails.

A negative lookup result triggers a diagnostic dependency, increments a context counter, and is normalized to a return value of −1.

These are established behaviors of the supplied static representation. Describing the operation as a database lookup is inferred intent, not an established contract for either dependency.

## Interface

### Inputs

The function has four arguments:

| Role | Interpretation |
|---|---|
| Context | Address of a writable context whose first word supplies an address-like value. An error path also accesses a counter at byte offset 64. |
| Selector | Unsigned integer passed to the lookup dependency or copied to the output word. |
| Optional descriptor | Zero denotes absence. A nonzero value is treated as an address, from which a control word is read at byte offset 4. |
| Output location | Address of a writable unsigned result word. The function writes this location on every normally completed path. |

The supplied representation indicates a 32-bit integer and address model. Under that model:

- The selector, descriptor address, descriptor control word, and output word are unsigned 32-bit quantities.
- The function result, lookup result, default result, and context counter are represented as signed 32-bit integers.
- The context counter is 16 integer elements from the context base, corresponding to byte offset 64.
- The default result is read at byte offset 120 from the address obtained from the context’s first word.

The exact target ABI, pointer width, byte order, and alignment rules are not independently established. The byte-offset interpretation of the counter depends on the indicated four-byte integer layout.

### Referenced layouts

Only these fields are relevant to the visible body:

| Referenced object | Offset | Visible use |
|---|---:|---|
| Context | 0 | Read a word used as the lookup argument or as the base address for the default-result read. |
| Context | 64 | After a failed lookup and diagnostic call, read, increment, and write the counter. |
| Optional descriptor | 4 | Read an unsigned control word when the descriptor address is nonzero. |
| Object addressed by the context’s first word | 120 | Read the signed default result on the default path. |

No read of the descriptor’s first word is visible. No other descriptor fields are examined by this function.

### Preconditions

For ordinary completion:

- The context must support a read of its first word.
- A nonzero descriptor must support a read at its address plus 4.
- The output location must support a word-sized write.
- The default path requires a readable signed word at the address obtained from the context’s first word plus 120.
- A lookup failure requires readable and writable counter storage at context offset 64 after the diagnostic dependency returns.
- Both dependencies must satisfy whatever additional preconditions their own contracts impose.

A zero descriptor is explicitly supported. A null or otherwise invalid context or output location is not handled explicitly.

### Outputs

The function returns:

- The stored default result on the default path, without interpreting its sign.
- The lookup dependency’s result if that result is nonnegative.
- Exactly −1 if the lookup dependency returns a negative value and the subsequent diagnostic and counter operations complete normally.

The output word is not a success indicator. In particular, it receives the descriptor address before a lookup failure can be detected.

## Behavior

### Selection rule and initial reads

If the descriptor is nonzero, the function first reads its control word at offset 4. It then reads the context’s first word.

If the descriptor is zero, no descriptor memory is read; the context’s first word is still read.

The lookup path is selected only when both conditions hold:

1. The descriptor is nonzero.
2. Its unsigned control word, shifted right by one bit, is nonzero.

For a 32-bit control word, the second condition is equivalent to at least one of bits 1 through 31 being set. Thus:

- Control values 0 and 1 select the default path.
- Control values 2 through 4,294,967,295 select the lookup path.
- Bit zero alone does not enable lookup.
- The control word is not treated as a signed number.

Descriptor absence always selects the default path, regardless of the selector value.

### Default path

The default path performs these observable actions in order:

1. Read the signed result at offset 120 from the address obtained from the context’s first word.
2. Write the selector to the output location.
3. Return the previously read result.

Neither dependency is called on this path.

The sign of the stored default result has no effect on error handling. A negative default result is returned directly; it does not trigger the diagnostic dependency or the counter increment.

The read precedes the output write. If the output location aliases the stored default-result word, the function returns the value read before overwriting that storage with the selector.

### Lookup path

The lookup path performs these observable actions in order:

1. Write the descriptor address to the output location.
2. Call F-d34857dd0073 with the value previously read from the context’s first word and the selector.
3. Interpret the returned value as signed and test whether it is negative.

If the result is nonnegative, it is returned unchanged. No further operation by this function is visible.

The first dependency argument is the context word captured before the output write. Consequently, if the output aliases the context’s first word, the write changes context storage but does not change the argument already obtained for this call.

### Lookup failure

If F-d34857dd0073 returns a negative value:

1. F-2022b0f73f3b is called with the context address, a fixed diagnostic-data address, and the selector.
2. After that call returns, the counter at context offset 64 is read.
3. One is added to that counter, and the result is written back.
4. The function returns −1.

Every negative lookup result takes this same visible failure path. The original negative value is not preserved as the function’s return value.

The counter is read after the diagnostic call, not before it. Therefore, any counter change made by that dependency affects the increment performed here. The increment must not be modeled as operating on the counter’s entry value.

The body does not restore the output word on failure. Its own output write has already stored the descriptor address. A dependency could nevertheless modify that storage through aliases or other accessible state; dependency preservation is not established.

## Side effects

### Direct writes

The visible body performs:

- One output-word write on either selected path.
- One counter write only after a negative lookup result and a normally returning diagnostic call.

The output write occurs before the lookup call on the lookup path, but after the default-result read on the default path.

There is no visible allocation, deallocation, explicit resource acquisition, or explicit resource release in this function.

### Aliasing

The supplied body contains no explicit non-aliasing check. Where overlapping accesses are valid under the target’s memory rules, the specified ordering matters:

- Descriptor control is read before the output write. An output alias to that word does not change the already selected branch.
- The context’s first word is read before the output write. An aliasing output write does not replace the already captured lookup argument or default-result base.
- The default result is read before an aliasing output write can overwrite it.
- If the output aliases the error counter, the descriptor-address write occurs before both dependencies. The later counter increment uses the counter value present after the diagnostic call.

These ordering statements do not establish that every possible overlap is valid under the target ABI or memory model.

### Ownership and dependency effects

The function directly accesses caller-provided storage without an explicit ownership operation. That does not prove that either dependency preserves or releases any object.

In particular, the visible body continues to use the context after the diagnostic call. Successful execution therefore requires that the counter location remain accessible, but the supplied evidence does not establish that F-2022b0f73f3b guarantees this.

Global mutations, logging, external I/O, allocation, and other dependency effects remain unknown.

### Errors and arithmetic

There is no visible recovery for invalid addresses, inaccessible memory, dependency non-return, or dependency exceptions or equivalent abnormal transfers.

The counter update is a signed increment in the supplied representation. Overflow at the maximum signed value is not given a portable contract here. Neither wrapping, saturation, nor an explicit overflow error is established.

Address addition likewise requires valid resulting accesses; the function does not visibly check address overflow or object bounds.

## Dependencies

### F-d34857dd0073

Called only when a nonzero descriptor has at least one control bit above bit zero set.

Visible arguments:

1. The value read from the context’s first word before the output write.
2. The selector.

The returned value is tested as signed:

- Nonnegative: returned unchanged.
- Negative: causes diagnostic handling and a final return of −1.

Its lookup semantics, accepted selector domain, internal state changes, ownership behavior, and error-code meanings are unknown.

### F-2022b0f73f3b

Called only after F-d34857dd0073 returns a negative value.

Visible arguments:

1. The context address.
2. A fixed diagnostic-data address.
3. The selector.

Its return value is not used. Counter access follows only if the call returns normally.

A database-related diagnostic is suggested by the supplied symbolic annotation, but the exact bytes, formatting, destination, and user-visible message are not established.

## Acceptance examples

These examples are conditional contracts, not runtime observations. They assume valid memory, normally returning dependencies, and no unspecified dependency mutation unless stated.

### Absent descriptor

Given a zero descriptor, selector 27, and stored default result 81:

- No descriptor memory is read.
- The output receives 27.
- The return value is 81.
- Neither dependency is called.
- No direct counter update occurs.

### Present descriptor with only bit zero set

Given a nonzero descriptor whose control word is 1:

- The default path is selected.
- The output receives the selector, not the descriptor address.
- The stored default result is returned.

Changing that control word from 1 to 0 does not change the selected path.

### High control bit set

Given a nonzero descriptor whose control word is 2,147,483,648:

- The lookup path is selected.
- The output receives the descriptor address before F-d34857dd0073 is called.
- The high bit does not cause the control word to be treated as a negative value.

### Successful lookup returning zero

Given a control word of 2 and a lookup result of zero:

- The output receives the descriptor address.
- The function returns zero.
- The diagnostic dependency is not called.
- The function does not directly increment the counter.

Zero is a successful lookup result under the visible sign test.

### Negative lookup result

Given a lookup result of −7 and a counter of 12 after the diagnostic dependency returns:

- The diagnostic dependency is called once.
- The counter becomes 13.
- The function returns −1, not −7.

Absent dependency mutation or alias effects, the output still contains the descriptor address.

### Diagnostic changes the counter

Given an entry counter of 12, a negative lookup result, and a diagnostic dependency that changes the counter to 40:

- The function reads 40 after that dependency returns.
- It writes 41.
- It returns −1.

### Negative stored default

Given a default-path stored result of −7:

- The function returns −7 directly.
- It writes the selector to the output.
- It does not call either dependency or directly increment the counter.

### Output aliases the default result

Given that the output location is the default-result location, initially containing 81, and the selector is 27:

- The function reads 81.
- It overwrites that location with 27.
- It returns 81.

## Uncertainties

The following remain **documented_only**:

- Exact ABI, pointer representation, byte order, and alignment requirements.
- Domain meanings of the selector, descriptor control bits, context word, and returned values.
- Complete contracts and side effects of both dependencies.
- Exact diagnostic contents and presentation.
- Whether aliases are permitted by the original calling contract.
- Concurrency, atomicity, and synchronization guarantees.
- Counter-overflow behavior and behavior for invalid address arithmetic.
- Memory lifetime guarantees across dependency calls.

No synchronization or atomic operation is visible. That absence does not establish a supported concurrent-use model.

## Evidence

The contract is derived solely from the supplied static function representation and opaque dependency mapping.

Established evidence includes the branch predicate, accessed offsets, read/write order, dependency argument order, signed lookup-result test, post-diagnostic counter increment, and return-value handling.

The database-oriented purpose and diagnostic interpretation are inferred intent. Unknown dependency behavior has not been promoted into preservation, ownership, or error-handling guarantees.

No runtime tests have been performed.
