<!-- Trusted provenance: F-ec777f35bfd3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ec777f35bfd3

## Purpose

F-ec777f35bfd3 dispatches an operation to a context-selected override or executes a built-in path.

The built-in path supports two forms:

- **Direct form:** operation selector `0x72`, requiring the third argument to equal `0x24`.
- **Descriptor-mediated form:** every other selector, resolved through a descriptor dependency and processed through two transformation calls with an intervening allocation.

Both built-in forms obtain a context-derived bound, compare it with a size-like value, and conditionally invoke a common final operation. A signed-positive result from that operation is written to the caller’s result location and produces return value `1`. A zero or negative result produces return value `0`.

The override receives all six original arguments. Its return value is forwarded without Boolean normalization.

These are established static control-flow properties. The application-level meanings of the operation, descriptor, payload, and result are not established.

## Interface

The recovered interface has six arguments and a 32-bit unsigned return value. Integer and address-bearing words in the supplied representation are 32 bits.

| Position | Observable role |
|---|---|
| First | A 32-bit operation selector. `0x72` selects the direct built-in form. Other values are supplied to F-83fcd2d25cd8. |
| Second | A 32-bit payload value. Passed directly to the final operation in the direct form; placed in a temporary input area in the descriptor-mediated form. |
| Third | A 32-bit value. Must equal `0x24` in the direct form; otherwise placed in the temporary input area. |
| Fourth | An opaque 32-bit value saved for the common final operation. |
| Fifth | An address of a writable 32-bit result location, conditionally written by the built-in path. |
| Sixth | A context address used for dispatch-field reads and passed to context-related dependencies. |

The operation-selector tests use exact equality. Signed interpretation matters for the bound comparison and the final result’s positivity test.

### Preconditions for normal completion

The locally visible accesses require:

- A readable 32-bit context flags field at byte offset `0x3c`.
- When flags bit `0x40` is set, a readable address-bearing context field at offset `0x08` and a readable callback field at offset `0x2c` in the referenced table.
- A callable six-argument override whenever the selected callback is non-null.
- For a non-null resolved descriptor, a readable 32-bit field at descriptor offset `0x0c`.
- A writable 32-bit result location if the built-in final operation returns a signed-positive value.

The table address is not checked for null before its callback field is read. No initial validation protects the context access.

These conditions describe this function’s visible accesses, not the complete requirements of its dependencies. An override receives the result address even when the built-in path would not dereference it. Dependencies may impose additional validity, lifetime, alignment, or aliasing requirements.

## Behavior

### Override selection and precedence

The function first reads the context flags word at offset `0x3c` and tests bit `0x40`.

If the bit is clear, it proceeds to built-in processing without reading the dispatch-table field.

If the bit is set, it reads the table address from context offset `0x08`, then reads the callback address from table offset `0x2c`.

- A non-null callback is invoked with all six original arguments.
- Its 32-bit return value becomes this function’s return value.
- A null callback permits built-in processing to continue.

Override dispatch precedes direct-form validation. Consequently, an active override receives even a direct-form request whose third argument would be rejected by the built-in path.

After invoking the override, this function performs no built-in descriptor lookup, bound query, allocation, transformation, final operation, result write, or cleanup.

### Direct built-in form

For operation selector `0x72`, the third argument must equal `0x24`, decimal `36`.

A different third argument causes a call to F-4fc89ec41a6d with arguments `4` and `0x75`, followed by return value `0` if that dependency returns normally. No context-bound query occurs on this rejection path.

For an accepted direct request:

- The size-like value is fixed at `36`.
- The second argument supplies the payload value for the final operation.
- No descriptor lookup, transformation call, or temporary allocation occurs.

The function does not locally establish whether the direct payload denotes an address, handle, or another 32-bit representation.

### Descriptor acceptance

For every operation selector other than `0x72`, the function calls F-83fcd2d25cd8 with the selector.

It rejects the request through the common diagnostic path if:

- The returned descriptor address is null; or
- The descriptor’s 32-bit field at byte offset `0x0c` equals zero.

A null descriptor is rejected without reading its offset-`0x0c` field. No other descriptor field is interpreted locally.

### Temporary records and first transformation

After accepting the descriptor, the function initializes temporary storage and invokes F-a08db0d7cd3f with a transformation argument block and a null second argument.

The recovered arrangement is:

| Region | Initially established contents |
|---|---|
| Transformation argument block | At offset `0`, a reference to the descriptor record; at offset `4`, a reference to the input area. |
| Descriptor record | At offset `0`, the resolved descriptor address; at offset `4`, a reference to the metadata pair. |
| Input area | At offset `0`, the third argument; at offset `8`, the second argument. The intervening word at offset `4` is not visibly initialized. |
| Three-word destination region | Its word at offset `4` is initialized to `5`; its word at offset `8` is initialized to `0`. Its first word is not initialized by this setup. |

The **metadata pair is the same storage as the last two words of the three-word destination region**. It is not an independent record whose contents are later copied into that region.

At entry to the first transformation, the descriptor record exposes the metadata pair initialized to `5` and `0`. The first transformation can therefore access those words through the supplied references. Its mutation and retention behavior is unknown.

The first transformation’s 32-bit return becomes the size-like value. There is no separate check for zero, negativity, or an error sentinel before the bound comparison.

The temporary argument block is not rebuilt between transformation calls. Its referenced records are likewise not restored to their initial contents. Any permitted mutations made through the exposed references can therefore affect the later call.

### Context-bound check and arithmetic

After establishing the size-like value, the function calls F-d7cb69444a05 with the context.

Its 32-bit return is used as the context bound. Under the recovered signed-integer interpretation, the request is rejected when:

**context bound minus `11` is less than the size-like value.**

Rejection invokes F-4fc89ec41a6d with `4` and `0x75`, then returns `0` if the diagnostic dependency returns normally.

Equality passes. With ordinary non-overflowing arithmetic, the direct form therefore requires a bound of at least `47`.

There is no independent nonnegativity requirement for either value. A negative first-transformation result is governed by the same signed comparison rather than being rejected solely for being negative.

The subtraction and the later addition of `1` are 32-bit arithmetic expressions. Behavior at signed-overflow boundaries is documented_only. The supplied representation does not establish whether the applicable contract uses machine wrapping, excludes those inputs, or requires another interpretation. Acceptance examples below avoid overflow.

The bound is queried once. No later dependency call causes a local refresh.

### Allocation and second transformation

Only the descriptor-mediated form allocates storage.

After the bound check passes, F-43da6b23bd44 receives three arguments:

1. Context bound plus `1`.
2. Zero.
3. Zero.

The allocation request is based on the bound, not the size-like value.

A null return causes the common diagnostic call and return value `0`. No cleanup dependency is called for that null result.

For a non-null return, the function assigns the allocated address to the first word of the existing three-word destination region. It then invokes F-a08db0d7cd3f with:

- The same temporary argument-block address used for the first transformation.
- The address of the three-word destination region.

Immediately before this call, the destination region has:

| Byte offset | Established contents |
|---|---|
| `0` | The allocated storage address, newly assigned by this function. |
| `4` | The current contents of the first shared metadata word, originally initialized to `5`. |
| `8` | The current contents of the second shared metadata word, originally initialized to `0`. |

**The trailing words are not reinitialized before the second transformation.** They need not still contain `5` and `0`. The first transformation may have modified them directly. Dependencies executing between the transformation calls may also affect them if references were retained and made available to those dependencies.

The explicit first-word assignment does not reset the metadata pair, descriptor record, input area, or argument-block references.

The second transformation’s return value is ignored. There is no local check for transformation failure, an output length, or agreement with the earlier size-like value.

The originally returned allocation address remains the payload value used for the final operation and the address used for cleanup. The function does not reread the destination region’s first word to select those later arguments.

### Common final operation

Both built-in forms call F-0ea5053a881c with five arguments:

1. The size-like value established before the bound query.
2. The direct payload value, or the allocation address for the descriptor-mediated form.
3. The saved fourth argument.
4. The context.
5. The 32-bit constant `1`.

The signed 32-bit return controls the local output write:

- If greater than zero, that exact value is written to the fifth argument’s result location.
- If zero or negative, no local write is made through that argument.

A positive return is the sole local success criterion. The result is not compared with the size-like value.

A nonpositive final result does not itself trigger the diagnostic dependency.

### Completion and cleanup

The direct form returns `1` for a positive final result and `0` otherwise, with no cleanup calls.

The descriptor-mediated form performs these calls after the conditional result write:

1. F-80308ee0e93f with the original allocation address and context bound plus `1`.
2. F-5990a26eaaa3 with the original allocation address.

Both calls occur regardless of whether the final result is positive, zero, or negative, provided execution reaches them and each dependency returns normally.

Their return values do not determine the built-in return value. After both return, this function reports whether the final operation’s result was positive.

## Side effects

### Established local effects and ordering

The function directly:

- Reads context dispatch fields.
- Reads the accepted descriptor’s field at offset `0x0c`.
- Initializes temporary records.
- Later assigns the allocation address to the destination region’s first word.
- Conditionally writes one 32-bit caller-result word.

No explicit context-field or global-object write appears in the supplied representation.

The principal observable order is:

1. Override selection.
2. Built-in validation and, where applicable, descriptor resolution.
3. Metadata/input initialization and first transformation.
4. Context-bound query and comparison.
5. Descriptor-mediated allocation.
6. Destination first-word assignment and second transformation.
7. Final operation.
8. Conditional caller-result write.
9. First cleanup call, then second cleanup call.

The shared metadata words are initialized before the first transformation and are not locally rewritten afterward. This is a storage-identity property, not merely a general possibility that dependencies have side effects.

### Ownership and lifetime

The allocation and paired post-operation calls suggest temporary-storage management. That is inferred intent, not proof of which dependency releases memory or whether either cleanup call overwrites it.

Other ownership facts remain unresolved:

- The descriptor is not locally released.
- The direct payload is not locally released.
- Temporary record addresses are exposed to the transformation dependency.
- Whether those references are retained is unknown.
- Whether the final operation retains the payload address is unknown.
- Aliasing restrictions among context, result location, direct payload, and dependency-managed storage are not established.

“No local result write” does not guarantee an unchanged result location across dependency calls. Similarly, a positive-result write occurs before cleanup and is not guaranteed to remain undisturbed if cleanup has relevant aliases or other access.

No dependency is assumed to preserve globals, context state, or exposed temporary storage.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-83fcd2d25cd8 | Receives a non-`0x72` selector and returns a descriptor address. |
| F-a08db0d7cd3f | Receives the temporary argument block first with a null destination, then with the shared three-word destination region. The first return supplies the size-like value; the second return is ignored. |
| F-d7cb69444a05 | Receives the context and returns the bound used in comparison, allocation sizing, and sized cleanup. |
| F-43da6b23bd44 | Receives bound plus `1`, zero, and zero; its non-null return supplies the temporary payload address. |
| F-0ea5053a881c | Receives the five final-operation arguments and returns the signed result governing local success. |
| F-80308ee0e93f | Receives the original allocation address and bound plus `1` after the final operation. |
| F-5990a26eaaa3 | Receives the original allocation address after F-80308ee0e93f. |
| F-4fc89ec41a6d | Receives `4` and `0x75` for explicitly handled validation, bound, and null-allocation failures. |

The dynamic override has no supplied opaque identity. Its known interface is the six-argument call and direct forwarding of its 32-bit return.

## Acceptance examples

These are conditional static examples, not executed tests. They assume valid referenced memory, normal dependency returns, and no arithmetic overflow.

1. **Override precedence.** With an active non-null override, selector `0x72` and third argument `35` reach the override rather than built-in rejection. An override return of `7` produces return value `7`.

2. **Null callback fallback.** With dispatch enabled, a readable table, and a null callback field, built-in processing continues.

3. **Direct validation failure.** Without an active override, selector `0x72` and third argument `35` cause diagnostic arguments `4` and `0x75`, then return `0`. No bound query or final operation occurs.

4. **Direct bound boundary.** For selector `0x72` and third argument `36`, bound `46` rejects the request; bound `47` passes.

5. **Positive direct result.** With a passing bound and final result `12`, the function writes `12`, returns `1`, and calls neither cleanup dependency.

6. **Nonpositive final result.** A final result of `0` or `-1` causes no local result write and produces built-in return `0`. Descriptor-mediated cleanup still occurs.

7. **Descriptor rejection.** A null descriptor, or a zero field at descriptor offset `0x0c`, causes diagnostic handling before the first transformation or bound query.

8. **Descriptor-mediated equality.** First-transformation result `20` and bound `31` pass the comparison. The allocation request is `32`, not `21`.

9. **Unmodified shared metadata.** If no dependency changes the metadata pair, the second transformation receives a destination whose trailing words remain `5` and `0`.

10. **Modified shared metadata.** If the first transformation changes those words to `9` and `2`, and they remain unchanged afterward, the second transformation receives trailing words `9` and `2`. The function does not restore `5` and `0`.

11. **Ignored second return.** Whatever value the second transformation returns normally, the final operation is still invoked with the first transformation’s size-like return and the original allocation address.

12. **Output before cleanup.** A descriptor-mediated final result of `9` causes a write of `9` before either cleanup call. Both cleanup calls then occur before return value `1`.

## Uncertainties

The following remain documented_only:

- Application-level meanings of the selector, payload, fourth argument, and result.
- Descriptor-field semantics beyond the nonzero acceptance test.
- The complete transformation ABI and interpretation of each record.
- The meaning and possible use of the uninitialized input word.
- Whether the first transformation is a pure size query.
- Actual mutations or retention of temporary references by either transformation.
- The trailing destination values at the second call unless dependency behavior establishes them.
- Transformation error conventions.
- Arithmetic behavior at signed-overflow boundaries.
- Allocation failure behavior beyond a null return.
- Ownership transfer, release, overwrite, and allocator effects of cleanup.
- Dependency effects on globals, context, caller output, and exposed storage.
- Whether diagnostic handling returns, terminates execution, or transfers control elsewhere.

## Evidence

This contract is based on the supplied static function representation and opaque dependency mapping.

Established claims concern visible field offsets, 32-bit values, comparisons, shared storage, explicit assignments, call arguments, ordering, conditional writes, and return handling. In particular, the metadata reference targets the destination region’s trailing two words; only the destination’s first word is assigned between allocation and the second transformation.

“Allocation,” “transformation,” and “cleanup” describe observed roles. Stronger semantic claims remain inferred intent or documented_only.

The supplied representation carries a type-propagation warning. Recovered types and temporary layouts therefore do not establish a complete external ABI or resolve overflow behavior.

No runtime tests have been performed.
