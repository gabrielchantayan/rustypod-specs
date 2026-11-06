<!-- Trusted provenance: F-addc1a87ba34; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-addc1a87ba34

## Purpose

F-addc1a87ba34 coordinates a preliminary check, a secondary operation, and one of two completion paths.

- The **local path** prepares temporary state, performs a 16-bit counter update, invokes an indirect dependency, derives and publishes a 32-bit value, sets a byte flag, and finalizes the temporary state.
- The **fallback path** prepares three temporary words and delegates completion.

The routine returns the preliminary check’s nonzero result, the fallback dependency’s result, or zero after normal local completion.

These are established control-flow and memory-access behaviors. Resource acquisition, registration, activation, and ownership transfer are not established meanings.

## Interface

### Inputs

The routine accepts three machine-level arguments:

| Input | Established width | Use |
|---|---:|---|
| Input record address | 32 bits | Supplies an unsigned 16-bit field at byte offset 16; passed unchanged to F-256e8a77aa59. |
| Auxiliary argument | 32 bits | Passed unchanged to F-256e8a77aa59. |
| Payload argument | 32 bits | Written into the first word of a two-word temporary region. |

The input record must permit the 16-bit read at offset 16 if execution reaches that read. The routine supplies no explicit null, bounds, or alignment check. Dependencies may impose additional requirements that are not established here.

### Return value

The exposed result is a 32-bit integer:

| Condition | Returned result |
|---|---|
| F-43ce968102ca returns nonzero | That result, unchanged. |
| F-43ce968102ca returns zero and F-256e8a77aa59 returns nonzero | The result of F-704f99fd5e22, unchanged. |
| Both preliminary dependencies return zero and local processing completes normally | Zero. |

Branch selection distinguishes zero from nonzero, not positive from negative. F-256e8a77aa59’s nonzero result selects fallback; it is not itself returned.

### Shared roots and observed layouts

Two shared address-bearing values are used:

- **Shared context root:** a 32-bit value whose referenced storage has an address-bearing 32-bit field at byte offset 4.
- **Shared lookup base:** a 32-bit value used in lookup-address formation.

After the first dependency returns, the routine saves the then-current shared context-root value. This **saved root** is distinct from later fresh reads of the shared context root.

A context address obtained through a root’s offset-4 field exposes these locations:

| Context offset | Width | Direct local-path use |
|---|---:|---|
| 764 | 32 bits | Store the derived value. |
| 768 | 8 bits | Store 1. |
| 770 | 16 bits | Read a counter value or receive its incremented value. |

The counter read and counter write independently obtain their context addresses. They are not guaranteed to address the same context.

These accesses do not establish complete object layouts, semantic field names, or object ownership.

### Temporary regions

The routine uses two call-local regions:

| Region | Size and layout |
|---|---|
| Three-word region | Three adjacent 32-bit words at byte offsets 0, 4, and 8. |
| Two-word region | Two adjacent 32-bit words at byte offsets 0 and 4. |

The regions are distinct. The fallback passes three interior pointers into the three-word region, with four-byte spacing.

Their storage exists for this invocation. Passing their addresses does not establish permission to retain them after return, nor establish ownership of anything their contents may represent.

### Address arithmetic and validity

Observed addresses are 32-bit address-bearing quantities. The lookup address is formed by adding the lookup base directly to a context address; there is no element-size multiplier or additional table-index indirection.

The representation does not establish a portable rule for overflowing address calculations. Required field, lookup, and callable-target addresses must be valid for the executed accesses. Invalid-address behavior, alignment tolerance, and address-overflow behavior are documented_only.

## Behavior

### 1. Obtain a token and prepare temporary inputs

The routine first calls F-5f5f14cedda3 with no explicit arguments and retains its 32-bit result as a token.

After that call, the routine performs these operations in order:

1. Reads the shared context-root value and retains it as the saved root.
2. Reads the unsigned 16-bit input-record field at byte offset 16.
3. Zero-extends that field into the third word of the three-word region.
4. Copies the payload argument’s 32-bit pattern into the first word of the two-word region.

At this point, the routine has not explicitly initialized the first two words of the three-word region or the second word of the two-word region.

The token dependency precedes both the saved-root capture and the input-record read. Its effects must not be assumed absent.

### 2. Prepare and invoke the preliminary check

F-43ce968102ca receives five arguments:

1. A context address obtained by freshly reading the shared context root and then reading that root’s 32-bit field at offset 4.
2. The retained token.
3. A 16-bit lookup value obtained using the shared lookup base plus a context address obtained through another fresh shared-context-root read and its offset-4 field.
4. The zero-extended input-record field held in the third temporary word.
5. Zero.

**Neither context-root read in these argument expressions uses the saved root.** The initial saved-root capture, the fresh root read for the first argument, and the fresh root read for the lookup expression are distinct reads.

Consequently:

- The first argument need not refer to the context used to address the lookup entry.
- Neither context need match a context subsequently reached through the saved root.
- A later change to the shared context-root value does not replace the already saved root.

The representation does not establish evaluation order among the argument expressions, including the relative order of the two fresh root reads. These reads are not an established atomic snapshot.

The lookup access is 16 bits. Unlike the explicitly unsigned input-field and fallback lookup conversions, the preliminary lookup expression does not establish how its 16-bit value is extended or represented in a wider argument slot.

If F-43ce968102ca returns nonzero, the routine returns that exact 32-bit result. It performs no subsequent secondary call, counter update, indirect call, publication, finalization, or fallback call.

### 3. Invoke the secondary operation

When the preliminary result is zero, F-256e8a77aa59 receives:

1. The original input record address.
2. The original auxiliary argument.

A zero result selects the local path. Any nonzero result selects fallback.

This dependency is not assumed to preserve the shared roots, the saved root’s referenced storage, input-record contents, or other reachable memory. The saved root’s retained numeric value nevertheless remains the root used by the later context-access expressions.

### 4. Local path

The following sequence applies when F-256e8a77aa59 returns zero and each dependency returns normally.

#### Prepare the three-word region

F-7a117b899bf6 receives the address of the three-word region. Its return value, if any, is not used directly.

Before this call, this routine has explicitly initialized only the third word of that region. The dependency may change the region; preservation of the third word is not established.

#### Perform the counter update

The update uses **two separate reads of the saved root’s offset-4 field**:

- One supplies the context address for the 16-bit counter read at offset 770.
- The other supplies the context address for the 16-bit counter write at offset 770.

The representation does not establish the relative evaluation order of these two address-producing reads. It does establish that the stored value is derived from the counter value read: that 16-bit bit pattern plus one, truncated to 16 bits.

If the two offset-4 reads return the same address, this behaves as an increment of one selected counter. If they return different addresses, the routine reads one context’s counter and writes the incremented result into another context’s counter. It does not necessarily increment the write destination’s previous value.

For example, a read bit pattern of 65,535 produces a stored bit pattern of zero, regardless of the write destination’s prior contents.

The actual counter store precedes the indirect call. No atomic read-modify-write operation, saturation behavior, or synchronization mechanism is established.

#### Invoke the indirect dependency

After the counter store, the routine:

- Reads the first word of the three-word region as an address-bearing value.
- Reads a callable target at byte offset 12 relative to that value.
- Calls that target with the three-word region’s address followed by the two-word region’s address.

This is a direct offset-12 target lookup from the address contained in the first word. An additional intermediate table-pointer read is not shown.

Only the first word of the two-word region was explicitly initialized by this routine. Whether the second word is output-only, unused, or required to contain some preexisting value is unresolved.

The indirect dependency’s return value is not used directly.

#### Derive the value to publish

F-bf18a6a8124b receives the two-word region’s address after the indirect call. Its 32-bit result becomes the value to publish.

The routine does not establish whether this dependency reads, changes, consumes, or releases anything represented by that region.

#### Publish the value and flag

After F-bf18a6a8124b returns, the routine reads the saved root’s offset-4 field once and retains that context address for both publication stores:

1. Store the derived 32-bit value at context offset 764.
2. Store byte value 1 at context offset 768.

Thus, both publication stores use one captured context address. This differs from the counter assignment’s independently evaluated read and write destinations.

The publication context need not equal either counter context. Dependencies may have changed the saved root’s offset-4 field in the meantime.

The value store precedes the flag store. No barrier or atomic-publication guarantee is established.

#### Finalize and return

F-ec22bd8215d3 receives the three-word region’s address after both publication stores.

If it returns normally, the routine returns zero, independently of that dependency’s return value. The finalizer may itself alter shared state; the contract does not promise that the publication fields remain unchanged afterward.

### 5. Fallback path

When F-256e8a77aa59 returns nonzero, the routine:

1. Writes zero into the second word of the three-word region.
2. Reads a context address through the saved root’s offset-4 field.
3. Reads an unsigned 16-bit lookup entry at the shared lookup base plus that context address.
4. Zero-extends the lookup entry into the first temporary word.
5. Calls F-704f99fd5e22.

The fallback receives:

| Argument | Value |
|---|---|
| First | A context address obtained by another read of the saved root’s offset-4 field. |
| Second | The original retained token. |
| Third | Address of the first temporary word. |
| Fourth | Address of the third temporary word. |
| Fifth | Address of the second temporary word. |

The context used for the lookup and the context supplied to fallback are obtained separately and need not match.

The pointer order is first word, third word, second word—not ascending address order. The routine has explicitly supplied the lookup value, the earlier input-field value, and zero in those respective words, subject to any intervening external modification.

F-704f99fd5e22’s result becomes the routine’s result. No subsequent local-path finalization, counter update, or publication occurs.

## Side effects

Established direct effects include:

- Reading the input record’s 16-bit field.
- Reading shared roots, offset-4 context-address fields, lookup entries, and an indirect callable target.
- Writing call-local temporary regions.
- On the local path, reading a 16-bit counter and writing its incremented bit pattern, potentially in a different context.
- Subsequently storing a 32-bit value and an 8-bit flag into one publication context.

The routine does not directly write the input record. Its address is passed to F-256e8a77aa59, so the complete operation is not established as read-only.

No explicit allocation, release, or ownership transfer is established. “Prepare” and “finalize” describe dependency placement, not proven resource-management semantics.

No rollback is shown. If the indirect dependency does not return normally, the counter store has already occurred but publication and finalization have not. If finalization does not return normally, both publication stores have already occurred. Exceptional propagation and any dependency-provided cleanup remain unknown.

## Dependencies

| Opaque function ID | Established role |
|---|---|
| F-5f5f14cedda3 | First call; supplies the retained 32-bit token. |
| F-43ce968102ca | Five-argument preliminary check; a nonzero result is returned immediately. |
| F-256e8a77aa59 | Receives the first two original inputs; selects local processing or fallback. |
| F-7a117b899bf6 | Receives the three-word region before the counter update. |
| F-bf18a6a8124b | Receives the two-word region after the indirect call; supplies the publication value. |
| F-ec22bd8215d3 | Receives the three-word region after publication. |
| F-704f99fd5e22 | Receives context, token, and three interior pointers; supplies the fallback result. |

An additional indirect dependency has no supplied opaque ID. Its target is read at offset 12 from the address contained in the first temporary word.

No dependency is assumed to preserve globals, preserve temporary contents, own represented resources, release memory, or refrain from retaining passed addresses.

## Acceptance examples

These are conditional acceptance criteria, not executed tests.

1. **Preliminary rejection.** If F-43ce968102ca returns 7, the routine returns 7 without calling F-256e8a77aa59 or either completion path.

2. **Nonzero bit patterns.** A preliminary result interpreted as negative still causes immediate return of that unchanged result.

3. **Independent preliminary roots.** Suppose the saved root is A, the fresh root read for the first preliminary argument yields B, and the separate root read for the lookup yields C. The first argument follows B’s offset-4 field; the lookup follows C’s offset-4 field. Later context accesses use A. No particular evaluation order between the B and C reads is required.

4. **Stable-pointer counter boundary.** If both counter-address reads yield context X and X’s counter reads as 65,535, the routine stores zero into X’s counter before invoking the indirect dependency.

5. **Different counter destinations.** If the counter-read address resolves to X with counter value 12, while the counter-write address resolves to Y, the routine stores 13 into Y’s counter. Y’s prior counter value does not determine that stored result. This condition does not prescribe which address-producing read occurs first.

6. **Normal local completion.** If both preliminary dependencies return zero, local dependencies return normally, and F-bf18a6a8124b returns V, the routine performs the counter update, invokes the indirect dependency, stores V and then byte 1 into the captured publication context, finalizes, and returns zero.

7. **Changed publication context.** If a dependency changes the saved root’s offset-4 field before the publication-context read, publication follows the new field value. Changing only the shared context-root value does not replace the saved root.

8. **Fallback result selection.** If F-256e8a77aa59 returns 9 and F-704f99fd5e22 returns 4, the routine returns 4 and performs no direct local-path counter or publication writes.

9. **Fallback layout and extension.** With input field 65,535 and fallback lookup value 32,768, and without intervening modification, the fallback’s last three arguments point to 32-bit words containing 32,768, 65,535, and zero, respectively.

10. **Publication before finalization.** Entry into F-ec22bd8215d3 occurs after both publication stores; publication is not deferred until finalization succeeds.

## Uncertainties

**documented_only:**

- Semantic meanings of the token, lookup values, counter, publication value, and flag.
- Complete layouts, declared types, calling convention, and dependency-specific validity requirements.
- Widening or signedness of the preliminary check’s 16-bit lookup argument.
- Dependency effects, ownership, allocation, release, address retention, and abnormal control flow.
- The indirect dependency’s identity and temporary-region requirements.
- Requirements concerning explicitly uninitialized temporary words.
- Byte order, alignment tolerance, address-overflow behavior, and invalid-address handling.
- Concurrency, memory ordering, and consistency across repeated root or context-field reads.
- Relative evaluation order of preliminary argument expressions and of the counter assignment’s two address-producing expressions.

**Inferred intent:** The local sequence resembles temporary-state preparation, an operation producing an output representation, publication, and finalization. That interpretation establishes neither ownership nor transactional behavior.

## Evidence

This contract derives from the supplied static representation and opaque dependency mapping, not from the prior draft’s claims.

The decisive evidence includes distinct saved-root and fresh-shared-root expressions; two separate saved-root offset-4 reads in the counter assignment; explicit access widths and offsets; unsigned conversions for the input field and fallback lookup; dependency ordering; interior-pointer relationships; and return-value selection.

The representation includes a type-propagation warning. Explicit memory-access widths and data-flow relationships therefore support stronger conclusions than inferred high-level types.

No implementation access, runtime execution, or runtime tests were performed. All acceptance examples are conditional static specifications.
