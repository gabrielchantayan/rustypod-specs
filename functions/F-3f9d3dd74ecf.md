<!-- Trusted provenance: F-3f9d3dd74ecf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3f9d3dd74ecf

## Purpose

F-3f9d3dd74ecf obtains context-dependent construction inputs, prepares a local record, requests a resource, and passes those results to an object-construction dependency. It then performs three operations on the returned object and returns that same object pointer.

Established behavior consists of argument forwarding, dependency-call order, local-field writes, indirect dispatch, and a late context read followed by character conversion. The domain meanings of the object, context, selected input, and final adjustment are not established.

The routine has no visible validation, early failure return, recovery branch, or explicit cleanup. Normal completion depends on the invoked operations returning normally and on the addresses subsequently accessed being valid.

## Interface

The routine accepts five positional arguments:

| Position | Representation | Established use |
|---|---|---|
| 1 | 32-bit value | Not read or forwarded. |
| 2 | 32-bit value | Forwarded unchanged as construction argument 3. |
| 3 | 32-bit value | Forwarded unchanged as construction argument 4. |
| 4 | 32-bit integer used as a context address | Passed to F-304d68214672; later used for 32-bit reads at byte offsets `0x5c` and `0x60`. |
| 5 | 32-bit value | Forwarded unchanged to the indirectly selected object operation. |

The return value is the object pointer returned by F-014e927798f9. Results from the subsequent operations do not replace it.

The context must permit full 32-bit reads beginning at offsets `0x5c` and `0x60` when those reads occur. The final character conversion does not reduce the second access to a one-byte read.

The returned object must support reading a dispatch-table reference from its beginning. That reference must support locating a callable entry at byte offset `0x11c`, decimal 284. The supplied representation is consistent with 32-bit addresses and words; the complete ABI and callable-entry representation remain undocumented.

The routine uses these distinct local regions:

- An initial temporary region passed to F-304d68214672. The routine later writes 32-bit fields at offsets 4 and 8 relative to that passed address.
- A result area whose first two adjacent 32-bit words are subsequently consumed as a selector and a companion value.
- A 96-byte construction-record region.
- A separate 64-byte area passed to F-ed03cb1b6aa9.

These addresses refer to call-local storage. No lifetime beyond this invocation is established.

## Behavior

### 1. Obtain context-dependent results

First, invoke F-304d68214672 with the context argument and the initial temporary-region address.

After that call returns, read the 32-bit context word at offset `0x5c`. Invoke F-ccabec065f5e with that word and the result-area address.

The context word is therefore read after F-304d68214672, not captured at routine entry. Any changes that dependency makes to the context can affect the supplied word.

The routine subsequently reads the first result-area word as the selector. It also consumes the adjacent word at offset 4 as a companion value. The supplied routine does not explicitly initialize either word before the dependency calls. Their production, validity, and broader layout require dependency contracts.

### 2. Choose the construction input

The selector determines exactly one dependency path:

- **Selector equal to zero:** invoke F-9d0ecc5f6adf without arguments. Its 32-bit result becomes the selected construction input.
- **Selector not equal to zero:** invoke F-283eda65a14c with the selector. Invoke F-682adfd5a561 with that result and literal `5`. The latter result becomes the selected construction input.

The branch tests equality to zero, not a signed ordering. Every nonzero 32-bit pattern takes the second path, including a pattern with its high bit set.

The routine does not interpret the selected result as a success indicator. A zero result is forwarded without a separate failure branch.

### 3. Prepare temporary fields and the construction record

After the selection path finishes, the routine performs the following ordered operations:

1. Read a 32-bit global value and retain it as the saved construction value.
2. Read the companion word and write it at offset 8 in the initial temporary region.
3. Read a second 32-bit global value and write it at offset 4 in that region.
4. Read a source value from a third global location and invoke F-unknown-01 with the construction-record address, that source value, and literal `0x60`, decimal 96.

F-unknown-01 is a document-local opaque label for the external operation whose supplied identifier is unavailable. Its argument shape suggests a 96-byte transfer into the local record. An actual copy contract is not established by the call alone.

After F-unknown-01 returns, the routine performs these explicit 32-bit writes, in this order:

| Write order | Record byte offset | Written value |
|---|---:|---|
| 1 | `0x10` | Saved construction value |
| 2 | `0x08` | Companion word, read again at this point |
| 3 | `0x0c` | Literal `5` |

The companion word used at temporary-region offset 8 and the companion word used at record offset `0x08` are separate reads. F-unknown-01 runs between them. Equality of the two observed values is not guaranteed without an applicable preservation contract.

The saved construction value is not reloaded from its global location. Its retained value supplies both the explicit write at record offset `0x10` and the later sixth construction argument.

**Record contents must be described at their execution points.** If F-unknown-01 performs an ordinary 96-byte copy, that copy establishes the corresponding record bytes when it completes. The three subsequent writes then replace bytes `0x08` through `0x13` as specified. Under that copy contract, immediately after the final explicit write, the other record bytes retain their copied contents.

That does not, by itself, establish the record contents when construction begins. F-8157a180d901 runs between these writes and construction. F-unknown-01 has already received the record address, and its retention or publication behavior is unknown. F-8157a180d901 could therefore reach and modify the record through shared or retained state. Any assertion that the prepared contents persist until construction requires an additional preservation contract covering this interval.

### 4. Request a resource and construct the object

Invoke F-8157a180d901 with literal `0x17c`, decimal 380.

Next, invoke F-014e927798f9 with six arguments:

1. The result of F-8157a180d901.
2. The selected construction input.
3. Original argument 2.
4. Original argument 3.
5. The construction-record address.
6. The saved construction value.

The literal 380 suggests a storage-size request, but allocation, initialization, ownership transfer, and release obligations are not established.

Neither the resource-request result nor the construction result is checked for zero. The construction result becomes the object pointer used by all remaining operations and eventually returned.

The fifth argument identifies the same local record region previously passed to F-unknown-01. It does not imply an immutable snapshot of that region. The sixth argument independently forwards the retained saved value, regardless of whether the corresponding record field has subsequently changed.

### 5. Operate on the constructed object

Three operations follow, in order.

**First:** invoke F-ed03cb1b6aa9 with the object pointer, the separate 64-byte local-area address, and literal zero.

No explicit initialization of that local area appears before this call. Its direction—input, output, or workspace—and any initialization requirement are undocumented.

**Second:** obtain the dispatch-table reference from the beginning of the object, locate the callable entry at table byte offset `0x11c`, and invoke F-unknown-02 with the object pointer and original argument 5.

F-unknown-02 is a document-local opaque label for the dynamically selected operation; it does not identify one fixed target.

The table lookup occurs after F-ed03cb1b6aa9 returns. A table reference obtained before that call would not preserve the established ordering if the call changes the object or its dispatch state.

**Third:** read the full 32-bit context word at offset `0x60`, convert it to a character-sized value, widen that value to an integer, and invoke F-ad87890b6117 with the object pointer and converted integer.

This context read occurs after the indirect operation returns. It can therefore observe changes made by any preceding dependency.

The character type’s width, signedness, and treatment of out-of-range values are not established. A universally unsigned-byte interpretation is not justified.

Finally, return the construction-result pointer.

## Side effects

The routine directly establishes:

- Two 32-bit context reads, separated by the intervening calls and preparation work.
- Reads from three global locations during construction preparation.
- Reads and writes involving call-local storage.
- The listed dependency calls and one dynamic dispatch.
- A read of the object’s dispatch reference and the selected dispatch entry.

No direct context or global write is visible. This is not a guarantee that those locations remain unchanged: dependencies may modify them.

The record-address exposure is important. F-unknown-01 receives that address before F-8157a180d901 runs. The routine neither establishes non-retention nor prevents the later dependency from accessing the record through shared state. Thus, even correctly executed explicit field writes are not a guarantee of field values at construction entry.

No explicit release operation appears for the resource-request result, selected input, constructed object, or other dependency-produced resources. Allocation and ownership interpretations remain undocumented.

Dependencies receive pointers into local storage. Whether they consume contents synchronously, copy them, retain addresses, or publish those addresses is unknown. Safe use after this routine returns requires an appropriate lifetime contract; it does not follow from the visible calls.

The routine contains no local handling for invalid pointers, unavailable resources, construction failure, invalid dispatch entries, or abnormal dependency termination. If a dependency does not return normally, subsequent actions and the final return are not established. No rollback is visible.

## Dependencies

| Opaque function ID | Established invocation |
|---|---|
| F-304d68214672 | Context and initial temporary-region address; first call. |
| F-ccabec065f5e | Word read from context offset `0x5c` and result-area address. |
| F-9d0ecc5f6adf | No arguments; supplies the zero-selector path’s selected input. |
| F-283eda65a14c | Nonzero selector; supplies an intermediate result. |
| F-682adfd5a561 | Intermediate result and literal `5`; supplies the nonzero path’s selected input. |
| F-unknown-01 | Record address, globally obtained source value, and literal `0x60`; external operation with unavailable identifier. |
| F-8157a180d901 | Literal `0x17c`; runs after explicit record writes and before construction. |
| F-014e927798f9 | Six construction arguments; returns the eventual result pointer. |
| F-ed03cb1b6aa9 | Object pointer, 64-byte local-area address, and zero. |
| F-unknown-02 | Dynamically selected operation receiving object pointer and original argument 5. |
| F-ad87890b6117 | Object pointer and the converted late context value. |

F-unknown-01 and F-unknown-02 are labels used only within this contract, not supplied dependency identities.

No unavailable dependency is assumed to preserve globals or exposed local regions, allocate or release memory, adopt ownership, or return a valid object.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests.

1. **Zero selector.** Given a valid zero selector and normally returning dependencies, F-9d0ecc5f6adf supplies the selected input. Neither F-283eda65a14c nor F-682adfd5a561 is invoked.

2. **High-bit nonzero selector.** A selector with bit pattern `0xffffffff` takes the nonzero path. F-283eda65a14c receives that pattern, and F-682adfd5a561 receives its result and `5`.

3. **Copy and writes at their actual points.** If F-unknown-01 performs an ordinary 96-byte copy, the record matches that transfer when it completes. After the three explicit writes finish, offsets `0x08`, `0x0c`, and `0x10` contain the second companion read, `5`, and the saved construction value respectively; the remaining bytes retain the copied contents at that point. These same contents are required at construction entry only if an additional contract preserves them across F-8157a180d901 and any other permitted intervening access.

4. **Saved value versus mutable record field.** Suppose the saved construction global initially contains a value A and changes to B during F-unknown-01. The explicit write at offset `0x10` uses A, and construction argument 6 is A. If F-8157a180d901 subsequently changes that record field to C through an exposed address, the construction call can receive a record containing C while still receiving A as argument 6.

5. **Distinct companion observations.** If F-unknown-01 changes the companion word through an available alias, the earlier temporary-region write uses the earlier observation, while record offset `0x08` is explicitly written using the later observation.

6. **Late context observation.** If the indirect operation changes the word at context offset `0x60`, the final conversion uses the changed word, provided the context remains valid and readable.

7. **Conditional character conversion.** If the target uses an 8-bit character conversion that retains the low eight bits, an input ending in `0xff` produces `255` when widened from an unsigned character. Under a conversion that interprets those retained bits as signed negative one, it produces `-1`. Neither conversion rule is selected unconditionally by this contract.

8. **Zero resource result.** If F-8157a180d901 returns zero normally, zero is still passed as construction argument 1. No early return is introduced.

9. **Null construction result.** A null construction result is passed to F-ed03cb1b6aa9 without a local check. Normal completion is not established, because subsequent dispatch requires object access.

10. **Return identity.** If every required operation completes normally, the returned pointer equals the construction result. Later dependency return values do not replace it.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the context, selector, companion value, globals, literals, and final adjustment.
- Full temporary-result layouts and which dependency initializes each consumed word.
- Actual transfer semantics of F-unknown-01.
- Retention, publication, aliasing, and mutation of the exposed record address.
- Preservation of record contents between the explicit writes and construction.
- Resource allocation, ownership adoption, copying, release, and lifetime obligations.
- Input/output direction and initialization requirements of the 64-byte local area.
- Exact character conversion, endianness, address representation, alignment requirements, and calling convention.
- Failure behavior and other effects of unavailable dependencies.

Offsets and literal sizes fit within 32 bits. The routine performs no variable-size allocation calculation and shows no checked address arithmetic. Valid context-offset accesses, table-entry addressing, alignment, and object lifetime are preconditions for the relevant operations; neither overflow handling nor safe address wrapping is established.

## Evidence

Established claims derive from static inspection of the supplied routine representation: argument widths, fixed offsets, branch conditions, explicit read/write order, dependency-call order, and return identity.

Allocation, template copying, and domain-level initialization are inferred interpretations rather than established dependency behavior. In particular, an ordinary copy contract establishes contents at the copy’s execution point; it does not establish preservation across the later resource request.

Missing dependency and ABI contracts remain explicitly unresolved. No runtime tests have been performed.
