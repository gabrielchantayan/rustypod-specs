<!-- Trusted provenance: F-3bb47bd29b7b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3bb47bd29b7b

## Purpose

F-3bb47bd29b7b returns either a stored four-byte value or the four-byte value one, according to a selector in caller-addressed storage.

A zero selector makes the stored value authoritative. Any nonzero selector overrides that value with one. The stored value is not restricted to Boolean values: when selected, its full 32-bit pattern is preserved.

The application meaning of this operation is unresolved. The evidence does not establish a validity check, permission decision, success indication, or Boolean query. Those interpretations must not narrow the permitted stored values or assign an error meaning to either zero or one.

## Interface

### Input and addressed storage

There is one signed-integer input, used as the base for locating storage. Its exact width and the execution environment’s address width are not established by the supplied evidence. It must not silently be redefined as an unrestricted pointer or unsigned address.

The relevant storage layout is:

| Role | Byte displacement from base | Established representation | Access condition |
|---|---:|---|---|
| Selector | 120 | Signed integer of the same type category as the input; exact width unresolved | Every ordinary completion |
| Stored result | 128 | Four-byte value, preserving all 32 bits | Only when the selector observation is zero |

These displacements identify the starts of the accessed values. They do not establish an object’s complete layout, allocation size, intervening fields, or padding. In particular, the eight-byte distance between the two starts does not determine the selector’s read width.

The stored result’s signedness and application type are unknown. Its treatment here is as a complete four-byte value, not as a serialized integer with an independently specified byte order.

### Return value

Ordinary completion returns four bytes:

- A zero selector requires the stored result, without normalization.
- A nonzero selector requires numeric one in the four-byte return representation.

There is no separate established error result, output buffer, status channel, or ownership-bearing return. Returning one does not identify which alternative was selected: the stored value itself may also be one.

### Preconditions

The input must support valid address formation and the accesses required by the selected behavior.

The selector location must be readable over its full, environment-defined extent. If its observed value is zero, the stored-result location must additionally permit a four-byte read. Required storage must have suitable lifetime, alignment, permissions, and representation.

These conditions are path-dependent. A nonzero selector does not require the stored-result location to be readable, nor does it require address formation used exclusively for that conditional access to succeed.

No validation of these conditions is established.

## Behavior

### Selection and value preservation

Zero is the sole selector value that chooses the stored result. Every representable nonzero selector chooses one, regardless of sign or magnitude. The decision is not a test of a particular selector bit, positivity, or equality to one.

When the stored result is selected, all its bits contribute to the returned value. There is no established Boolean conversion, masking, saturation, range restriction, or rejection of high-bit patterns. A stored zero remains zero; a stored value with only its highest bit set retains that pattern; a stored value with all 32 bits set retains all 32 bits.

When the selector is nonzero, the stored result has no role in determining the return value. Altering that storage alone must not change the required result.

### Observable access requirements

The established logical access sequence consists of a selector observation followed, only if that observation is zero, by a stored-result observation. The nonzero case requires no logical read of the stored result.

This conditional access is part of the behavior, not merely a performance preference. Unconditionally reading both locations would introduce a storage-access requirement absent from the evidence.

No selector reread, retry, or reconciliation is established. If the environment permits a selector change after its observation, the function has no established obligation to reconsider its selection. This does not authorize an otherwise invalid concurrent modification.

When both locations are observed, the function does not independently guarantee that the observations belong to one coherent snapshot. Any such guarantee must come from the surrounding environment or protocol.

These are logical access requirements. They do not specify machine-instruction counts, bus transactions, speculative activity, cache effects, atomicity, or special-memory ordering.

### Address formation and arithmetic

Both storage locations use fixed byte displacements from the input. Neither observed value changes those displacements.

The address expressions use signed-integer addition. The evidence does not establish modulo arithmetic, an unsigned address domain, or permission to wrap around the address space. Exact arithmetic bounds depend on the unresolved integer width and applicable execution semantics.

The displacement of 120 is required for the selector access. The displacement of 128 is required only when the selector observation is zero. An integration must therefore assess address validity according to the accesses actually required, rather than imposing both accesses on every invocation.

An arithmetic result that resembles a machine address is not, by itself, evidence that overflow or integer-to-address conversion is valid.

### Invalid conditions and errors

No explicit error detection, recovery, alternate data source, or fallback value is established.

An unreadable selector cannot be bypassed by using a readable stored result. Likewise, a zero selector does not permit substitution of one when the required stored-result read is invalid.

Consequences of invalid arithmetic, invalid address interpretation, inaccessible memory, expired lifetime, or unsuitable alignment are **documented_only**. The evidence does not select a fault, termination, language-level invalid behavior, or any particular recovery mechanism.

Zero and one remain ordinary possible results. Neither is reserved here as an error indicator.

## Side effects

The established storage effects are reads. There are no explicit writes to caller storage or global state, called functions, allocations, releases, or external-resource operations.

No retention of the supplied base or transfer of ownership is established. The caller must arrange validity for the required observations, but the function does not identify the storage’s allocator, owner, eventual releaser, or broader lifetime protocol.

The absence of explicit writes does not guarantee that every possible read is externally inert. Device-backed or other special storage may associate effects with reads. Those effects are **documented_only** and depend on the storage environment.

No locking, synchronization, memory barrier, or atomic-access guarantee is established. Read-only behavior alone does not make concurrent modification safe.

## Dependencies

There are no called-function dependencies. No additional opaque function IDs are needed, and no assumptions about a callee preserving globals, retaining addresses, or releasing memory are required.

The contract depends on environmental definitions for:

- Input and selector widths and representations.
- Valid signed-addition ranges and consequences outside those ranges.
- Interpretation of calculated integer values as addresses.
- Storage lifetime, alignment, permissions, and read semantics.
- Representation and delivery of the four-byte return value.
- Applicable calling-convention and concurrency rules.

Byte order is unresolved. Value preservation within a compatible environment does not define a portable external byte format.

No constructor, allocator, cleanup operation, or caller-specific consistency protocol is established.

## Acceptance examples

These are conditional acceptance requirements, not executed tests. Except where an example explicitly concerns invalid conditions, assume valid required arithmetic, address interpretation, readable ordinary storage, compatible representations, and stable values.

### Return values

| Selector observation | Stored-result contents | Required return | Required logical reads |
|---|---|---|---|
| Zero | All 32 bits clear | All 32 bits clear | Selector, then stored result |
| Zero | Numeric one | Numeric one | Selector, then stored result |
| Zero | Only the highest of 32 bits set | Identical 32-bit pattern | Selector, then stored result |
| Zero | All 32 bits set | Identical 32-bit pattern | Selector, then stored result |
| One | Any contents | Numeric one | Selector only |
| Any other representable positive value | Any contents | Numeric one | Selector only |
| Any representable negative value | Any contents | Numeric one | Selector only |

The high-bit cases impose bit preservation without assigning signed numeric meanings to the stored result.

Returning one for every nonzero stored value would violate the zero-selector cases. Testing only the selector’s lowest bit would violate a case involving a representable nonzero selector whose lowest bit is clear.

A particular four-byte selector pattern is not an unconditional input example: the selector’s width and representation remain unresolved. These examples apply to valid selector values in the applicable environment.

### Conditional access

**Nonzero selector; inaccessible stored-result location.** If the selector access and its address formation are valid, the required result is one. Inaccessibility of storage that this case does not read must not itself introduce a required access or fallback.

**Zero selector; inaccessible stored-result location.** The required four-byte observation cannot meet its preconditions. No ordinary-completion value or recovery behavior is specified.

**Inaccessible selector; readable stored result.** The readable stored result supplies no alternate path to a specified return value.

**Changed stored value; unchanged nonzero selector.** The required result remains one, without a logical stored-result read.

### Observation boundaries

**Return value one.** The return value alone does not establish whether the stored-result location was read.

**Selector changes after observation.** Where the environment permits this change, no reread or restart is required by the available evidence.

**Stored result changes between observations.** Where such modification is permitted, the function contributes no snapshot guarantee. The returned stored value is governed by the applicable read semantics.

**Required signed addition is outside its valid range.** Wrapped-address behavior and ordinary completion are not required. Consequences remain **documented_only** unless the environment defines them.

## Uncertainties

The following remain **documented_only**:

- Application meanings of the selector, stored result, and returned one.
- Exact input and selector widths, including the selector’s read extent.
- Address width and integer-to-address conversion rules.
- Signed-arithmetic limits and invalid-operation consequences.
- Complete object layout, allocation size, ownership, and lifetime protocol.
- Stored-result signedness and application type.
- Byte order, alignment requirements, and misaligned-access behavior.
- Permitted concurrent modification, atomicity, and memory ordering.
- Read-triggered effects of special storage.
- Physical activity beyond the established logical reads.
- Calling-convention details and environmental fault handling.

An **inferred intent** is that the selector activates an override of a stored answer. This interpretation is not required behavior and supplies no basis for Boolean normalization, permission semantics, or success terminology.

## Evidence

**Established static evidence:** One signed-integer input supplies the base for storage at byte displacements 120 and 128. The first location supplies a signed-integer selector. The second supplies a four-byte value. The return width is four bytes. A zero selector selects the stored value; every nonzero selector selects one.

**Access and effect evidence:** The selector observation precedes the conditional stored-result observation. No explicit reread, write, call, allocation, release, validation, or recovery operation is present.

**Width and arithmetic boundary:** The stored result and return have explicit four-byte widths. Exact input and selector widths require an execution-model definition. Signed address-forming additions do not establish wraparound semantics.

**Inference boundary:** The conditional selection and logical reads are established. Application purpose, ownership protocols, concurrency safety, physical memory activity, and invalid-operation outcomes are not.

**Verification status:** Static analysis only. No runtime tests have been performed. The acceptance examples state conditional requirements derived from the supplied evidence, not observed executions.
