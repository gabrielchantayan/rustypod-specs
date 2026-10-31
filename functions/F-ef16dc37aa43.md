<!-- Trusted provenance: F-ef16dc37aa43; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ef16dc37aa43

## Purpose

F-ef16dc37aa43 performs an unconditional preparation sequence, passes its two original arguments to another dependency, optionally invokes an additional dependency, and conditionally transfers a 16-bit field into storage selected by a later dependency call.

Its return value is the retained 32-bit result of the two-argument dependency. Later calls, field tests, and the conditional store do not select a replacement return value.

The established contract concerns call order, argument forwarding, memory-access order, and return-value propagation. The domain meanings of the arguments and fields are unknown. Interpreting the conditional transfer as initialization, inheritance, or context synchronization is inferred intent, not established behavior.

## Interface

### Inputs

The function accepts two arguments represented as 32-bit integers:

- **Object address:** Forwarded unchanged to two dependencies and used as the base address of a 16-bit field at byte offset `0xE6` (230).
- **Control value:** Forwarded unchanged to the two-argument dependency and tested locally for equality with zero. Every nonzero 32-bit representation satisfies the nonzero condition; the value need not be one.

The first argument is used as an address despite its integer representation. No local null check, bounds check, alignment check, or object-type check is visible.

### Output

On normal completion, the function returns the same 32-bit representation returned by dependency U-01.

No local interpretation of that result as success, failure, a Boolean, an address, or a count is established. In particular, no result-value test suppresses subsequent processing.

U-01 is a document-local opaque function identifier for the dependency whose published identity is not supplied.

### Directly accessed layout

| Address basis | Byte offset | Width | Access |
|---|---:|---:|---|
| Original object address | `0xE6` | 16 bits | Unconditional field test after the preceding calls |
| Second result from F-fdd8589eca2d, if reached | `0x80` | 16 bits | Destination-field test |
| Original object address | `0xE6` | 16 bits | Fresh read if the store path is reached |
| Third result from F-fdd8589eca2d, if reached | `0x80` | 16 bits | Conditional store |

“Second” and “third” count calls within this invocation, including the initial unconditional call. Their results need not identify the same object or storage.

Zero/nonzero tests apply to the complete 16-bit field representation. Signed interpretation does not change these tests: `0x8000` and `0xFFFF`, for example, are nonzero. The final transfer reads and stores 16 bits; it does not contribute to or replace the 32-bit return value.

Offsets are byte offsets. The supplied representation does not establish address-overflow behavior, integer-to-address conversion rules, or a defined result for address arithmetic outside the valid address range.

### Preconditions and ownership

Normal completion requires all dependencies that are reached to return normally and every reached field access to address suitable storage:

- The source field must be readable even when control is zero.
- The second getter result must support a 16-bit read at offset `0x80` when that call is reached.
- The source field must remain readable when the final reread is reached.
- The third getter result must support a 16-bit store at offset `0x80` when the store is reached.

These are path-dependent requirements, not validations performed by the function.

No allocation, release, ownership transfer, reference-count operation, or lifetime extension is directly established. No dependency is assumed to preserve object lifetimes or ownership.

## Behavior

### Preparation

The function first calls F-fdd8589eca2d with no arguments.

It then calls F-3e5212464798 with three arguments, in this order:

1. The result of that first call.
2. The original object-address argument.
3. Zero.

Both calls occur regardless of the control value and before any direct source-field access. The result of F-3e5212464798, if any, does not determine this function’s return value or a visible branch.

### Forwarding and retention of the result

Next, the function calls U-01 with the original object address followed by the original control value.

It retains U-01’s 32-bit result for return after the remaining processing. The supplied body contains no test of that result. A representation that a caller might interpret as failure therefore does not, by itself, bypass the optional call or field logic.

This retention does not guarantee eventual normal completion: a later dependency or memory access may fail to complete normally.

### Optional call

If control is nonzero, the function calls F-5c5958cfd5fe with no arguments. If control is zero, it skips that call.

This call occurs after U-01 and before the first source-field read. Its result, if any, is not used.

Consequently, the first source observation is made after all preceding calls have completed. The observed field need not equal its value at function entry.

### Source-field gate

The function reads the 16-bit field at the original object address plus `0xE6` and tests it against zero.

This read precedes the control-value condition in the field-processing gate. It therefore occurs even when control is zero.

The remaining getter calls and destination accesses require both:

- A nonzero value from this source-field read.
- Nonzero control.

If either requirement is absent, the function performs no further direct field access and returns the retained U-01 result.

### Destination-field gate

When both requirements hold, the function calls F-fdd8589eca2d a second time.

It reads the 16-bit field at offset `0x80` from this call’s result. If the field is nonzero, processing stops without a third getter call or direct store, and the function returns the retained result.

A zero observation permits the final transfer. It establishes a condition only on the storage read through this second result; it does not establish that a later getter result will identify the same storage.

### Final transfer

After observing a zero destination field, the function calls F-fdd8589eca2d a third time.

It then rereads the 16-bit source field at the original object address plus `0xE6` and stores that representation at offset `0x80` from the third result.

There is no zero test on the final store target. There is also no second nonzero test on the reread source.

Accordingly:

- The store target may differ from the destination previously tested.
- The final target may already contain a nonzero value.
- The copied value may differ from the initially tested source value.
- The copied value may be zero.
- The final target may coincide with, or partially overlap, the source field.

After the store, the function returns U-01’s retained 32-bit result. No further direct field access is expressed.

### Call counts

For an invocation that completes normally:

| Dependency | Calls |
|---|---:|
| F-fdd8589eca2d | One, two, or three |
| F-3e5212464798 | Exactly one |
| U-01 | Exactly one |
| F-5c5958cfd5fe | Zero for zero control; otherwise one |

The second getter call requires nonzero control and a nonzero source observation. The third additionally requires a zero observation at offset `0x80` from the second getter result.

## Side effects

### Direct memory effects

The body expresses at most one direct object-memory store: a 16-bit store at offset `0x80` from the final getter result.

There is no separate store expressed through the original object address. This does **not** imply that the source field is never written. The destination store may access the source field itself or other overlapping storage.

In particular, if the third getter result equals the original object address plus `0x66`, then its store address is:

`original object address + 0x66 + 0x80 = original object address + 0xE6`.

Under valid, representable address arithmetic, this is exactly the source-field address. The function then directly stores to that address. The store is still expressed through the getter result, and no separate store operation is added.

Likewise, an offset relationship producing partial overlap can make the destination store affect part of the source field or nearby storage. There is no general guarantee that storage neighboring the source remains unchanged.

On the longest path, direct field accesses and intervening calls occur in this order:

1. Read the source field.
2. Call F-fdd8589eca2d for the second time.
3. Read the tested destination field.
4. Call F-fdd8589eca2d for the third time.
5. Reread the source field.
6. Store the reread 16-bit representation to the final target.

The source reread precedes the store, including when their addressed storage overlaps. The static representation does not establish processor-level atomicity, memory barriers, or synchronization.

### Dependency effects and aliasing

Dependencies may change globals, object contents, ownership, or lifetimes. None is established to return a stable address or preserve storage used by later accesses.

The original object, the objects returned by repeated calls, and the fields addressed through them are not established to be disjoint. Equal field addresses do not require equal base addresses.

Any mutation completed before the final source read can affect the transferred representation. The earlier nonzero source observation and zero destination observation are not revalidated at the store.

### Errors and abnormal completion

No local error reporting, exception handling, rollback, retry, or recovery is visible.

If a dependency does not return normally, execution of the remaining sequence is not established. Invalid memory or unsupported accesses have no defined recovery behavior in the supplied evidence.

The function does not undo earlier dependency effects if later processing fails. Returning U-01’s result describes normal completion only.

## Dependencies

| Opaque identifier | Established interaction | Unresolved behavior |
|---|---|---|
| F-fdd8589eca2d | Called without arguments once unconditionally and up to twice more; results supply one preparation argument and later field bases | Result meaning, stability, ownership, lifetime, global changes, and other side effects |
| F-3e5212464798 | Receives the initial getter result, original object address, and zero | Purpose, return meaning, memory effects, ownership effects, and failure behavior |
| U-01 | Receives both original arguments; provides the retained 32-bit result | Published identity, internal behavior, result meaning, and side effects |
| F-5c5958cfd5fe | Called without arguments exactly when control is nonzero, after U-01 | Purpose, return meaning, memory effects, ownership effects, and failure behavior |

Caching getter results would require a stability guarantee absent from the evidence. Reusing the initially tested source value would likewise require a preservation guarantee not supplied here.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Each assumes normal dependency returns and suitable memory for all reached accesses.

### Zero control still reads the source

With zero control, the preparation calls and U-01 execute. F-5c5958cfd5fe does not execute.

The function nevertheless reads the source field. It makes no second getter call and performs no direct destination-field access. It returns U-01’s 32-bit result.

An implementation that skips the source read solely because control is zero would not preserve the expressed access sequence.

### Nonzero control with a zero source

If control is nonzero and the source field is zero when first read, F-5c5958cfd5fe executes before that read.

The getter is called only once overall. No direct destination-field access or store occurs, and U-01’s result is returned.

### Nonzero tested destination prevents the store

If control and the source observation are nonzero, but the second getter result has `0x8000` at offset `0x80`, the destination condition fails.

The getter is called twice overall. There is no third call and no direct store. The high bit does not make the field equivalent to zero.

### Stable storage permits a straightforward copy

Suppose control is nonzero, both source reads produce `0x1234`, and the second and third getter calls identify the same valid destination field, initially zero.

The getter is called three times overall. The destination receives `0x1234`, and the return value remains U-01’s result.

### A different final target may be overwritten

Suppose the second getter result identifies a zero destination field, but the third identifies a different field containing `0x5678`.

The function stores the freshly read source representation into the third result’s field without checking `0x5678`. The second field’s zero observation does not protect the final target from overwrite.

### The source may become zero before copying

Suppose the initial source observation is nonzero and the tested destination is zero, but the third getter call changes the source field to zero.

The final source reread obtains zero, and the function stores zero. It does not reuse the earlier nonzero representation.

### The store may address the source field itself

Suppose control is nonzero, the initial source observation is nonzero, and the second getter result supplies a zero tested field. Suppose the third getter result equals the original object address plus `0x66`, with valid address arithmetic and writable source storage.

The final destination address equals the source address at offset `0xE6`. The function rereads that source field and then directly stores the read representation back to the same address.

Even if the stored bits equal the existing bits, this remains a direct store to source-field storage. The absence of a separately expressed source-based store does not eliminate it.

### All return bits are retained

If U-01 returns `0xFFFFFFFF` and later processing completes normally, this function returns `0xFFFFFFFF`.

That representation neither becomes a Boolean nor independently suppresses the remaining calls and accesses.

## Uncertainties

The following are **documented_only**:

- Domain meanings of both arguments and accessed fields.
- Whether getter results designate contexts, global objects, or another kind of storage.
- Dependency effects on globals, object contents, ownership, and lifetimes.
- U-01’s published identity and the meaning or signedness of its result.
- Exact ABI details beyond the represented argument and result widths.
- Endianness, alignment requirements, address-overflow behavior, and invalid-access behavior.
- Concurrency, asynchronous interruption, and processor-level access guarantees.
- Concrete byte outcomes for partially overlapping fields without an established byte order and memory model.

A possible intent is to populate an empty field from another object when control is active. That interpretation is inferential. It must not replace the established repeated calls, fresh source read, or possibility that the final store aliases the source.

## Evidence

This contract is based solely on the supplied static function representation and dependency mapping. The prior draft and feedback are not independent behavioral evidence.

The supplied body establishes the call ordering, unchanged argument forwarding, retained 32-bit return value, source-first conditional evaluation, 16-bit accesses, byte offsets, repeated getter calls, fresh source reread, and single conditional destination-based store.

The exact-alias case follows from the accessed offsets: `0x66 + 0x80 = 0xE6`. No visible identity or non-overlap check excludes that relationship.

No dependency bodies, independent ABI specification, or runtime observations were supplied. No runtime tests have been performed. The acceptance examples are conditional static deductions, not reports of observed execution.
