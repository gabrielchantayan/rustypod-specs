<!-- Trusted provenance: F-376abbbf47ef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-376abbbf47ef

## Purpose

F-376abbbf47ef conditionally creates and publishes an object into an indexed slot associated with a caller-supplied base address.

Its normal-return behavior has three outcomes:

- Return **2** when a preliminary dependency returns a nonzero result.
- Return **25** when that preliminary result is zero but the selected construction dependency returns a zero object address.
- Return **0** after storing a nonzero construction result into the selected slot and invoking a post-publication dependency.

The function also acquires a cleanup value through an entry dependency and passes that value to a release-like dependency on every visible normal-return path.

These are established control-flow facts. Descriptions such as “slot availability check,” “constructor,” “allocation,” and “release” express inferred roles; the dependencies’ internal behavior is not supplied.

## Interface

### Inputs

The interface has nine positional inputs. The supplied representation uses 32-bit integer or opaque-word values throughout.

| Position | Contract name | Width and interpretation |
|---|---|---|
| 1 | Base address | 32-bit address-bearing value. Used to derive an entry-dependency address, passed to other dependencies, and used as the base of the publication store. |
| 2 | Payload A | Opaque 32-bit word forwarded to the selected construction dependency. A local copy is established before the first dependency call. |
| 3 | Payload B | Opaque 32-bit word forwarded to the selected construction dependency. |
| 4 | Payload C | Opaque 32-bit word forwarded to the selected construction dependency. |
| 5 | Payload D | Opaque 32-bit word forwarded to the selected construction dependency. |
| 6 | Payload E | Opaque 32-bit word forwarded to the selected construction dependency. |
| 7 | Slot index | 32-bit integer used by the preliminary dependency and in the indexed publication address. No local range check is visible. |
| 8 | Payload F | Opaque 32-bit word forwarded to the selected construction dependency. |
| 9 | Variant selector | 32-bit integer tested only for equality with zero. Zero selects one construction path; every nonzero value selects the other. |

No string encoding, buffer length, ownership annotation, or payload-specific structure is established for the opaque words.

### Output

The function returns a 32-bit status word:

| Value | Established meaning |
|---|---|
| **0** | The selected construction dependency returned a nonzero address; that address was stored into the selected slot; the post-publication and cleanup dependencies subsequently returned. |
| **2** | The preliminary dependency returned nonzero; cleanup subsequently returned. |
| **25** | The preliminary dependency returned zero, the selected construction dependency returned zero, and cleanup subsequently returned. |

These meanings describe completed visible sequences, not independent guarantees about the resulting object’s validity or the dependencies’ success.

### Addressable layout

Two locations are derived from the base address:

- **Entry-dependency address:** base address plus **28 bytes**.
- **Publication slot address:** base address plus **4 × slot index** bytes.

The publication store writes a **32-bit value** containing the nonzero address returned by the selected construction dependency.

The supplied body does not establish the containing object’s total size, slot count, field boundaries, alignment guarantees, or whether the two derived locations can overlap.

### Preconditions

For a defined, successful execution, the caller and dependencies must collectively make every executed memory access and address calculation valid. In particular:

- The entry dependency must accept the address derived by adding 28 bytes to the base.
- On a publication path, the indexed slot must be writable for a 32-bit store.
- The index and base must permit the indexed address calculation under the target’s actual arithmetic and addressing rules.
- Dependencies must accept their supplied inputs and return normally if one of the documented status results is expected.

There is no visible local validation of a null base, a negative index, an oversized index, address overflow, or slot alignment.

## Behavior

### Entry processing

Before the first dependency call, the function establishes local copies of the base address and the first three payload words.

It then calls **F-bb2fbec1af57** with:

1. An address of local storage intended to receive a cleanup value.
2. The base address plus 28 bytes.

A value from that local storage is later supplied to **F-fd58bea73663**. The body does not itself initialize that value before the entry call. Consequently, valid cleanup use depends on the entry dependency supplying a usable value or on some other dependency behavior not established here.

This arrangement resembles entering and leaving a protected region, but neither locking nor restoration of state is established.

### Preliminary decision

After the entry dependency returns, the function calls **F-75684af4daa6** with:

1. The base address.
2. The slot index.
3. The literal value **1**.

Only the distinction between zero and nonzero matters. The specific nonzero value is not propagated.

If the result is nonzero:

- Neither allocation path is entered.
- Neither construction dependency is called.
- No direct publication store is performed by this function.
- The cleanup dependency is called with the saved cleanup value.
- The function returns **2**, provided cleanup returns.

This does not prove that the slot is occupied or that no state changed. Those interpretations depend on the preliminary dependency’s contract.

### Variant selection and allocation

If the preliminary result is zero, the variant selector determines exactly one construction path.

For a **zero selector**:

- **F-8157a180d901** is called with **88**.
- Its returned 32-bit value is passed as the first argument to **F-9b75370a8671**.

For a **nonzero selector**:

- **F-8157a180d901** is called with **96**.
- Its returned 32-bit value is passed as the first argument to **F-50a8085cd9b2**.

The values 88 and 96 are established request arguments. Treating them as byte allocation sizes is inferred from the surrounding usage, not proven by the supplied dependency implementations.

There is **no direct zero-result check** between the allocation-like call and the construction call. A zero result from F-8157a180d901 is still passed to the selected construction dependency. The function’s subsequent success decision depends on the construction result, not directly on the allocation-like result.

Both construction dependencies receive seven arguments in the same order:

1. The value returned by F-8157a180d901.
2. Payload A.
3. Payload B.
4. Payload C.
5. Payload D.
6. Payload E.
7. Payload F.

The base address, slot index, and variant selector are not supplied as separate arguments to these construction calls.

### Construction failure

If the selected construction dependency returns zero:

- The function performs no direct publication store.
- It does not call the post-publication dependency.
- It calls **F-fd58bea73663** with the saved cleanup value.
- It returns **25**, provided cleanup returns.

There is no visible separate disposal call for the value returned by F-8157a180d901. Whether construction failure consumes, releases, retains, or leaks any allocation is not established.

### Publication and completion

If the selected construction dependency returns nonzero, the function performs these actions in order:

1. Store that returned 32-bit address at the base address plus four times the slot index.
2. Call **F-554ea3a78660** with the base address.
3. Call **F-fd58bea73663** with the saved cleanup value.
4. Return **0**.

The publication store precedes the post-publication call. There is no visible recheck of the preliminary condition immediately before publication, no direct read of the previous slot contents, and no visible preservation or disposal of an overwritten slot value.

The post-publication dependency’s return value does not control the final status. If it returns normally, execution proceeds to cleanup and then returns zero.

## Side effects

### Direct memory effects

The explicit caller-addressed write in the supplied body is the successful-path 32-bit publication store.

The function also writes local working storage and exposes an address within that storage to F-bb2fbec1af57. The extent of memory that dependency may access is not established.

No direct read of the publication slot is visible. Any inspection performed during the preliminary decision belongs to F-75684af4daa6.

### Dependency-mediated effects

Every dependency may have effects beyond those visible in this body. No guarantee is available that any dependency preserves globals, leaves the base object unchanged, avoids callbacks, or refrains from allocation or deallocation.

Thus “no publication store” means no such direct store by this function. It does not mean the invocation leaves caller-visible memory unchanged.

### Ownership and lifetime

On the success path, the construction result is published into the indexed slot. This establishes storage of an address, not an independently proven ownership transfer.

The following remain undocumented:

- Who owns the allocation-like result before or after construction.
- Whether the construction result must equal the allocation-like result.
- Who owns the published object after return.
- Whether construction failure releases anything.
- Whether the cleanup dependency manages a lock, reference, resource, or unrelated state.

On every visible normal-return path, cleanup is called exactly once after the entry call has returned. No exceptional cleanup, unwinding, rollback, or recovery path is shown.

## Dependencies

| Opaque ID | Observed interaction | Interpretation limits |
|---|---|---|
| **F-bb2fbec1af57** | Called first with local output storage and base plus 28 bytes. | Acquisition-like role inferred; output initialization and memory-access extent unknown. |
| **F-75684af4daa6** | Receives base, index, and literal 1. Nonzero ends processing with status 2 after cleanup. | Availability or eligibility checking is inferred, not established. |
| **F-8157a180d901** | Receives 88 or 96; its result is forwarded without a zero check. | Allocation-like role inferred; failure and ownership rules unknown. |
| **F-9b75370a8671** | Selected only for a zero variant selector; receives the allocation-like result and six payload words. | Zero means failure to this caller; other semantics unknown. |
| **F-50a8085cd9b2** | Selected for every nonzero variant selector; receives the same argument arrangement. | Zero means failure to this caller; other semantics unknown. |
| **F-554ea3a78660** | Called with the base after successful publication. | Refresh, notification, or bookkeeping intent unknown; result does not affect status. |
| **F-fd58bea73663** | Receives the saved cleanup value on all visible normal-return paths. | Release-like role inferred; effects and resource ownership unknown. |

## Acceptance examples

These examples are conditional static expectations, not executed tests. They assume valid addresses, suitable arithmetic, and normally returning dependencies.

### Preliminary rejection

If F-75684af4daa6 returns any nonzero value, the expected result is **2** after cleanup. Neither allocation nor construction dependency is called. This function does not perform its publication store.

### Zero-selector success

If the preliminary dependency returns zero, the selector is zero, and F-9b75370a8671 returns a nonzero address:

- The allocation-like request is **88**.
- F-9b75370a8671 receives its result followed by the six payload words.
- The returned address is stored at the indexed slot.
- Post-publication processing precedes cleanup.
- The final status is **0**.

For a valid index of **3**, the publication location is **12 bytes** beyond the base.

### Nonzero-selector construction failure

If the preliminary dependency returns zero, the selector is **−1**, and F-50a8085cd9b2 returns zero:

- The allocation-like request is **96**.
- The nonzero-selector construction dependency is selected despite the selector being negative.
- No direct publication store or post-publication call occurs.
- Cleanup precedes a return of **25**.

### Zero allocation-like result

If F-8157a180d901 returns zero, construction is still attempted with zero as its first argument.

A zero construction result leads to **25** after cleanup. A nonzero construction result leads to publication and **0**, assuming subsequent operations complete normally.

### Different allocation and construction addresses

If the allocation-like dependency returns one address and construction returns a different nonzero address, the **construction result** is stored in the slot. No equality check is visible.

### Out-of-range index

A negative or excessively large index is not locally rejected. If the preliminary dependency returns zero, construction can occur before the indexed store is attempted. No safe error result is established for an invalid publication address.

## Uncertainties

The following are **documented_only**:

- Exact target ABI, pointer model, alignment requirements, and overflow behavior.
- Valid index range and containing object layout.
- The relationship between indexed slots and the location at byte offset 28.
- Dependency effects on globals, shared memory, callbacks, or synchronization.
- Initialization guarantees and valid values for the cleanup output.
- Allocation failure behavior and construction failure ownership.
- Whether post-publication processing may alter or remove the published value.
- Behavior when any dependency faults, throws, fails to return, or transfers control elsewhere.
- Whether status values have domain-specific names beyond their observed control-flow meanings.

The four-byte slot stride and 32-bit stored value are established. They do not prove that every index denotes a valid slot. No wraparound, saturation, bounds enforcement, or safe recovery rule should be assumed for the address arithmetic.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes call order, branch conditions, argument forwarding, request constants, the indexed 32-bit publication store, and the three returned status values.

Allocation, construction, synchronization, and notification terminology is inferred from usage and does not replace missing dependency contracts.

No runtime tests have been performed. The acceptance examples specify conditional expectations rather than observed execution results.
