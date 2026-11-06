<!-- Trusted provenance: F-c0aad31f3bdb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c0aad31f3bdb

## Purpose

F-c0aad31f3bdb conditionally dispatches an input record to one of three object-associated handlers. A selected handler can stop further processing by returning a nonzero 32-bit result. Otherwise, the function obtains a fallback receiver and, when that receiver is non-null, forwards the original record address to it.

The established behavior is dispatch and fallback, not any particular event-processing semantics. Interpreting the record as an event, the first argument as an object, or the handler result as “handled” is inferred intent.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Arguments and result

The function accepts two address-valued arguments:

- **Primary object address:** points to an object whose first machine word supplies the address of an indirect-call table.
- **Record address:** points to a record containing the 32-bit fields inspected during dispatch.

The function has no explicit return value. Results from selected handlers affect control flow but are not returned to the caller.

The supplied representation uses 32-bit integer accesses and a 32-bit address/table model. Pointer validity, alignment requirements, and calling-convention details are not independently established.

### Relevant layouts

Only the following locations are established as relevant:

| Base | Byte offset | Interpretation |
|---|---:|---|
| Primary object | `0x00` | Address of its indirect-call table |
| Record | `0x04` | 32-bit record kind |
| Record | `0x08` | 32-bit selector for the discriminator-dependent routes |
| Record | `0x0C` | 32-bit selector for the kind-`0x13` route |
| Primary call table | `0xCC` | Fallback-provider target |
| Primary call table | `0xDC` | First selected-handler target |
| Primary call table | `0xE8` | Second selected-handler target |
| Primary call table | `0xEC` | Third selected-handler target |
| Fallback receiver | `0x00` | Address of its indirect-call table |
| Fallback call table | `0x30` | Forwarding target |

Offsets are byte offsets. Table offsets identify locations containing callable addresses, not inline instructions.

The function also consults a global 32-bit dispatch discriminator. Its value and initialization are not supplied.

### Preconditions

The record must permit a 32-bit read at offset `0x04`. Additional record fields must be readable when their corresponding branch is reached.

The primary object and the particular call-table entry used by the chosen path must be valid. There is no explicit null check for the primary object, its table address, or any callable target.

A non-null fallback receiver must likewise provide a valid first-word table address and a callable entry at offset `0x30`.

If a selected handler returns zero, the primary object must remain usable for the subsequent fallback-provider lookup. The function does not establish this condition itself.

For the ordinary route descriptions below, the global dispatch discriminator is assumed stable during route selection. Behavior when its repeated observations differ is separately classified under uncertainties.

## Behavior

### Initial record inspection

The first established memory access is a 32-bit read of the record kind at offset `0x04`.

That kind is retained for route selection. The function does not explicitly reload it after reading another record field.

No primary-object table lookup occurs before the record classification work for the chosen route.

### First route: kind and trailing selector

If the retained kind equals `0x13`, the function reads the 32-bit field at record offset `0x0C`.

If that field equals `0x00008223`, the first route is selected. The function:

- Reads the primary object’s table address.
- Loads the callable target from table offset `0xDC`.
- Calls that target with the primary object address and the original record address, in that order.

This route has priority over both discriminator-dependent routes. A successful first-route match does not consult the global dispatch discriminator or read the selector at offset `0x08`.

If the kind is not `0x13`, the field at offset `0x0C` is not read by this route. If the kind is `0x13` but the field does not match, route selection continues.

### Second route: discriminator and selector `0x00040001`

After the first route fails, the retained kind is compared with the global dispatch discriminator.

If they match, the function reads the 32-bit field at record offset `0x08`. A value of `0x00040001` selects the second route:

- Read the primary object’s table address.
- Load the target from table offset `0xE8`.
- Call that target with the primary object address and the original record address.

A second-route match prevents consideration of the third route.

### Third route: discriminator and selector `0x00010002`

If the second route is not selected, the function again compares the retained kind with the global dispatch discriminator.

When that comparison matches, it reads the 32-bit selector at record offset `0x08` and derives a comparison value by subtracting `0x00010000`. A further comparison against the global discriminator precedes the final selector decision.

Under a stable discriminator, the third route is selected exactly when:

- The retained kind equals that discriminator; and
- The selector is `0x00010002`.

The selected target is obtained from primary-table offset `0xEC` and called with the primary object address and original record address.

The subtraction-based condition does not describe a range of accepted selectors. It identifies the single 32-bit selector value `0x00010002`.

### Repeated reads during classification

The discriminator-dependent paths do not establish a single cached observation of the global discriminator.

With a stable discriminator equal to the retained kind, and an initial selector that does not select the second route, the function can read record offset `0x08` twice:

1. Once for the second-route decision.
2. Again for the third-route decision.

Thus, the record selector is not established as a snapshot across those decisions. If externally modified between the reads, the later value can govern third-route selection.

The retained record kind, by contrast, comes from the initial read.

### Selected-handler result

At most one of the three selected-handler call sites is invoked.

The selected handler’s result is evaluated as a 32-bit integer:

- **Nonzero:** return immediately.
- **Zero:** continue to fallback.

No particular nonzero value is required. In particular, the branch is not limited to a result of one or to positive values.

A zero result does not cause another selected handler to be tried. Route selection is finished before that result is evaluated.

### Fallback

Fallback occurs either because no route was selected or because the selected handler returned zero.

The function reads the primary object’s first word, obtains the callable target at table offset `0xCC`, and invokes it. No explicit argument list is established for this provider call. It must not be specified as receiving either incoming argument merely because those arguments are available in the surrounding function.

The provider result is interpreted as a fallback-receiver address:

- If null, return without forwarding.
- If non-null, read that receiver’s first-word table address, load the target at table offset `0x30`, and call it with the fallback receiver address and original record address.

The forwarding call’s result is not inspected. After it returns, F-c0aad31f3bdb returns.

When fallback follows a selected-handler call, the primary table is looked up after that call. The function does not reuse the earlier table address or assume that the handler preserved it.

## Side effects

### Direct effects

The established body performs memory reads and indirect calls. It contains no explicit store to the primary object, record, global discriminator, or fallback receiver.

It contains no explicit allocation, deallocation, reference-count operation, or ownership transfer.

Those observations do not establish that the overall operation is read-only. Any invoked dependency may modify memory or global state.

### Ordering and visibility

The following ordering is established for ordinary returning calls:

1. Read the record kind.
2. Perform the record and discriminator reads required for route selection.
3. If selected, resolve and invoke one selected handler.
4. If fallback is required, resolve and invoke the fallback provider.
5. If the provider returns a non-null receiver, resolve and invoke its forwarding target.

Consequences include:

- Mutations made by a selected handler can affect the subsequent primary-table lookup.
- Mutations made by the provider can affect the fallback receiver and its table.
- Forwarding uses the original record address, not a record copy.
- The forwarded record’s contents may differ from the contents initially inspected.

There is no explicit rollback if a handler returns zero after modifying state.

### Ownership and lifetime

Passing an address does not establish transfer of ownership. The function neither explicitly acquires nor explicitly releases either incoming argument.

The fallback receiver’s ownership and lifetime are unknown. No release operation is visible after forwarding, but that does not prove a leak or establish borrowed ownership.

A selected handler that invalidates the primary object and then returns zero would leave the subsequent lookup without an established valid object. A provider returning an invalid non-null receiver would likewise leave forwarding without established valid storage.

### Errors and abnormal exits

No explicit error code, diagnostic, validation failure, or recovery path is present.

Null callable targets, invalid tables, inaccessible record fields, and failures inside dependencies are not handled by an established local mechanism. Their consequences depend on the execution environment and dependency behavior.

The immediate-return descriptions presume that the relevant dependency returns normally.

## Dependencies

These opaque IDs are local specification labels for indirect call sites. They do not identify recovered implementations or prove that different table entries contain different target addresses.

| Opaque function ID | Resolution | Explicit inputs | Result used here |
|---|---|---|---|
| F-dep01 | Primary table, offset `0xDC` | Primary object, record | 32-bit zero/nonzero decision |
| F-dep02 | Primary table, offset `0xE8` | Primary object, record | 32-bit zero/nonzero decision |
| F-dep03 | Primary table, offset `0xEC` | Primary object, record | 32-bit zero/nonzero decision |
| F-dep04 | Primary table, offset `0xCC` | No explicit arguments established | Null or non-null receiver address |
| F-dep05 | Fallback table, offset `0x30` | Fallback receiver, record | Not inspected |

No dependency is established to preserve globals, preserve object layout, avoid reentrancy, or acquire or release memory.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests. Unless stated otherwise, assume valid storage, callable targets, normally returning dependencies, and a stable global discriminator.

### First-route priority

Given kind `0x13` and offset-`0x0C` value `0x00008223`, F-dep01 is invoked with the two incoming addresses.

Even if the discriminator and offset-`0x08` selector could otherwise match another route, neither later selected handler is invoked. If F-dep01 returns nonzero, the fallback provider is not called.

### Second-route selection

Given failure of the first route, kind equal to the discriminator, and offset-`0x08` value `0x00040001`, F-dep02 is invoked.

If it returns zero, processing goes to F-dep04 rather than trying F-dep03.

### Third-route selection

Given failure of the first route, kind equal to the discriminator, and offset-`0x08` value consistently equal to `0x00010002`, F-dep03 is invoked.

Values `0x00010000`, `0x00010001`, and `0x00010003` do not satisfy this selector condition.

### No selected route

Given a kind other than `0x13` that also differs from the discriminator, no selected handler is invoked. Fallback begins directly.

On this path, neither offset `0x08` nor offset `0x0C` needs to be read by this function.

### Failed first-route selector can still reach another route

Given kind `0x13`, an offset-`0x0C` value other than `0x00008223`, discriminator `0x13`, and offset-`0x08` value `0x00040001`, F-dep02 is selected.

The failed first-route selector is not itself grounds for immediate fallback.

### Nonzero is not limited to one

If a selected handler returns the 32-bit value `0xFFFFFFFF`, the function returns without invoking F-dep04. The decision is zero versus nonzero.

### Null fallback receiver

If F-dep04 returns null, the function returns without reading a fallback table or invoking F-dep05.

### Forwarding after mutation

If a selected handler changes the primary object’s table address and returns zero, fallback resolves offset `0xCC` through the table address observed afterward.

If the eventual provider returns a valid receiver, F-dep05 receives the original record address, including any intervening changes to the record’s contents.

## Uncertainties

- **documented_only — discriminator value:** Its constant value, initialization, storage discipline, and intended meaning are unknown.
- **documented_only — changing discriminator:** Repeated comparisons may observe different values if the environment permits intervening mutation. The supplied representation can then expose a temporary-value dependency that is not sufficiently established to define a stable public rule. No atomic snapshot or synchronization guarantee is inferred.
- **documented_only — concurrent record mutation:** Repeated selector reads are visible in the static representation, but supported concurrency semantics are unknown.
- **documented_only — ABI:** The provider call has no explicit arguments in the supplied representation. Register contents or hidden calling-convention inputs are not promoted into interface requirements.
- **documented_only — pointer model:** The representation supports a 32-bit layout description; portability to another pointer width is not established.
- **documented_only — dependency behavior:** Actual targets, side effects, ownership rules, and abnormal exits are unavailable.
- **inferred intent:** The structure resembles prioritized handling followed by delegation. No domain-specific meaning is established for any selector.

## Evidence

The contract is grounded in the supplied static function representation: field offsets, selector constants, indirect-call offsets, argument order where explicit, conditional branches, and return-result checks.

The third selector is derived from the 32-bit condition “selector minus `0x00010000` equals two,” yielding `0x00010002`. This match does not require an assumption about signed ordering or overflow.

No dependency implementations, surrounding type definitions, runtime traces, or execution results were supplied. Comments about indirect control-flow recovery do not establish additional runtime behavior.

No runtime tests have been performed.
