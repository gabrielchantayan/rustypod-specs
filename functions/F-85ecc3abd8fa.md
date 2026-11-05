<!-- Trusted provenance: F-85ecc3abd8fa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-85ecc3abd8fa

## Purpose

F-85ecc3abd8fa is a conditional dispatcher over a context and an address-designated object. It tries an optional preliminary handler, selects one of two context-provided callbacks, and may then invoke a final context-provided callback.

The function uses a **nonzero result as a terminal result**: any invoked handler that returns nonzero determines the function’s return value immediately. A zero result permits the remaining dispatch stages to proceed. If no invoked handler returns nonzero, the function returns zero.

This dispatch structure is established by static evidence. The application meaning of the object tags, discriminator values, callbacks, and returned integers is unknown. In particular, the evidence does not establish that nonzero means success or that zero means failure.

## Interface

### Inputs

The function receives two arguments:

1. **Context reference.** A pointer to storage containing a dispatch-table reference at byte offset 0. A later conditional stage may also read a 32-bit value at byte offset 80.
2. **Object address.** An integer-valued argument used as the address of an object. The function directly reads a 32-bit tag at byte offset 4. Depending on that tag, it may read another 32-bit value at byte offset 8.

The same two argument values are passed, in the same order, to F-9541095f511f and to every indirect callback invoked by this function. F-468ee00d626a receives only the object address.

These are address-bearing inputs, not copies of the objects they designate. Passing the same argument values after a call does not imply that the designated storage remains unchanged.

### Relevant storage layout

Only the following portions of the layouts are established:

| Storage | Byte offset | Observed use |
|---|---:|---|
| Context | 0 | Reference to a dispatch table |
| Context | 80, hexadecimal 0x50 | 32-bit comparison value used by the final stage |
| Object | 4 | 32-bit tag, read at several distinct stages |
| Object | 8 | Either a nested-object address or a 32-bit comparison value, depending on the current tag |
| Nested object | 4 | 32-bit discriminator used to select the primary callback |
| Dispatch table | 224, hexadecimal 0xE0 | Final callback entry |
| Dispatch table | 268, hexadecimal 0x10C | Primary callback entry for the distinguished nested-object case |
| Dispatch table | 272, hexadecimal 0x110 | Primary callback entry for all other cases |

Offsets are in bytes. Unlisted fields have no established role in this function.

The supplied representation uses 32-bit integer accesses. It also represents some addresses through integer-valued storage. That supports describing the observed integer fields as 32-bit quantities, but does not independently establish the complete platform ABI, pointer representation, endianness, or alignment rules.

### Output

The result is an integer:

- The exact nonzero value returned by the first invoked handler to return nonzero; or
- Zero if dispatch reaches the end without such a result.

The function does not normalize nonzero values to one. Negative and positive nonzero values receive the same terminal treatment.

### Preconditions

Every memory location actually reached by the selected path must be readable, and every selected callback entry must designate a callable target with a compatible calling convention.

Required storage validity is conditional:

- The object’s tag field is needed on every normally completing path.
- The object’s field at offset 8 is needed during primary selection only when the freshly read tag is 22.
- That field must designate suitable nested storage when primary selection follows the nested-object path.
- The context comparison field at offset 80 is needed during final-stage evaluation only when the freshly read object tag is 15.
- Dispatch-table storage and callback entries must be valid when their respective calls are reached.

There are no visible null, bounds, lifetime, or callback-validity checks.

## Behavior

### 1. Optional preliminary handling

The function first reads the object’s 32-bit tag at offset 4.

If that value is not 13, neither preliminary dependency is called, and processing proceeds to primary callback selection.

If the value is 13:

1. F-468ee00d626a is called with the object address.
2. Its integer result is compared with 111, hexadecimal 0x6F.
3. Only if that result equals 111 is F-9541095f511f called with the context reference and object address.
4. If F-9541095f511f returns nonzero, that exact value is returned immediately.
5. Otherwise, processing continues to primary callback selection.

The original tag condition is not rechecked between these two dependencies. Therefore, if F-468ee00d626a changes the object’s tag but returns 111, F-9541095f511f is still invoked.

A result from F-468ee00d626a other than 111 is not propagated, even if it is nonzero. That dependency’s result serves only as an equality discriminator.

### 2. Primary callback selection

The function reads the object’s tag at offset 4 again. It does not reuse the initial tag value.

If this fresh tag is 22, hexadecimal 0x16:

1. The function reads the 32-bit field at object offset 8 and uses it as a nested-object address.
2. It reads the nested object’s 32-bit discriminator at offset 4.
3. If that discriminator equals 42, hexadecimal 0x2A, the selected callback is the entry at dispatch-table offset 268.
4. Otherwise, the selected callback is the entry at dispatch-table offset 272.

If the fresh tag is anything other than 22, the selected callback is the entry at dispatch-table offset 272. The object’s offset-8 field is not read for primary selection on that path.

After determining which entry to use, the function reads the context’s dispatch-table reference and retrieves the selected callback entry. It invokes that callback with the context reference and object address.

If the primary callback returns nonzero, the function immediately returns that exact result. The final stage is not evaluated.

If the primary callback returns zero, processing continues.

Thus, every normally executing invocation that does not terminate in the preliminary handler invokes exactly one primary callback. The two primary entries are alternatives, not sequential fallback handlers.

### 3. Conditional final handling

After a zero primary result, the function reads the object’s tag at offset 4 again.

If the tag is not 15, hexadecimal 0x0F, the function returns zero without invoking the final callback.

If the tag is 15:

1. The function reads the object’s 32-bit field at offset 8.
2. It then reads the context’s 32-bit comparison field at offset 80.
3. It compares those two values for equality.
4. If they differ, the function returns zero.
5. If they match, the function reads the context’s dispatch-table reference again, obtains the callback at table offset 224, and invokes it with the context reference and object address.

A nonzero final-callback result is returned unchanged. A zero final-callback result produces a final return value of zero.

The final comparison is a comparison of the two stored 32-bit values. The function does not dereference the object’s offset-8 value in this stage.

### Mutation-sensitive sequencing

The tag is independently read at the initial check, primary selection, and final-stage check. Earlier dependency or callback activity can therefore change which later branch is taken.

Similarly, the dispatch-table reference is loaded when each indirect call is prepared. The final callback need not come from the same table used for the primary callback if intervening activity changes the context’s table reference.

These observations do not assume that dependencies mutate storage. They establish that the function uses fresh reads rather than enforcing a snapshot across stages.

## Side effects

The function contains no explicit stores to the context, object, nested object, or global storage. Its directly visible operations are reads, calls, comparisons, and returns.

Its calls may nevertheless produce externally observable effects. No available dependency contract establishes that any dependency or callback:

- Preserves the context, object, dispatch table, or global state;
- Leaves unrelated memory unchanged;
- Avoids allocation or deallocation;
- Retains, transfers, or releases ownership;
- Returns normally.

Consequently, this function provides no established ownership-transfer or release operation of its own. It also provides no established protection against a callee invalidating storage that later stages would read.

For normally returning dependencies, the maximum dispatch sequence contains four calls:

1. F-468ee00d626a;
2. F-9541095f511f;
3. One primary callback;
4. The final callback.

Each is invoked at most once by this function. There is no visible retry or loop.

No local recovery, rollback, or error translation is present. Invalid memory access, an invalid callback target, or abnormal behavior inside a dependency has no specified recovery path here.

## Dependencies

| Dependency | Invocation condition | Arguments | Result use |
|---|---|---|---|
| F-468ee00d626a | Initial object tag equals 13 | Object address | Equality test against 111 only |
| F-9541095f511f | Initial tag equals 13 and F-468ee00d626a returns 111 | Context reference, object address | Nonzero returned unchanged; zero continues |
| Callback at table offset 268 | Primary-stage tag equals 22 and nested discriminator equals 42 | Context reference, object address | Nonzero returned unchanged; zero continues |
| Callback at table offset 272 | Every other primary-selection case | Context reference, object address | Nonzero returned unchanged; zero continues |
| Callback at table offset 224 | Primary result is zero, final-stage tag equals 15, and comparison fields match | Context reference, object address | Nonzero returned unchanged; zero yields zero |

The indirect callbacks are identified by their table positions because their function identities are not supplied. Their effects and application-level meanings remain unspecified.

## Acceptance examples

These are conditional expectations derived from static evidence, not reports of executed tests. Each assumes valid reached storage and normally returning callees.

1. **Preliminary handler terminates dispatch.**  
   The initial tag is 13, F-468ee00d626a returns 111, and F-9541095f511f returns −7. The function returns −7. Neither primary callback nor the final callback is invoked.

2. **A nonmatching preliminary discriminator does not terminate dispatch.**  
   The initial tag is 13 and F-468ee00d626a returns 112. F-9541095f511f is not called. Primary selection proceeds; 112 is not returned merely because it is nonzero.

3. **The preliminary tag is not revalidated before the preliminary handler.**  
   The initial tag is 13. F-468ee00d626a changes the tag to another value and returns 111. F-9541095f511f is still called. If it returns zero, primary selection uses a fresh tag read.

4. **The distinguished nested case selects offset 268.**  
   At primary selection, the tag is 22 and the nested discriminator is 42. Only the primary entry at table offset 268 is invoked. If it returns 9, the function returns 9 without evaluating the final-stage condition.

5. **Other nested discriminators select offset 272.**  
   At primary selection, the tag is 22 but the nested discriminator is not 42. The callback at table offset 272 is selected. A zero nested-object address is not specially handled; it must still support the required access for this path to execute normally.

6. **A nonnested tag avoids the nested read.**  
   At primary selection, the tag is not 22. The callback at offset 272 is selected without reading the object’s offset-8 field for that selection.

7. **A primary callback can make the final stage eligible.**  
   The primary callback changes the object tag to 15 and returns zero. The function then compares the current object field at offset 8 with the current context field at offset 80. Equality causes the final callback to run.

8. **A primary callback can make the final stage ineligible.**  
   The object previously had tag 15, but the primary callback changes it to another value and returns zero. The function returns zero without reading either comparison field for the final-stage match.

9. **A matching final comparison is insufficient after a terminal primary result.**  
   The primary callback returns 4. Even if the current tag is 15 and both comparison fields match, the result is 4 and the final callback is not invoked.

10. **The final callback uses the current table reference.**  
    The primary callback changes the context’s table reference and returns zero. If final eligibility holds, the entry at offset 224 is obtained from the newly referenced table.

## Uncertainties

### Established static evidence

The branch conditions, byte offsets, call ordering, repeated reads, result propagation, and absence of explicit stores are directly supported by the supplied function representation.

### Inferred intent

The structure suggests a preliminary specialized handler, a general two-way dispatch, and a conditional final handler. These descriptions summarize control flow; they do not establish application semantics or success/failure conventions.

### documented_only unknowns

- Exact platform ABI, pointer width, address-conversion behavior, endianness, and alignment requirements.
- Complete layouts, allocation sizes, and semantic types of the referenced structures.
- The meanings of tags 13, 22, and 15, and discriminator values 111 and 42.
- Callback identities beyond their dispatch-table positions.
- Dependency effects on memory, ownership, globals, and control flow.
- Whether callers guarantee stable storage or permit concurrent mutation.
- Whether any external mechanism handles faults or abnormal dependency exits.

The function contains no visible arithmetic on field values that could introduce signed overflow or truncation. Relevant arithmetic consists of fixed-offset addressing, including the context field at byte offset 80. Address-wrap behavior is not established.

## Evidence

This contract is based solely on static inspection of the supplied representation of F-85ecc3abd8fa and the supplied opaque dependency mapping.

No runtime tests have been performed. The acceptance examples are derived behavioral conditions, not observed execution results. No dependency implementation or independent ABI evidence was supplied.
