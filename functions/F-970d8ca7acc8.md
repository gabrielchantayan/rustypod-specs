<!-- Trusted provenance: F-970d8ca7acc8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-970d8ca7acc8

## Purpose

F-970d8ca7acc8 performs two callback-mediated requests, validates an unsigned 32-bit quantity against a mode-dependent upper bound, writes a derived count into an object, and delegates loading to F-977b73bb902a.

The apparent purpose is to load a location-related table with two-byte or four-byte elements. That interpretation is inferred from diagnostic labels, division by two or four, and the loading arguments. The established contract is narrower: the function validates a quantity, records its truncated quotient, and forwards the undivided quantity to a dependency.

The operation is not transactional. The enclosing function performs no restoration of earlier state changes on its ordinary error-return paths. This is an ordering and no-restoration guarantee, not a guarantee that earlier values persist: dependencies have unknown mutation behavior and may overwrite the count or other state before returning.

## Interface

The supplied representation exposes four arguments and a signed 32-bit result.

| Input | Representation | Use |
|---|---|---|
| Object address | 32-bit address-bearing value | Provides a callback slot, a mode field, and object-relative destinations. Forwarded to both callbacks. |
| Context | Opaque 32-bit value | Forwarded unchanged to both callbacks and F-977b73bb902a. |
| Third argument | Opaque 32-bit value | Unused by this function. |
| Initial quantity | Unsigned 32-bit value | Initializes local writable storage passed to the second callback. |

The context’s interpretation as a pointer, handle, or scalar is unknown.

### Object layout

Offsets are byte offsets from the object address.

| Offset | Access established by this function |
|---|---|
| `0x0D2` | Read as a 16-bit mode field after both callbacks succeed. Zero and nonzero select different bounds and divisors. |
| `0x1F8` | Read as a callable address separately for each callback invocation. |
| `0x298` | Address supplied as the fourth argument of the first callback. No output width is established. |
| `0x2B4` | Written as an unsigned 32-bit derived count after validation succeeds. |
| `0x2B8` | Address supplied as the third argument of F-977b73bb902a. No output layout is established. |

The function also accesses a global slot containing the address of a diagnostic-control object. Within that object, offset `0x68` is read as a signed 32-bit diagnostic level. A diagnostic call is enabled only when the applicable level is greater than one.

Two 32-bit request selectors are used:

- The first is read from global storage for the first callback.
- The second comes from the first four bytes of static storage whose diagnostic-message address begins four bytes later.

The supplied evidence does not establish their numeric values or the byte order needed to interpret the second selector.

### Result

| Path | Signed 32-bit result |
|---|---|
| First callback returns nonzero | First callback result, unchanged |
| Second callback returns nonzero | `144`, hexadecimal `0x90` |
| Quantity fails its selected bound | `8` |
| F-977b73bb902a returns nonzero | Dependency result, unchanged |
| F-977b73bb902a returns zero | `0`, following any enabled completion diagnostic |

These results are path-dependent. A propagated status may itself equal 8 or 144; those numeric values do not uniquely identify a path.

## Behavior

### 1. Initialize the quantity and perform the first request

The function copies the initial quantity into local unsigned 32-bit storage before calling any dependency.

It reads the callback address from object offset `0x1F8` and invokes it with these arguments:

1. Object address.
2. First request selector.
3. Context.
4. Address at object offset `0x298`.

The enclosing function does not initialize the destination at `0x298` before this call. The callback’s output width, output interpretation, and other mutations are unknown.

After the callback returns, the function reads the global diagnostic-control pointer and retains its value for later checks. This pointer read occurs before testing the first callback’s result. Consequently, a nonzero first callback result still follows that global-pointer read.

If the callback returned nonzero, the function returns that status unchanged. It does not perform the second request, read the mode field, explicitly write the derived count, or call F-977b73bb902a.

### 2. Optionally request the initial diagnostic

After a zero first callback result, the function checks the signed 32-bit diagnostic level through the global diagnostic-control pointer expression. A level greater than one causes a call to F-4457a9a44a68 with an initial, location-related message address.

The supplied representation distinguishes this initial check from later checks:

- The initial check uses the global pointer expression.
- The rejection and completion checks use the pointer value retained immediately after the first callback.

The diagnostic level itself is not cached. Each reached check reads the applicable level field again.

A diagnostic dependency may modify globals or object contents. No preservation guarantee is established.

### 3. Perform the quantity request

The function reads the callback slot at object offset `0x1F8` again. It does not reuse a saved callback address from the first request.

The second invocation receives:

1. Object address.
2. Second request selector.
3. Context.
4. Address of the local unsigned 32-bit quantity.

Changes to the callback slot by the first callback or the intervening diagnostic dependency can therefore change which callable handles the second request.

The local storage was initialized from the caller’s fourth argument. The enclosing function does not clear it before the second request. If this callback returns zero without modifying that storage, its previous value remains the quantity used for validation.

If the second callback returns nonzero, the function returns 144 regardless of the particular nonzero status. It does not read the mode field, validate the quantity, explicitly write the count, call the loader, or request a rejection or completion diagnostic on that path.

### 4. Read the mode and validate the quantity

After both callbacks return zero, the function reads the 16-bit mode field at object offset `0x0D2`. Earlier dependencies may have changed this field before the read.

| Mode field | Accepted unsigned quantity | Derived count |
|---|---|---|
| Zero | Strictly less than `0x20000`, decimal 131,072 | Quantity divided by two, rounded down |
| Any nonzero representation | Strictly less than `0x40000`, decimal 262,144 | Quantity divided by four, rounded down |

The quantity comparisons are unsigned. Equality with the selected bound is rejected. A quantity with its highest bit set is therefore too large in either mode, not a negative value eligible for acceptance.

Only zero versus nonzero matters for the mode. A 16-bit representation that would be negative under signed interpretation still selects the nonzero branch.

The divisions are unsigned right shifts. There is no rounding up, divisibility check, or minimum quantity:

- Odd quantities are permitted in zero mode.
- Quantities not divisible by four are permitted in nonzero mode.
- Zero is permitted in either mode.
- A positive accepted quantity smaller than the selected divisor produces count zero.

For every accepted quantity, the derived count is between zero and 65,535 inclusive. The destination write nevertheless occupies 32 bits.

### 5. Reject an oversized quantity

If the quantity fails its selected bound, the function reads the signed diagnostic level through the retained diagnostic-control address.

If that level exceeds one, it invokes F-4457a9a44a68 with the oversize-related message address. This address begins four bytes after the static location used for the second selector.

After any such diagnostic returns normally, the function returns 8.

On this path, the enclosing function does not write the derived count and does not invoke F-977b73bb902a. This does not guarantee that the count or other object fields are unchanged: earlier callbacks and diagnostic calls have unknown mutation behavior.

### 6. Write the count and invoke the loader

For an accepted quantity, the function writes the derived count as an unsigned 32-bit value at object offset `0x2B4`.

It then invokes F-977b73bb902a with:

1. Context.
2. Accepted, undivided unsigned 32-bit quantity.
3. Address at object offset `0x2B8`.

The loader receives the original accepted quantity, not the derived count or a quantity rounded to an element-width multiple. A zero derived count does not suppress the loader call.

If F-977b73bb902a returns nonzero, the function immediately returns that status unchanged. The count write occurred before the call, and the enclosing function performs no restoration afterward. The count field’s value after the dependency returns is not guaranteed: the dependency may have overwritten it.

If F-977b73bb902a returns zero, the function reads the diagnostic level through the retained diagnostic-control address. A level greater than one causes a completion-related call to F-4457a9a44a68. The function then returns zero.

Neither the loader nor the completion diagnostic is established to preserve the count field. Thus, even on overall success, the earlier count write does not establish the field’s value at function return.

## Side effects

The enclosing function explicitly performs:

- Initialization of local 32-bit quantity storage.
- A 32-bit object-field write at `0x2B4` on the accepted path.
- Calls that expose object-relative destinations and local writable storage to dependencies.

The count write precedes the loading call. There is no explicit count restoration after loader failure, no post-loader count rewrite, and no verification that the count retains its written value.

Dependency effects are not bounded by their apparent output arguments. Both callbacks receive the object address and context. F-977b73bb902a receives context and an interior object address. Diagnostic calls may also affect state. None is assumed to preserve globals, unrelated object fields, or previously written values.

No allocation, release, ownership transfer, or cleanup operation is directly visible in this function. Dependency allocation and release behavior remain unknown.

The second callback receives the address of local storage. That storage is available during the invocation; the evidence does not authorize retaining or using its address after this function returns. No asynchronous use is established.

### Preconditions and abnormal behavior

Reached accesses require valid memory of the indicated widths, valid callable entries, and suitable object lifetimes. The enclosing function performs no visible null, size, alignment, or callback-validity checks.

The global diagnostic-control pointer slot is read even when the first callback fails. Its pointee is read only on paths reaching diagnostic-level checks.

The retained diagnostic-control object must remain readable for any later rejection or completion check. Replacing the global pointer does not redirect those later checks. If a dependency invalidates the retained object, no recovery is visible.

Invalid accesses, dependency faults, and nonlocal exits are not mapped to the ordinary result codes.

## Dependencies

### Dynamically selected callback

No separate opaque function ID is supplied for the callable read from object offset `0x1F8`. Its identity is selected independently at each invocation.

The enclosing function interprets its result as a 32-bit status:

- First invocation: nonzero is returned unchanged.
- Second invocation: nonzero is replaced by 144.

Selector meanings, destination layouts, mutation scope, ownership effects, and resource behavior are documented_only.

### F-4457a9a44a68

Receives a diagnostic-message address at three possible logical stages:

- After the first request succeeds.
- After quantity rejection.
- After the loader succeeds.

The rejection and completion stages are mutually exclusive. Consequently, one ordinary execution makes at most two diagnostic calls.

Any return value is ignored. Output mechanism, exact message bytes, global mutations, object mutations, and resource effects are documented_only.

### F-977b73bb902a

Receives context, the undivided accepted quantity, and the address at object offset `0x2B8`.

Zero permits the completion path; nonzero is propagated unchanged. Destination layout, allocation behavior, ownership, partial-failure effects, and mutation of the count or other state are documented_only.

## Acceptance examples

These are conditional static expectations, not executed tests. They assume valid reached memory and normally returning dependencies. Count-write expectations describe the enclosing function’s write, not an unconditional final field value.

1. **First request fails.**  
   If the first callback returns 23, the function reads the global diagnostic-control pointer and then returns 23. There is no second request or loader call.

2. **Second request fails.**  
   If the first callback succeeds and the second returns −1, the result is 144. No mode read or explicit count write follows.

3. **Quantity storage is unchanged.**  
   If the initial quantity is 10 and the second callback succeeds without modifying its storage, zero mode produces a count write of 5; nonzero mode produces a count write of 2. The loader receives quantity 10.

4. **Zero-mode upper accepted edge.**  
   With zero mode and quantity 131,071 at validation, the function writes 65,535 and calls the loader with 131,071.

5. **Zero-mode rejected boundary.**  
   With zero mode and quantity 131,072, the function returns 8 without its count write or a loader call. The rejection diagnostic occurs only when the retained object’s signed level exceeds one.

6. **Nonzero-mode upper accepted edge.**  
   With mode `0xFFFF` and quantity 262,143, the function writes 65,535 and calls the loader with 262,143. Neither the mode’s signed interpretation nor the quantity’s remainder prevents acceptance.

7. **Nonzero-mode rejected boundary.**  
   With nonzero mode and quantity 262,144, the result is 8 without a loader call.

8. **Small quantities still load.**  
   With nonzero mode and quantity 3, the function writes zero and calls the loader with 3. Quantity zero likewise passes either mode’s bound and reaches the loader.

9. **Loader failure establishes ordering, not persistence.**  
   With zero mode and quantity 10, the function writes 5 before invoking F-977b73bb902a. If that dependency returns 37, the function returns 37 without restoring the count. If the dependency overwrote the count, the enclosing function does not undo that overwrite; no final count value is guaranteed.

10. **Callback replacement is observed.**  
    If the first callback changes the callback slot and returns zero, the second invocation uses the slot’s subsequently read contents, including any intervening diagnostic changes.

11. **Diagnostic thresholds are signed.**  
    At any applicable level read, one suppresses the diagnostic, two enables it, and a negative signed value suppresses it.

12. **Later diagnostics use the retained address.**  
    If the second callback replaces the global diagnostic-control pointer, a later rejection or completion check still reads through the address retained after the first callback, assuming that object remains valid.

## Uncertainties

The following remain documented_only:

- Exact selector values and diagnostic-message bytes.
- Meaning and width of the first callback’s destination.
- Loader destination layout and ownership.
- Context interpretation and quantity units.
- Whether either callback is expected to write its supplied destination on success.
- Dependency effects on globals, object fields, local storage, resources, and lifetimes.
- The count field’s value after the loader or a subsequent diagnostic returns.
- Alignment requirements and behavior for invalid addresses.
- Additional caller-side invariants.

The location-table interpretation and two-byte/four-byte element interpretation are inferred intent, not extra acceptance requirements.

## Evidence

Established static evidence includes argument forwarding, access offsets and widths, separate callback-slot reads, diagnostic-pointer selection, call and branch ordering, strict unsigned bounds, unsigned shifts, the pre-loader count write, and status propagation.

The representation establishes no restoration by the enclosing function after loader failure. It does not establish persistence of the count write across dependency calls.

No dependency implementations, runtime traces, or external format definitions were supplied. No runtime tests have been performed. The acceptance examples express conditional consequences of the supplied control flow, not observed executions.
