<!-- Trusted provenance: F-dc8fc626c77d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dc8fc626c77d

## Purpose

F-dc8fc626c77d searches a contiguous sequence of 32-bit values for the first value equal to a supplied key. If a match is found and a subsequent guard permits removal, it closes the gap by copying later values toward the beginning and decreases the sequence’s stored end address by four bytes.

Independently of whether removal occurs, it decreases another 32-bit object field by one. It then conditionally invokes an unresolved indirect dependency, using a pointer and a value reached through that pointer as the invocation gate.

The interpretation as “remove one matching element” is inferred intent. The established behavior is the ordered sequence of calls, reads, comparisons, copies, and field updates described below. In particular, the separate decrement occurs even when the sequence is empty or the key is absent.

## Interface

### Inputs

The function has three supplied arguments:

| Argument | Role |
|---|---|
| Object address | Base address used to access the object fields and to form the initial dependency argument. |
| Search key | A 32-bit value compared for equality with sequence elements. |
| Third argument | No observable influence on the function’s behavior in the supplied body. It is replaced with a loaded element before any comparison that uses it. |

The supplied representation uses 32-bit integer-sized storage and four-byte element strides. Object addresses and stored sequence addresses are represented as 32-bit quantities in that representation. No wider-address interface is established.

Equality compares the complete 32-bit element value with the complete 32-bit key. Signedness does not change this equality relation.

### Object layout

Offsets are measured in bytes from the object address:

| Offset | Accessed contents | Role established by use |
|---|---|---|
| +4 | Address-sized field, also passed by address to the initial dependency | Later loaded as a pointer to a 32-bit value used by the final invocation gate. |
| +12 | 32-bit field | Decreased by one on the ordinary path, regardless of search outcome. |
| +20 | Sequence-begin address | Initial search position; conditionally reread before removal. |
| +24 | Sequence-end address | Exclusive search limit; conditionally decreased by four bytes. |

This table describes accessed locations, not a complete object definition. No meaning is established for other offsets.

For an ordinary well-formed sequence, the begin and end addresses delimit a readable contiguous range of four-byte elements. Elements overwritten during removal must also be writable. The end address is exclusive: an empty sequence has equal begin and end addresses.

### Output

There is no explicit return value and no reported “found,” “removed,” or error result.

On ordinary completion, observable results consist of memory changes and dependency effects. If either dependency fails to return normally, this body does not establish ordinary completion.

## Behavior

### 1. Initial dependency invocation

Before reading either sequence boundary, the function invokes F-4a19e3742733 with the address of the object’s field at offset +4.

The argument is the address of that field, not its stored value.

The function does not establish what this dependency does. In particular, it cannot be assumed to preserve:

- The field at offset +4.
- The sequence boundaries.
- Sequence contents.
- The field at offset +12.
- Other memory or global state.
- The lifetime or ownership of referenced storage.

All later behavior uses the state that exists after this call returns. Any valid-memory preconditions for those later accesses must therefore hold after the call, not merely at entry.

### 2. Boundary reads

After the initial dependency returns, the function reads:

1. The sequence-end address from offset +24.
2. The sequence-begin address from offset +20.

These values become the search limit and initial search position, respectively. The search limit is retained rather than reloaded on each iteration.

The end-before-begin read order is part of the supplied body’s access sequence. No atomic snapshot or synchronization guarantee is established.

### 3. Search

At each search position:

- If the position equals the retained end address, the search stops without reading an element there.
- Otherwise, the function reads the 32-bit element at that position.
- If the element equals the search key, the search stops at that position.
- Otherwise, the position advances by four bytes and the process repeats.

Consequently:

- An empty range causes no element read.
- The first matching element is selected.
- Elements after the first match are not read by the search.
- A missing key causes every element in a valid finite range to be read once by the search.
- The caller-supplied third argument does not substitute for an element value, including in the empty-range case.

Termination relies on a valid, reachable end address or an earlier match. The function does not validate that the end lies after the begin address, that the distance is a multiple of four, or that the traversed memory is accessible.

### 4. Removal guard

Removal requires both of the following:

1. The search stopped at a position different from the retained end address.
2. A fresh read of the begin field at offset +20 differs from that retained end address.

The second condition is checked only when the first condition succeeds. If the search reaches the end, the begin field is not reread for this guard.

For a stable, well-formed, non-overlapping object, finding a match already implies a nonempty range. Nevertheless, the extra begin-field read and comparison are present and remain part of the behavior. The refreshed begin value is used only for this guard; it does not replace the selected position or restart the search.

### 5. Gap closing

If removal is permitted, each element after the selected element is copied to the immediately preceding slot, in increasing address order.

The first copy reads the selected element’s successor and writes the selected element’s slot. Subsequent copies proceed toward the retained end address. Each source value is read before being written to its preceding destination.

If the selected element is immediately before the retained end address, no element copy occurs.

For an ordinary sequence, this produces stable removal: the relative order of all remaining elements is preserved. If several elements equal the key, only the first selected occurrence is removed by this invocation.

The number of element copies is the number of elements after the selected occurrence. There is no allocation or replacement of the backing storage established by this body.

### 6. End-field update

After any required copying, the function reads the current 32-bit value at offset +24, subtracts four, and writes the result back to that field.

The update uses this fresh field value. It does not simply store the retained search-end address minus four.

For a stable object with non-overlapping metadata and element storage, those values coincide. If earlier writes affect the field through aliasing, they need not coincide.

The old final element slot is not cleared. It lies outside the shortened logical range but retains whatever bits were last stored there.

### 7. Unconditional field decrement

After the search and any removal work, the function reads the 32-bit field at offset +12, subtracts one, and writes the result back.

This operation occurs when:

- A match is removed.
- No match is found.
- The sequence is empty.
- A match is found but the removal guard rejects it.

“Unconditional” here means unconditional on the search result, provided execution reaches this stage. A prior nonreturning call or invalid memory access can prevent it.

There is no check for zero, no saturation behavior, and no consistency check against the sequence length. The supplied body establishes a 32-bit subtract-and-store operation. It does not independently establish the source-language rules for signed overflow. Exact overflow-boundary results require confirmation of the underlying arithmetic semantics.

### 8. Final invocation gate

After the decrement, the function reads the pointer stored at offset +4.

- If that pointer is null, it does not dereference it and does not invoke the final indirect dependency.
- If the pointer is nonnull, it reads the 32-bit value at the pointed-to address.
- If that value is zero, it does not invoke the final indirect dependency.
- If that value is nonzero, it invokes the unresolved indirect dependency.

The dependency’s target comes from a separate indirect-call location, not from the pointer stored in the object’s offset +4 field.

The gate is evaluated after all preceding mutations. If the pointed-to word aliases a location modified earlier, the final decision uses the updated value.

No explicit argument list, return-value use, or ownership contract is established for the indirect invocation. If it returns normally, the function then completes without further explicit object accesses in the supplied body.

## Side effects

### Direct memory effects

This body directly performs only the following writes:

- Zero or more four-byte sequence-element copies.
- One four-byte end-field update if removal is permitted.
- One four-byte decrement at offset +12 whenever that stage is reached.

It does not explicitly clear the removed element’s former trailing slot, allocate storage, release storage, or resize an allocation.

These statements do not constrain the dependencies. Either dependency may have additional effects that are undocumented here.

### Ownership and lifetime

No ownership transfer is established for the object, sequence storage, gate pointer, or gate pointee.

The initial dependency receives an address within the object. Whether it retains that address, releases anything, or changes ownership is unknown. The final dependency’s ownership behavior is also unknown.

After removal, the stored end address identifies a shorter logical range under the ordinary sequence interpretation. This alone does not establish destruction of an element or release of its associated resources.

### Errors and abnormal cases

The body contains no explicit error-reporting path and no validation for:

- An invalid object address.
- Invalid sequence boundaries.
- Misaligned or inaccessible element addresses.
- A nonnull but unreadable gate pointer.
- An invalid indirect-call target.
- Arithmetic boundary conditions.
- Concurrent mutation.

An empty sequence avoids element reads during the search, but it does not avoid object-field accesses or the later gate evaluation.

The function offers no rollback. Failure after some copies or after the end update may leave partial mutations visible.

## Dependencies

### F-4a19e3742733

- Invoked once before sequence-boundary reads.
- Receives the address at object offset +4.
- No result is consumed by this body.
- Its implementation, memory effects, ownership effects, and failure behavior are not supplied.
- Later operations require the memory they access to remain valid after this dependency returns.

### Unresolved indirect dependency

No opaque function ID is supplied for this target.

- Invoked at most once by this body.
- Invoked only when the final gate pointer and its loaded 32-bit value are both nonzero.
- Target obtained through a separate indirect-call location.
- Call interface, target identity, state effects, ownership effects, and failure behavior are undocumented.
- No assumption of global-state preservation is justified.

## Acceptance examples

These examples are conditional behavioral checks, not observed runtime results. Unless stated otherwise, assume valid memory, stable non-overlapping fields and sequence storage, normally returning dependencies, and arithmetic values away from overflow boundaries.

### First of several matches

Given sequence `[8, 3, 8, 5]`, key `8`, and decrement field `10`:

- The first element is selected.
- The next three values are copied one slot toward the beginning.
- The logical sequence becomes `[3, 8, 5]`.
- The end address decreases by four bytes.
- The decrement field becomes `9`.
- The old trailing slot is not cleared.

The later `8` remains in the logical sequence.

### Match at the end

Given sequence `[2, 6, 9]` and key `9`:

- All three elements are read during the search.
- No element copy is performed.
- The end address decreases by four bytes.
- The logical sequence becomes `[2, 6]`.
- The separate field is still decremented.

### Missing key

Given sequence `[2, 6, 9]`, key `4`, and decrement field `7`:

- Each sequence element is read.
- No sequence element is written.
- The end field is unchanged.
- The decrement field becomes `6`.
- The final invocation gate is still evaluated.

### Empty sequence

Given equal begin and end addresses and decrement field `7`:

- No sequence element is read.
- No removal occurs.
- The end field is unchanged.
- The decrement field becomes `6`.
- Changing only the supplied third argument cannot change these outcomes.

### Gate outcomes

After preceding work:

- A null pointer at offset +4 prevents both a pointee read and the indirect invocation.
- A nonnull readable pointer to zero causes a pointee read but no indirect invocation.
- A nonnull readable pointer to a nonzero 32-bit value causes the indirect invocation.

### Initial dependency changes the sequence

If F-4a19e3742733 replaces the sequence boundaries before returning, the search uses the replacement boundaries. Entry-time boundaries are not loaded or retained by this body before that call.

### Gate aliases the decrement field

If the pointer loaded from offset +4 points to the field at offset +12, and that field changes from `1` to `0`, the final pointee read observes zero and suppresses the indirect invocation.

## Uncertainties

### Inferred intent

- The begin/end pair appears to describe a contiguous sequence.
- The copy-and-shorten operation appears intended to erase the first matching value.
- The field at offset +12 may represent a count, but its unconditional decrement prevents establishing that it is the current sequence length.
- The final gate may relate to lifecycle handling, but no such semantics are established.

### documented_only

The following remain unspecified:

- Full object type, allocation size, alignment requirements, and ownership model.
- The behavior of F-4a19e3742733.
- The identity and interface of the indirect dependency.
- Whether overlapping metadata and sequence storage are permitted.
- Concurrency, synchronization, and externally changing memory semantics.
- Source-language overflow rules and behavior at invalid pointer-arithmetic boundaries.
- Failure behavior of memory accesses and dependencies.

These unknowns must not be replaced with assumptions about locking, reference counting, destruction, callback arguments, or resource release.

## Evidence

The contract is based solely on static inspection of the supplied function body and dependency mapping.

Directly supported observations include the initial dependency call, byte offsets, four-byte element stride, first-match search, conditional forward copying, fresh end-field subtraction, unconditional separate decrement, and final pointer-and-pointee gate.

Semantic labels such as “sequence,” “removal,” and “logical range” describe the observed structure and are distinguished from undocumented domain intent.

No runtime tests have been performed. No dependency implementation or wider caller contract was supplied.
