<!-- Trusted provenance: F-a37ab6b2e1db; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a37ab6b2e1db

## Purpose

F-a37ab6b2e1db conditionally invokes an operation using caller-provided context and output storage. It first checks admission conditions, writes a zero-valued 16-bit output prefix, and performs a lookup. A nonzero lookup result leads to another operation and a replacement prefix derived from the admission result. A zero lookup result instead leads to a 32-bit read from global storage, whose current value is returned.

The established contract concerns call ordering, memory accesses, integer widths, and return-value selection. The application meanings of the context, lookup, prefix, and dependent operation are unknown. Terms such as “record,” “decoder,” or “enumerator” would describe inferred intent rather than established behavior.

The second input is used only to reject negative signed 32-bit values. Its nonnegative magnitude has no further visible local effect.

## Interface

### Inputs

| Input | Visible representation | Requirements |
|---|---|---|
| Context | 32-bit address-bearing value | Must support the field accesses reached on the selected path and any dependency-specific uses. |
| Selector | Signed 32-bit integer | Must be nonnegative to pass the initial local checks. |
| Output | Pointer to a 16-bit storage unit | Must be non-null to proceed; admitted execution requires two writable bytes for local stores. Further requirements are dependency-defined. |

“Selector” is an interface label, not evidence that its value selects an element. Zero and all positive signed 32-bit values have the same visible local role.

There is no explicit context-null check. There are also no visible checks for alignment, accessible storage, output capacity, internal pointer validity, or allocation boundaries. The output-null check establishes only that the pointer is not zero.

### Return value

The visible return value is 32 bits, selected from three sources:

| Path | Return source |
|---|---|
| Initial checks fail, or eligibility comparison fails | `0xFFFFFFCE`, equivalent to −50 when interpreted as signed 32-bit |
| Lookup returns zero | A 32-bit value read from global storage **after the lookup returns zero** |
| Lookup returns nonzero and the final dependency returns normally | The exact 32-bit result of F-5ef10d079522 |

The global-storage value is not established as a constant, a cached value, or a snapshot taken before the lookup. F-f8ec0897aceb may change that storage before the read. Neither the meaning nor the stability of its contents is established.

No error-code namespace is identified. The rejection value must not be assigned a named platform error solely because of its numeric value.

### Directly accessed storage

Offsets are in bytes. Address-bearing values and address calculations use the visible 32-bit address model.

| Storage | Offset | Direct use |
|---|---:|---|
| Context | `0xF50` | Read a 32-bit value for F-f8ec0897aceb. |
| Context | `0xF60` | Read a 32-bit address-bearing value, then derive an address `0xC0` bytes beyond it. |
| Nonzero lookup result | `0x20` | Read a 32-bit address-bearing value. |
| Object reached through the preceding field | `0x28` | Read a 32-bit value for F-5ef10d079522. |
| Output | `0` | Write a 16-bit prefix. |
| Output | `2` | Supply this address to F-5ef10d079522; no direct local payload access is shown. |
| Unnamed global storage | Not specified here | Read 32 bits on the zero-lookup branch, after the lookup returns. |

These observations do not establish complete structure layouts, object sizes, field names, byte order, or ownership. In particular, the global storage’s identity and location are not needed to describe its observable role: a branch-specific, post-lookup read supplying the return value.

## Behavior

### Admission checks

A null output pointer or a negative selector causes immediate return of `0xFFFFFFCE`.

On either early-rejection path, this function performs no dependency call, no direct context-field read, no local output store, and no read of the return-value global.

When the output is non-null and the selector is nonnegative, F-c36a3e9987c8 is called with the context. Its result is treated as 64 bits divided into two unsigned 32-bit halves:

- The lower half consists of bits 0 through 31.
- The upper half consists of bits 32 through 63.

Execution proceeds only if the upper half is strictly less than the lower half under unsigned comparison. Equality and greater-than both reject with `0xFFFFFFCE`.

An eligibility rejection occurs after F-c36a3e9987c8 has run. This function performs no local output write, lookup, final-operation call, or return-global read on that path. Effects of the eligibility dependency are not undone.

This comparison is not signed, and no subtraction or count calculation is visible.

### Preliminary prefix and lookup

Once eligibility succeeds, this function writes zero to the first 16-bit output unit.

After that store, it reads the current 32-bit value at context offset `0xF50` and invokes F-f8ec0897aceb with that value and the integer one.

The store-before-read order is observable. If the output overlaps the context field, the zero store may change the value subsequently supplied to the lookup. No non-overlap condition is explicitly enforced.

The context field is also read after F-c36a3e9987c8 returns. It is not a pre-call snapshot, and changes made by that dependency can affect the lookup argument.

### Zero lookup result

If F-f8ec0897aceb returns zero, this function reads the unnamed global storage as a 32-bit value and returns that value.

The observable sequence is:

1. Eligibility succeeds.
2. The local 16-bit zero store occurs.
3. The lookup argument is read from the context.
4. F-f8ec0897aceb runs and returns zero.
5. The global return storage is read.
6. The read value is returned.

There is no call to F-5ef10d079522 and no final derived-prefix store on this branch.

The global read must not be moved conceptually before the lookup. A change to that storage made by F-f8ec0897aceb is relevant to the returned value. The function provides no visible restoration, validation, or transformation of the value read.

The preliminary zero store has occurred, but the final contents of the output prefix are not necessarily zero: the lookup may modify reachable or aliased memory. A zero-valued final prefix is conditional on the absence of such intervening modification.

### Nonzero lookup result

A nonzero lookup result is used as an address without an additional visible validity check.

F-5ef10d079522 receives four arguments:

1. The address obtained from the current 32-bit field at context offset `0xF60`, advanced by `0xC0` bytes.
2. The 32-bit value obtained by reading an address at lookup-result offset `0x20`, then reading at offset `0x28` from that address.
3. The caller’s output address advanced by two bytes.
4. An address within this invocation’s local stack storage.

Both context-based and lookup-based argument preparation occur after the lookup returns and before F-5ef10d079522 is entered. The relative ordering of those two independent preparation chains is not established by the supplied representation. Within the lookup-based chain, obtaining the nested address necessarily precedes reading the value through it.

The field at context offset `0xF60` is not cached before the lookup. Changes made by earlier dependencies can affect its value.

The stack-storage argument is separate from the caller’s output. Its extent, data type, initial contents, and intended use are unknown. No explicit initialization of that storage or later local consumption of its contents is visible.

This branch does not read the global return storage.

### Final prefix and return

After F-5ef10d079522 returns normally, this function unconditionally replaces the first 16-bit output unit with a value derived from the earlier eligibility result’s upper 32-bit half. It then returns the dependency’s 32-bit result unchanged.

The prefix consists of bits 1 through 16 of that saved upper half, shifted down to occupy bits 0 through 15. Equivalently:

**prefix = (upper half logically shifted right by one) masked with `0xFFFF`.**

The visible derivation uses a 32-bit unsigned left shift by 15 followed by a logical right shift by 16, then a 16-bit store. Bits shifted beyond the 32-bit width are discarded. The result is neither saturating nor restricted to 15 bits.

The final store does not depend on the meaning of the operation’s return value. If the dependency returns normally with a value interpreted elsewhere as failure, the prefix is still written.

## Side effects

### Direct local effects

Depending on the branch, this function directly performs:

- A 16-bit zero store to the output after eligibility succeeds.
- A subsequent 16-bit derived-value store after a nonzero lookup and normal operation return.
- The context and nested-object reads listed in the interface.
- A 32-bit read of unnamed global storage after a zero lookup result.

The global read supplies that branch’s return value. No direct write to that global is visible.

No direct allocation, release, reference-count change, or context-field write is visible. Aliasing can nevertheless cause the output stores to affect storage also reachable through another input.

### Dependency effects and interruption

The dependencies may modify globals, context fields, output storage, or other reachable state. No preservation guarantee is established.

In particular, F-f8ec0897aceb may change both:

- The global storage later read on its zero-result branch.
- Context or object state later consumed on its nonzero-result branch.

The function supplies no visible rollback or transaction boundary. If a dependency fails to return normally, subsequent local actions are not established to occur. Earlier stores and dependency effects have already occurred; this function contains no visible mechanism to reverse them.

### Ownership and lifetime

Passing an address does not establish an ownership transfer. Conversely, the absence of a direct release does not establish that dependencies retain or release nothing.

The context, caller output, lookup result, and nested object must remain usable for the accesses actually performed. Dependency-specific ownership and retention rules remain unknown.

The local stack address is available during the dependent call. Safe retention or use after this invocation ends is not established.

### Arithmetic and invalid storage

Fixed offsets and pointer advances are expressed in the visible 32-bit address model. No overflow, bounds, or alignment checks are visible. This contract does not guarantee a recoverable error or a valid wrapped access when address arithmetic produces an unusable address.

The function receives no output-capacity argument. Its direct stores need two writable bytes, while the required capacity beginning at output offset two is unknown. The contract does not infer a payload size from the prefix.

## Dependencies

| Opaque ID | Established interaction | Unresolved aspects |
|---|---|---|
| F-c36a3e9987c8 | Receives the context; supplies a 64-bit result whose unsigned 32-bit halves determine admission. | Application meaning, side effects, memory requirements, and failure behavior. |
| F-f8ec0897aceb | Receives the current context field at `0xF50` and one; returns zero or a value used as an address. | Lookup semantics, ownership, preservation, and effects on global or reachable storage. |
| F-5ef10d079522 | Receives the context-derived address, nested 32-bit value, output address plus two, and local stack address; its 32-bit result is propagated after the prefix store. | Output extent, stack-argument contract, result semantics, ownership, and other effects. |

These roles describe only the interactions established at this call site. They do not imply purity, successful completion, memory ownership, or global-state preservation.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Except where a rejection prevents access, examples assume usable storage and normally returning dependencies.

1. **Null output:** With a null output pointer, return `0xFFFFFFCE`. Invoke no dependency and perform no direct context read, output store, or return-global read.

2. **Negative selector:** With a non-null output and selector −1, return `0xFFFFFFCE` without calling a dependency or writing the output. Every negative signed 32-bit selector has the same local rejection behavior.

3. **Equal halves:** If F-c36a3e9987c8 supplies upper half seven and lower half seven, reject after that call. Do not zero the prefix, invoke the lookup, or read the return global.

4. **Unsigned boundary:** Upper half `0x80000000` and lower half one reject. Interpreting the upper half as a negative signed value would give an incorrect admission result.

5. **Zero lookup with changed global:** Let upper half be two and lower half three. If the lookup changes the return-global storage from `0x11223344` to `0xA5A5A5A5` and returns zero, then, absent another intervening modification, return `0xA5A5A5A5`. The pre-lookup value is not the required result. Do not call F-5ef10d079522.

6. **Zero lookup and output mutation:** If the lookup returns zero after replacing the output prefix, do not perform another local prefix store. If instead it leaves the prefix untouched, the earlier zero remains.

7. **Odd upper half:** With upper half three, lower half four, and a nonzero lookup result, a normal operation return must be followed by prefix value one. Return the operation result unchanged.

8. **Full 16-bit prefix:** With upper half `0x0001FFFF` and lower half `0x00020000`, the completed nonzero-lookup path produces prefix `0xFFFF`, not `0x7FFF`.

9. **Discarded high bit:** With upper half `0x00020000` and lower half `0x00020001`, that same completed path produces prefix zero.

10. **Operation-level failure result:** If F-5ef10d079522 returns normally with a value its consumer considers failure, still store the derived prefix and propagate that value unchanged.

11. **Lookup changes context:** If the lookup changes context offset `0xF60`, later argument preparation uses the changed field, assuming no further modification. It does not use a pre-lookup snapshot.

12. **Output aliases the lookup field:** If the initial prefix store overlaps the field at context offset `0xF50`, the subsequent field read observes storage after that store. The contract does not prescribe byte-level consequences without an established byte order and overlap arrangement.

## Uncertainties

The following remain **documented_only**:

- Application meanings of the inputs, eligibility halves, prefix, and dependent operation.
- Identity, value, initialization, stability, and semantic interpretation of the global return storage.
- Complete structure layouts, allocation extents, alignment requirements, and byte order.
- Payload format and required capacity beginning at output offset two.
- Size, initial contents, intended use, and retention rules for the local stack argument.
- Dependency preservation guarantees, ownership rules, side effects, and abnormal completion behavior.
- Relative ordering of the independent context-based and lookup-based argument-preparation reads.
- Defined behavior for inaccessible storage, address overflow, or concurrent modification.

The post-lookup timing and 32-bit width of the global read are established despite those uncertainties. They must not be weakened into an unspecified return source.

The selector’s otherwise unused magnitude could suggest a reserved interface component, but that is inferred intent only. It does not justify additional selector-dependent behavior.

## Evidence

This contract is based on the supplied static representation: branch conditions, call arguments, fixed-offset reads, 16-bit stores, the branch-specific global read, integer widths, shifts, and return sources.

The zero-lookup branch explicitly obtains its return value from global storage after F-f8ec0897aceb has returned. The nonzero branch instead propagates F-5ef10d079522’s result after writing the derived prefix.

Embedded comments, the prior draft, and reviewer feedback are not independent behavioral authority. Application interpretations and unavailable dependency contracts remain separated from the directly visible behavior.

No runtime tests have been performed. The acceptance examples are prospective checks, not reported execution results.
