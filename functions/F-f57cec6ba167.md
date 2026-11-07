<!-- Trusted provenance: F-f57cec6ba167; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f57cec6ba167

## Purpose

F-f57cec6ba167 requests a descriptor from F-fefc45cb405b, uses the descriptor’s first word to locate destination storage, transforms bytes from a supplied source address, and finally writes a pointer-sized value to a caller-supplied output slot.

Mode zero places the transformed bytes 12 bytes beyond the destination base. Every nonzero mode places them directly at that base. The modes also supply different size arguments to the dependency, but both transform the full input length.

The final published value is the **then-current contents of the private descriptor-pointer slot** passed to the dependency. It is not unconditionally the descriptor pointer originally supplied by that dependency: transformation writes can change the private slot if destination storage aliases it.

The evidence establishes directional byte transformation and delayed publication. It does not establish format validation, inverse operations between modes, initialization of a 12-byte header, or ownership transfer.

## Interface

The interface has four arguments and returns an unsigned 32-bit value.

| Argument | Width and interpretation |
|---|---|
| Output slot | Address of storage receiving a 32-bit pointer representation at final publication. |
| Length | Unsigned 32-bit byte count, denoted \(N\). |
| Mode | Signed 32-bit selector. Zero and nonzero are the only distinguished cases. |
| Source address | Unsigned 32-bit address value, denoted \(S\). Zero is rejected. |

The function maintains a private descriptor-pointer slot, initialized to null. Its address is passed to F-fefc45cb405b. Denote the pointer in that slot immediately after a successful dependency call by \(Q\).

The function reads the first 32-bit word at \(Q\), obtaining a destination base \(B\). No other descriptor field is explicitly inspected. The destination start \(D\) is:

- Mode zero: \(D=(B+12)\bmod 2^{32}\).
- Nonzero mode: \(D=B\).

These definitions describe the supplied 32-bit address operations. They do not independently establish the execution platform’s complete pointer, alignment, or fault semantics.

The return value is one of:

- The shared 32-bit error value, for either initial-validation failure.
- The dependency’s unchanged nonzero 32-bit status.
- Zero after transformation and final output publication complete.

The shared error value’s contents are unknown. If it can contain zero, a zero return does not by itself distinguish successful publication from initial rejection.

### Preconditions for normal completion

The function does not establish or enforce the following conditions:

- The output slot supports the final 32-bit write.
- After dependency success, the private slot contains a pointer permitting a first-word read.
- Every visited source byte is readable.
- Every visited destination byte is writable.
- Required alignment and pointer-representation rules are satisfied.

A nonzero source address does not establish source accessibility. A successful dependency status does not, from this body alone, establish a valid descriptor or sufficient destination capacity.

No non-aliasing requirement is checked. In particular, the supplied body does not establish that destination storage is separate from the private pointer slot whose address the dependency receives.

## Behavior

### Initialization and rejection

The private descriptor-pointer slot is initialized to null before the source-address check. This initialization is private bookkeeping; it is not publication to the caller’s output slot.

The source-address check has precedence:

- If \(S=0\), the function reads and returns the shared error value.
- It does not call F-fefc45cb405b, read a descriptor, transform bytes, or directly write the caller’s output slot.

For \(S\ne0\):

- Mode zero accepts every unsigned length, including zero.
- Nonzero mode rejects \(N<12\), returning the same shared error value without a dependency call or direct output-slot write.

Negative mode values follow the nonzero-mode path. A null source is rejected even when \(N=0\).

### Dependency request

For arguments passing validation, F-fefc45cb405b is called exactly once with:

| Mode | First argument | Second argument |
|---|---:|---|
| Zero | \(N\) | Address of the private descriptor-pointer slot |
| Nonzero | \(N-12\) | Address of the private descriptor-pointer slot |

The nonzero-mode subtraction occurs only after checking \(N\ge12\), so it does not underflow.

Only an exact zero dependency status permits continuation. Any nonzero status, including a value with its high bit set, is returned unchanged. That failure path performs no subsequent descriptor read, byte transformation, or direct publication.

This is not a rollback guarantee. The dependency may already have modified memory, globals, or its supplied output slot. Its effects are not constrained by the visible body.

### Destination acquisition

After dependency success, the function reads the private pointer slot to obtain \(Q\), then reads the first 32-bit word at \(Q\) to obtain \(B\).

Neither \(Q\) nor \(B\) is validated. A null pointer left in the private slot is not handled as a separate error.

The destination base is retained for the byte transformation. Later modification of the descriptor’s first word or of the private pointer slot does not cause the function to reacquire the destination base.

This distinction matters: the address used for transformation comes from the earlier descriptor read, while final publication uses a later read of the private pointer slot.

### Byte transformation

For a source byte \(x\), interpreted as an unsigned value from 0 through 255, the stored byte is:

\[
T(x)=(159x+89)\bmod256.
\]

The supplied arithmetic first multiplies the promoted source character by 73, subtracts one, and multiplies by \(-89\). Its low eight bits yield the expression above.

Signed versus unsigned interpretation of an 8-bit source character does not change the stored low eight bits. The two interpretations differ by a multiple of 256, which remains a multiple of 256 through this arithmetic. The visible intermediate values fit within signed 32-bit range for either interpretation.

Every visited byte is transformed, including zero. There is no terminator, delimiter, content check, or value-dependent early exit.

### Length and destination placement

For mode zero:

- The dependency receives size \(N\).
- Destination processing starts at \(B+12\), modulo \(2^{32}\).
- The function transforms \(N\) bytes if execution completes.

For nonzero mode:

- The dependency receives size \(N-12\).
- Destination processing starts at \(B\).
- The function still transforms \(N\) bytes if execution completes.

In particular, nonzero mode with \(N=12\) requests size zero but transforms 12 bytes. The dependency’s request size is not established to mean exact writable capacity. The mismatch is established; whether it exceeds available storage remains dependent on the unknown dependency contract.

### Traversal and address arithmetic

An unsigned comparison of \(D\) and \(S\) selects traversal:

- If \(D<S\), visit indices \(0,1,\ldots,N-1\).
- Otherwise, visit indices \(N-1,N-2,\ldots,0\).

Equality selects descending traversal.

At each visited index \(i\), the function reads the source byte at \(S+i\), computes its transformed value, and writes that byte at \(D+i\). Each byte read precedes its corresponding byte write. There is no complete source snapshot or intermediate payload buffer.

Address arithmetic follows the visible unsigned 32-bit operations. Additions can wrap modulo \(2^{32}\); no explicit overflow error is produced. The direction decision compares starting addresses only and does not examine range ends.

The ascending loop does not require a counter wrap even for the maximum unsigned length: its last visited index is \(N-1\), and the next counter value equals \(N\). Descending traversal stops after processing index zero.

For ordinary nonwrapping source and destination ranges, the direction choice is consistent with preserving unread source bytes during overlap. That rationale is inferred intent, not a guarantee for wrapped ranges or arbitrary aliases involving control storage.

When \(N=0\), no payload bytes are read or written. Only mode zero can reach this case. Dependency acquisition, the descriptor’s first-word read, and final publication still occur after dependency success.

### Final publication

After completing the byte transformation, the function reads the private descriptor-pointer slot again and writes its then-current value into the caller’s output slot. It then returns zero.

The private slot’s contents need not still equal \(Q\). Its address was exposed to the unknown dependency, and the body does not establish separation between that slot and destination storage. Destination writes can therefore overwrite some or all of its pointer representation before publication.

Consequently:

- If the slot remains unchanged, the originally acquired descriptor pointer is published.
- If the slot has changed, its changed value is published.
- The published value is not dereferenced or checked before return.
- The destination base used during transformation remains the value acquired earlier, even if publication later uses a different pointer value.

There is no explicit cleanup or rollback after dependency success. A fault during transformation or final publication can leave earlier writes in place; recovery behavior is not established.

## Side effects

### Observable ordering

For a path that passes initial validation and receives dependency status zero, the relevant order is:

1. Call F-fefc45cb405b with access to the private pointer slot.
2. Read that slot and the referenced descriptor’s first word.
3. Perform source-byte reads and destination-byte writes in the selected direction.
4. Read the private pointer slot again.
5. Write that value to the caller’s output slot.
6. Return zero.

On an initial-validation failure, the relevant external read is the shared error value. On dependency failure, processing stops after the dependency returns.

The function contains no explicit write to the shared error value. This does not establish that unknown dependency effects or aliases cannot change it.

### Aliasing consequences

No alias checks or protective copies are present.

- **Source and destination:** They may overlap. The selected traversal determines which reads happen before potentially overlapping writes.
- **Output slot and payload:** The final output write occurs after transformation and may replace transformed bytes or source bytes.
- **Destination and descriptor:** Destination writes may modify descriptor storage. The already-acquired destination base is not reread.
- **Destination and private pointer slot:** Destination writes may modify the value later published.
- **Source and private pointer slot:** If they alias, source reads can observe that slot’s representation, including changes caused by earlier overlapping writes.
- **Dependency effects and source:** The dependency runs before any payload read. Source contents are not protected from its effects.

The private slot is not an immutable record of the descriptor initially returned. Its local placement does not establish non-aliasing once its address has been supplied to the dependency.

### Ownership and lifetime

No explicit release operation appears. The function does not explicitly free the source, the acquired descriptor, or destination storage.

Publication alone does not establish ownership transfer, reference-count changes, storage lifetime, or a required release procedure. If aliasing changes the private slot, publication may not even expose the originally acquired descriptor.

Whether an unexposed or overwritten descriptor requires cleanup is unknown. No cleanup mechanism for that situation is visible.

## Dependencies

### F-fefc45cb405b

Established interaction:

- Receives an unsigned 32-bit size.
- Receives the address of a private pointer slot initially containing null.
- Returns a 32-bit status.
- Has each nonzero status propagated unchanged.
- On zero status, is followed by use of the slot’s contents as a descriptor pointer.

The following remain **documented_only**:

- Whether the dependency allocates, retrieves, borrows, or reuses storage.
- What its size argument measures.
- Whether zero status guarantees a nonnull, readable descriptor.
- The capacity and layout of referenced storage.
- Whether it initializes any header or payload bytes.
- Whether it preserves globals or other caller-accessible memory.
- Whether it retains the private slot’s address.
- Whether it guarantees destination storage cannot overlap that slot.
- Whether failure leaves resources or other effects behind.
- Ownership, lifetime, and release requirements.

### Shared error value

A shared 32-bit value supplies both initial-validation returns. Its numeric contents, initialization, mutability, and relationship to dependency statuses are unknown.

It is read on the selected rejection path rather than established here as a fixed constant.

## Acceptance examples

These are conditional static expectations, not executed tests. Successful examples require accessible memory and compatible pointer-representation rules.

1. **Null source takes precedence.**  
   With \(S=0\), any length, and any mode, return the shared error value. Do not call the dependency or directly write the caller’s output slot.

2. **Short nonzero-mode input is rejected.**  
   With \(S\ne0\), \(N=11\), and any nonzero mode, return the shared error value without a dependency call. A negative mode is not a separate case.

3. **Dependency failure is preserved.**  
   With accepted initial arguments and dependency status \(0x80000001\), return \(0x80000001\). Do not subsequently read the descriptor or directly publish a value. Dependency side effects remain possible.

4. **Zero mode displaces the destination.**  
   With mode zero, \(N=4\), source bytes \(00,01,02,FF\), and valid destination storage, request size 4 and write hexadecimal bytes \(59,F8,97,BA\) beginning at \(B+12\). If the private slot remains unchanged, publish \(Q\).

5. **Zero length still reads the descriptor.**  
   With mode zero, \(N=0\), and \(S\ne0\), request size zero. After dependency success, read the descriptor’s first word, touch no payload bytes, and publish the private slot’s then-current value.

6. **Minimum nonzero-mode length transforms 12 bytes.**  
   With nonzero mode and \(N=12\), request size zero but transform 12 source bytes into storage beginning at \(B\). Transforming zero bytes would not match this behavior.

7. **Exact in-place transformation runs backward.**  
   With \(D=S\) and \(N=3\), process indices 2, 1, and 0. Assuming no additional aliases affect control storage, each byte is replaced by its own transformed value.

8. **Lower destination starts forward.**  
   With \(D<S\), process increasing indices, including for overlapping ranges. For ordinary nonwrapping ranges, this direction preserves source bytes needed by later iterations.

9. **Publication can overwrite payload.**  
   If the caller’s output slot lies within destination storage, complete transformation first, then write the private slot’s current pointer value there. The final payload may therefore differ from the transformed byte sequence at that location.

10. **Destination writes can change the published pointer.**  
    Suppose mode zero, \(N=4\), and the dependency supplies a readable descriptor whose base plus 12 identifies the private pointer slot. Suppose separate source storage contains \(00,01,02,FF\), and all accesses are valid. Transformation writes \(59,F8,97,BA\) over the slot’s four representation bytes. Final publication reads that altered representation, not a preserved copy of \(Q\). Its numeric interpretation depends on the target representation; publication does not require it to identify a valid descriptor.

## Uncertainties

The following remain **documented_only**:

- The intended names and meanings of the two modes.
- Whether the displacement corresponds to a header.
- Whether either mode represents encoding, decoding, or another operation.
- Descriptor layout beyond its first word.
- Destination capacity and the meaning of dependency request sizes.
- Dependency guarantees concerning the private pointer slot and aliases.
- Ownership and cleanup obligations, including when publication differs from \(Q\).
- The shared error value’s contents and stability.
- Platform-specific alignment, invalid-pointer, fault, concurrency, and memory-mapped-I/O behavior.
- Additional restrictions imposed by callers.

The body does not establish that the originally acquired descriptor is always published. Such a guarantee requires an additional condition ensuring that the private slot retains its value through final publication.

## Evidence

Established static evidence consists of the supplied function body and opaque dependency mapping:

- Private pointer-slot initialization precedes validation.
- Source rejection precedes mode-specific length rejection.
- Only nonzero mode requires \(N\ge12\).
- Dependency sizes are \(N\) and \(N-12\); both successful modes transform \(N\) bytes.
- Only dependency status zero continues.
- The descriptor’s first word supplies a destination base acquired before transformation.
- An unsigned start-address comparison selects traversal direction.
- Each source-byte read precedes its corresponding destination-byte write.
- Final publication reads the private pointer slot after transformation.
- No visible check prevents transformation from modifying that slot.
- No explicit release, rollback, or final published-pointer validation appears.

Overlap preservation is inferred intent. Dependency guarantees, storage sufficiency, ownership, and execution-platform details are not established. No runtime tests have been performed.
