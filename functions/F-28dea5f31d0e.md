<!-- Trusted provenance: F-28dea5f31d0e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-28dea5f31d0e

## Purpose

F-28dea5f31d0e conditionally processes an input through a receiver-dispatch path or a notification path. Before selecting either path, it performs preliminary classification that may cause a separate operation on an embedded context region.

The two main paths are selected by a second, independent classification of the original input:

- Result **1** selects receiver dispatch.
- Result **3** selects notification handling.
- Any other result causes a normal return of **0**.

Preliminary processing may already have caused effects before a return of **0**.

“Receiver,” “notification,” “classification,” and “extraction” are descriptive labels for the observed operations. Their application-domain meanings are not established. A return of **1** identifies completion of a recognized path, not externally verified success of the dependencies invoked along that path.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Inputs and result

The supplied representation exposes two 32-bit inputs:

| Input | Established use |
|---|---|
| Context address | Base for embedded-region addresses, a receiver-pointer field, and a one-byte state field. |
| Input value | Supplied to input classification and, when nonzero, to selected extraction calls. |

The input value’s domain is unknown. It may encode an address, handle, or other value. Zero suppresses extraction at specified sites, but does not suppress input classification.

There is no visible top-level null check on the context address. Required storage accessibility depends on the path actually taken and on dependency behavior.

The result is a 32-bit quantity. Every visible normal return is exactly **0** or **1**:

- **0** indicates an explicit rejection or an initially absent receiver.
- **1** indicates normal completion of a recognized receiver or notification path.

Dependency return values are not propagated directly as this function’s result. No negative error result or exception-to-result translation is visible.

### Context layout

All offsets below are byte offsets from the context address.

| Offset | Established access |
|---|---|
| `0x24` | Embedded-region address passed to F-5bcae2eff9a7 after an accepted notification. |
| `0x28` | One-byte field explicitly written with **1** immediately before that call. |
| `0x40` | 32-bit receiver-pointer field, initially checked and later reloaded on dispatching branches. |
| `0x48` | Embedded-region address passed to F-5bcae2eff9a7 when preliminary classification permits it. |

Passing an embedded-region address is not itself a direct read of that region’s contents. The dependency’s accesses through that address are unknown.

For receiver dispatch, the first 32-bit word at the receiver address supplies a dispatch-table address. A 32-bit call target is selected from that table using the following byte offsets:

| Receiver-selection result | Table offset |
|---|---|
| **6** | `0x0c` |
| **7** | `0x08` |
| **8** | `0x10` |
| **9** | `0x14` |

The representation supports a 32-bit word and pointer model. It does not establish structure sizes, byte order, alignment allowances, or valid address ranges.

### Preconditions

Normal completion requires every address actually accessed to remain valid for the required read, write, or call. In particular, a reloaded receiver and its dispatch table must be usable when dereferenced.

There are no visible bounds checks, alignment checks, table-size checks, or indirect-target checks. The receiver’s initial nonzero check does not establish validity after intervening dependencies, and the later receiver reload is not followed by another zero check.

## Behavior

### Descriptor-source model

Extraction calls use values read from **six global descriptor-source slots**. The slots are distinct storage locations; their contents are not established immutable descriptors.

For this contract, the six slots are labeled **S1** through **S6**. Four extraction forms use them as follows:

| Extraction form | Candidate input | First descriptor-argument source | Second descriptor-argument source |
|---|---|---|---|
| **A: preliminary** | Original input | S1 | S2 |
| **B: receiver intermediate** | Original input | S3 | S4 |
| **C: receiver secondary** | Receiver intermediate | S4 | S5 |
| **D: notification** | Original input | S3 | S6 |

“First” and “second” identify argument positions, not the temporal order of the reads.

For each executed extraction, F-28dea5f31d0e reads the corresponding two globals while preparing that call. The supplied representation does **not** establish which descriptor-source read occurs first. It also does not establish an atomic snapshot across the two reads.

When extraction is skipped because its candidate input is zero, **both descriptor-source reads for that extraction are skipped**, as is the extraction call. The extracted value is instead taken to be zero.

Forms B and D share S3 as their first descriptor-argument source. Form B’s second source and form C’s first source are both S4. These are storage relationships, not guarantees of equal argument values across different calls. Earlier dependencies may modify a slot before a later extraction reads it.

Every visible extraction call supplies the 32-bit all-ones pattern `0xffffffff` after the two descriptor arguments. Whether F-b3bed2dfc5ec interprets this as signed **−1**, an unsigned maximum, or another sentinel is unknown. Descriptor contents, meanings, and declared storage widths are not established.

### 1. Preliminary processing

The function first calls F-de6064227850 with the original input value.

If the result is not exactly **1**, it skips preliminary extraction and the preliminary embedded-region operation.

If the result is exactly **1**:

- A zero input produces a preliminary value of zero, without reading S1 or S2 and without calling F-b3bed2dfc5ec.
- A nonzero input causes reads of S1 and S2 to prepare extraction form A, followed by F-b3bed2dfc5ec with the original input, the two read values, and `0xffffffff`.

These descriptor reads occur after the first F-de6064227850 call returns. Values written to the slots by that dependency may therefore be observed.

The preliminary value, whether extracted or synthesized as zero, is passed to F-118693b436a8. If its result is exactly **6** or **7**, F-5bcae2eff9a7 is called with the context’s `0x48` embedded-region address. Every other result skips that call.

Preliminary processing has no direct normal return from F-28dea5f31d0e. Execution proceeds to the second input classification if all preceding operations return normally.

### 2. Independent second classification

F-de6064227850 is called again with the original input value.

This is a new dependency invocation. The first classification result is not reused, and the function does not require the two results to agree. Preliminary dependencies may have changed state affecting the second result.

The second result selects the remaining behavior:

- **1:** receiver-dispatch path.
- **3:** notification path.
- Any other value: return **0**.

### 3. Receiver-dispatch path

The function first reads the 32-bit receiver-pointer field at context offset `0x40`.

If the field is zero, it returns **0**. No form B descriptor reads, form B extraction, or receiver-selection call occur. Earlier preliminary effects remain possible.

If the field is nonzero, the function obtains an intermediate value:

- A zero original input produces intermediate zero without reading S3 or S4.
- A nonzero original input causes reads of S3 and S4 and an extraction-form-B call to F-b3bed2dfc5ec.

Form B’s descriptor reads occur after the second F-de6064227850 call and after the initial receiver check. All preliminary calls also precede them. The descriptor values are not retained from form A or otherwise fixed by an earlier phase.

The function then calls F-118693b436a8 to select a receiver operation.

**Unresolved argument:** the representation exposes no explicit argument for this receiver-selection call. Its placement after intermediate extraction does not establish that the intermediate value is its effective argument. The following branches are conditional on the result it returns.

#### Results 6, 8, and 9

These three results use the same payload-processing sequence, differing only in the dispatch-table offset.

If the intermediate value is zero, the secondary value is zero, with no reads of S4 or S5 and no form C extraction.

Otherwise, the function reads S4 and S5 and calls F-b3bed2dfc5ec using extraction form C. These reads occur after the receiver-selection call returns. That call, form B extraction, and all earlier dependencies may have modified the slots.

In particular, the S4 value read for form C need not equal the S4 value previously read for form B.

The secondary value is then passed to F-4a882b291036. This call occurs even when the secondary value is zero.

Only after F-4a882b291036 returns does the function:

1. Reload the receiver-pointer field at context offset `0x40`.
2. Read the dispatch-table address from the receiver’s first word.
3. Read the target at the selected table offset.
4. Invoke that target with the reloaded receiver address and the 32-bit result of F-4a882b291036.

The selected offset is `0x0c` for result **6**, `0x10` for **8**, and `0x14` for **9**.

The receiver used for dispatch may differ from the receiver observed by the initial check. No second nonzero check protects the dereference. If the indirect call returns normally, F-28dea5f31d0e returns **1**, without testing its result.

#### Result 7

The function reloads the receiver-pointer field, reads the receiver’s dispatch-table address, and invokes the target at table offset `0x08`.

No form C descriptor reads, form C extraction, or F-4a882b291036 call occur.

The representation exposes no explicit arguments for this indirect invocation. A receiver argument or other machine-level argument must not be inferred solely from the table lookup.

Normal return from the target causes F-28dea5f31d0e to return **1**.

#### Any other result

The function returns **0** without form C descriptor reads, secondary extraction, conversion, or indirect receiver dispatch.

### 4. Notification path

When the second input classification is exactly **3**, the function obtains a notification value:

- A zero input produces zero without reading S3 or S6 and without calling F-b3bed2dfc5ec.
- A nonzero input causes reads of S3 and S6 and an extraction-form-D call.

These descriptor reads occur after the second input-classification call returns and after any preliminary dependencies. They use the then-observed slot contents, not immutable descriptor values.

The notification value is passed to F-6b065a9b9915.

If its result is not exactly **3**, the function returns **0**, with no direct state-byte write and no call on the `0x24` region.

If its result is exactly **3**, the function:

1. Writes the single byte **1** at context offset `0x28`.
2. Calls F-5bcae2eff9a7 with the context’s `0x24` embedded-region address.
3. Returns **1** if that call returns normally.

The byte write precedes the call. The dependency may subsequently change that byte; its final value is not guaranteed by this function alone.

## Side effects

### Direct accesses

The only explicit data write is the one-byte store of **1** at context offset `0x28`.

Direct reads include:

- The two relevant descriptor-source globals for each executed extraction.
- The initial receiver-pointer check on the receiver path.
- The later receiver-pointer reload and dispatch-table accesses on a selected dispatch branch.

Skipped extractions do not read their descriptor-source slots. No global descriptor values are explicitly cached across extractions.

There is no visible direct write to a descriptor-source slot or the receiver-pointer field. There is no visible allocation or explicit release.

### Dependency-mediated effects and ordering

Dependencies and indirect targets may modify globals, context fields, input-associated storage, or receiver state. No preservation guarantee is established.

Important ordering constraints are:

- Preliminary processing precedes the second input classification.
- Each extraction’s descriptor-source reads occur after the preceding dependency calls on that path and before entry to that extraction dependency.
- The initial receiver check precedes form B descriptor reads and extraction.
- Receiver selection precedes form C descriptor reads and extraction.
- Secondary extraction and F-4a882b291036 precede the receiver reload for results **6**, **8**, and **9**.
- The notification state-byte write precedes the call on the `0x24` region.

No relative evaluation order between the two descriptor-source reads within one extraction is established.

A return of **0** therefore does not imply absence of reads, dependency calls, or state changes.

### Ownership, errors, and arithmetic

No ownership transfer or lifetime extension is established for either input, extracted values, descriptor-derived values, the conversion result, or the receiver. Whether dependencies retain, allocate, or release anything remains unknown.

Dependencies, storage, and indirect targets must remain usable through their actual accesses. The function supplies no visible recovery for invalid accesses, nonreturning dependencies, exceptions, or failed indirect calls. Such events are not established to become normal returns of **0**.

Address formation uses fixed byte offsets. No range or overflow checks are visible. Valid-address preconditions are required; portable overflow behavior outside the represented machine model is not established.

## Dependencies

| Opaque function ID | Established use |
|---|---|
| F-de6064227850 | Classifies the original input twice, in separate calls. |
| F-b3bed2dfc5ec | Performs conditional extraction using a nonzero candidate, two freshly read descriptor-source values, and `0xffffffff`; some renderings also expose unresolved additional operands. |
| F-118693b436a8 | Classifies the preliminary value and later selects a receiver operation. The later call’s effective argument is unresolved. |
| F-4a882b291036 | Receives the secondary value on receiver results **6**, **8**, or **9**; its 32-bit result is forwarded to the indirect target. |
| F-6b065a9b9915 | Receives the notification value; only exact result **3** permits the direct write and subsequent embedded-region call. |
| F-5bcae2eff9a7 | Receives the context’s `0x48` or `0x24` embedded-region address under different conditions. |

The indirect targets have no supplied opaque function IDs. Their table offsets identify their selection, not their semantics.

Additional machine-state operands appear in some extraction renderings. Their status as actual arguments is unresolved and does not establish additional caller obligations.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Dependencies are assumed to return normally except where stated otherwise.

1. **Preliminary effect followed by rejection.**  
   The first input classification returns **1**, preliminary classification returns **6**, and the second input classification returns **2**. The `0x48` embedded-region call occurs before the function returns **0**. No receiver-field access is required.

2. **Descriptor mutation before preliminary extraction.**  
   The input is nonzero. The first input-classification call changes S1 and returns **1**. Form A reads S1 after that call returns; it does not use an assumed entry-time descriptor value.

3. **Zero suppresses descriptor reads, not classification.**  
   The input is zero and the first classification returns **1**. Neither S1 nor S2 is read for form A, but F-118693b436a8 still receives zero. The second input-classification call still occurs.

4. **Missing receiver prevents form B reads.**  
   The second classification returns **1**, and the receiver field is zero. The function returns **0** without reading S3 or S4 for form B, even when the original input is nonzero.

5. **Shared slot, different observed values.**  
   With a valid initial receiver and nonzero input, form B reads S4. Receiver selection then changes S4 and returns **8**, with a nonzero intermediate value. Form C reads S4 again after that change. The two extraction calls need not receive equal values from their shared source slot.

6. **Zero intermediate bypasses form C sources.**  
   Receiver selection returns **9**, and the intermediate value is zero. S4 and S5 are not read for form C. F-4a882b291036 nevertheless receives zero, after which dispatch uses table offset `0x14`.

7. **Receiver replacement during conversion.**  
   F-4a882b291036 replaces the receiver field with another valid receiver. Dispatch uses the replacement. If it replaces the field with zero, no visible guard prevents the subsequent dereference.

8. **Result 7 bypasses secondary processing.**  
   Receiver selection returns **7**. The `0x08` target is invoked without visible explicit arguments. There are no form C descriptor reads or conversion call. Normal completion returns **1**.

9. **Notification descriptor timing and write ordering.**  
   The input is nonzero, and the second input classifier changes S3 before returning **3**. Form D reads S3 and S6 afterward. If F-6b065a9b9915 returns **3**, the byte at `0x28` is written before the `0x24` embedded-region call. The byte’s value after that dependency returns remains unconstrained.

10. **Rejected notification.**  
    F-6b065a9b9915 returns a value other than **3**. The function returns **0** without the explicit byte write or the `0x24` call. Any preceding descriptor reads, extraction, and preliminary effects have already occurred.

## Uncertainties

The following remain **documented_only**:

- Descriptor-source contents, declared widths, meanings, and validity requirements.
- The relative order of the two descriptor-source reads within one extraction.
- Synchronization or consistency guarantees for descriptor-source updates.
- The domain meaning of classification constants and extracted values.
- The receiver-selection call’s effective argument.
- The complete calling convention of the `0x08` indirect target.
- Whether additional displayed extraction operands are actual arguments.
- Dependency effects on globals, memory, ownership, allocation, retention, and release.
- Thread safety, reentrancy, structure sizes, byte order, alignment, and invalid-address behavior.

The embedded-region operations may serve a state-management purpose, and the indirect layout resembles receiver-method dispatch. These are inferred interpretations, not established dependency contracts.

## Evidence

Established static evidence consists of the supplied function representation: exact comparisons, separate classification calls, conditional extraction, six global descriptor-source locations and their sharing relationships, call-relative descriptor-read timing, context offsets, dispatch-table offsets, visible operands, the one-byte write, and normal return constants.

The representation establishes source-slot identity, not immutable descriptor values or equality across calls. It does not establish a relative evaluation order for the two descriptor arguments.

No dependency implementations, descriptor contents, external interface specification, or runtime traces were supplied. No runtime tests have been performed.
