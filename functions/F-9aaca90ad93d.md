<!-- Trusted provenance: F-9aaca90ad93d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9aaca90ad93d

## Purpose

F-9aaca90ad93d selects one of two resource families, programs resource locations from two address-like inputs and a count-derived extent, and prepares additional chunks by repeatedly overwriting one shared four-word descriptor.

Established behavior consists of selection, address calculations, memory accesses, control construction, descriptor mutation, and dependency calls. Chunked transfer is inferred intent. The evidence does not establish a device type, transfer direction, submission protocol, or completion guarantee.

The routine explicitly returns either selection rejection or zero. Zero means execution reached the final return; it does not establish external success or completion.

## Interface

### Inputs and arithmetic

| Input | Width and interpretation | Visible use |
|---|---|---|
| First address-like input | Unsigned 32-bit | Written with bit 31 cleared; its working value advances by each chosen chunk size. |
| Second address-like input | Unsigned 32-bit | Written with bit 31 cleared; not advanced by this routine. |
| Count-like input | Signed 32-bit | Multiplied by four, then assigned to an unsigned 32-bit working extent. |

“Address-like” describes visible use, not validity, ownership, or source/destination direction.

The count multiplication is represented as signed arithmetic before conversion to unsigned. Its product is representable for counts from −536870912 through 536870911, inclusive. Representable negative products convert to their unsigned 32-bit values. Behavior following signed multiplication overflow is **documented_only**; wrapping, rejection, and checked arithmetic must not be assumed.

Working first-address additions and remaining-extent subtractions are unsigned 32-bit operations, modulo \(2^{32}\). Each stored address-like input is masked with `0x7fffffff`. This clears bit 31 without validating or saturating the value.

### Return value

| Value | Explicit condition |
|---|---|
| `0x00000013` | The family-output checks reject selection. |
| `0x00000000` | A supported branch is selected and execution reaches the final return. |

Dependency return values are ignored. No additional explicit status covers invalid storage, arithmetic outside established semantics, dependency failure, or nontermination.

### Shared storage and layouts

Shared state supplies:

- A descriptor address and chunk limit.
- Three resource-base globals for each family.
- A metadata base.
- A control-preservation mask.

The descriptor has four accessed 32-bit words:

| Byte offset | Visible contents |
|---|---|
| `+0` | Initially a snapshot of the second-input resource location; subsequently the masked second input. |
| `+4` | Initially a snapshot of the first-input resource location; subsequently the masked advancing first input. |
| `+8` | Descriptor’s own address or zero. |
| `+12` | Initially a control-location snapshot; subsequently an additional-chunk control value. |

Resource addressing uses a 32-byte index stride. The routine does not establish a coherent snapshot of resource bases or selection outputs:

- The two address-like locations use separate observations of the same selected address-pair base.
- The retained control location and configuration location use separate observations of the selected control base.
- Configuration uses a fresh conversion of the index output rather than necessarily reusing the retained converted index.

Matching address-pair base observations guarantee that the two address-like locations are four bytes apart.

Matching control-base observations and matching converted indices guarantee that configuration is four bytes beyond the retained control location. They are **sufficient, not necessary**, conditions. If \(B_c\) is the retained control base, \(B_f\) the configuration base, \(i\) the retained index, and \(j\) the fresh configuration index, the mathematical address difference is:

\[
(B_f-B_c)+4+32(j-i).
\]

This expression describes the relationship where the represented address calculations are valid. Compensating base and index differences can also yield a difference of four bytes.

The evidence establishes accessed widths and calculated locations, not complete allocation sizes, alignment, ownership, nonoverlap, or memory attributes.

## Behavior

### 1. Capture state and request selection

Before calling the selector, the routine reads and retains the chunk limit, then the descriptor address. It computes the count-derived extent and initializes two 32-bit output slots:

- Index output: `0xffffffff`.
- Family output: zero.

F-cbaa20924752 receives six fixed scalar arguments—`0x1c`, 2, 3, `0x1c`, 2, 3—followed by the family-output address and index-output address. Its return value is ignored.

The routine first checks whether the family output’s low byte is 1. If so, it selects family 1. Otherwise, a second low-byte check must observe 2 to select family 2; failure returns `0x13`.

With stable output storage, this accepts precisely low-byte values 1 and 2. Upper bits do not participate. If the output changes between the checks, selection follows the values actually observed.

Rejection occurs before this routine writes selected resources or the descriptor. It does not exclude effects performed by F-cbaa20924752.

Within a supported branch, the index output is converted through a plain character type and promoted for address calculations. This is the **retained index**. The complete 32-bit output is not used directly. Plain-character signedness and out-of-range conversion behavior are unresolved.

There is no explicit index-range check, including when the selector leaves the initialized `0xffffffff` unchanged.

The output-slot addresses escape to F-cbaa20924752. The evidence does not establish whether that dependency retains them or whether their contents remain unchanged afterward.

### 2. Resolve resource locations

Each supported family performs the following base observations in the represented order:

| Order | Observation | Derived location or retained value |
|---|---|---|
| 1 | First address-pair base observation | Second-input location: base plus 32 times retained index. |
| 2 | Second address-pair base observation | First-input location: base plus 4 plus 32 times retained index. |
| 3 | First control-base observation | Retained control location: base plus 32 times retained index. |
| 4 | Descriptor-pointer base observation | Descriptor-pointer location: base plus 32 times retained index. |
| 5 | Second control-base observation | Base used for configuration. |

After the fifth observation, configuration addressing converts the current index output through the plain character type again. The configuration location is the fifth observed base plus 4 plus 32 times this freshly converted index.

The address relationships therefore depend on the observations, not merely on the nominal table layout. A changed address-pair base can alter the spacing or order of the address-like locations. Changed control bases and indices can separate configuration from control, or compensate to preserve their four-byte spacing.

All family-base observations occur after the selector. Changes made by F-cbaa20924752 can affect them. Changes to the descriptor-address or chunk-limit globals during that call do not replace the values already captured.

Once resolved, all five resource locations are retained. Later output-slot or base-global changes do not cause their recalculation.

### 3. Program the initial chunk

The routine first writes zero to the configuration location. It then reads and saves the retained control location. Bits 21–23 of this saved control determine the initial count shift, from zero through seven.

The initial chunk size is the unsigned minimum of the original extent and captured limit.

The subsequent externally relevant accesses occur in this represented order:

1. Write the masked second input to its resolved resource location.
2. Write the masked first input to its resolved resource location.
3. Read a 32-bit metadata word using the metadata-base global and the current family output’s low byte. Its byte offset is `0xdc` plus four times that byte.
4. Write the descriptor’s own address to descriptor offset `+8`.
5. Read the retained control location and copy that value to descriptor offset `+12`.
6. Read the first-input resource location and copy that value to descriptor offset `+4`.
7. Read the second-input resource location and copy that value to descriptor offset `+0`.
8. Write the descriptor address to the descriptor-pointer location.
9. Read and retain the preservation-mask global for additional chunks.
10. Construct and write the initial control, using another observation of the preservation-mask global.
11. Write the constructed configuration.

The initial descriptor-write order is therefore `+8`, `+12`, `+4`, then `+0`. Its initial control snapshot is not the newly constructed initial control. Likewise, its address-like fields are read back from resource locations rather than copied directly from the inputs.

The initial control is the bitwise union of:

- A fresh retained-control read, ANDed with the independently reread preservation mask.
- The initial chunk size shifted right by the previously saved shift, then masked with `0x0fff`.
- `0x08000000`.

The control read and mask read inside this expression have no established relative evaluation order. The mask saved for later chunks and the mask used here are separate observations and need not agree.

The count contribution occupies at most 12 bits. The preservation mask is not established to clear those bit positions, so preserved control bits can overlap the count contribution.

Configuration is the bitwise union of:

- The metadata word’s low nibble shifted left by six.
- That same nibble shifted left by one.
- `0x8001`.

Other metadata bits do not contribute. The metadata family observation is fresh; it need not match the family previously used for resource resolution.

After programming, the routine subtracts the initial chunk size from the remaining extent and advances the first working address by that size. The second input remains unchanged.

### 4. Prepare additional chunks

Additional-chunk processing is selected exactly when the captured limit is less than the original unsigned extent.

For each iteration with nonzero remaining extent, the routine:

1. Writes the masked current first working address to descriptor offset `+4`.
2. Writes the masked second input to descriptor offset `+0`.
3. Chooses the unsigned minimum of remaining extent and captured limit.
4. Constructs and writes descriptor offset `+12`.
5. Writes descriptor offset `+8`.
6. Advances the first working address by the chosen size.
7. Calls F-dbb1e1e8544e with freshly evaluated selection-output arguments.
8. Subtracts the chosen size from remaining extent after that call returns.

Each additional control is the bitwise union of:

- A current retained-control read ANDed with the saved preservation mask.
- The chunk size shifted right according to bits 21–23 of another current retained-control read, then masked with `0x0fff`.
- `0x08000000`.

The two control reads are distinct in the static representation. Their relative evaluation order is not established, and they need not return equal values. Additional controls use the saved mask, not necessarily the mask used for initial control programming.

If the chosen chunk consumes all remaining extent, descriptor offset `+8` becomes zero. Otherwise, it becomes the descriptor’s own address.

Every iteration overwrites the same descriptor. No separate descriptor allocation or retained list of per-chunk descriptors is visible. Safe asynchronous consumption would require dependency or environmental behavior not established here.

### 5. Finish and return

If the original extent fits within the captured limit, the routine skips additional-chunk processing and clears the descriptor-pointer location. This occurs after descriptor initialization and after installing the descriptor address there.

After either terminating path, the routine calls F-dbb1e1e8544e, then F-4cf587a7e13d, and returns zero.

Every invocation of either dependency reevaluates both selection outputs:

- Family argument: current family output masked to its low byte.
- Index argument: current index output converted through the plain character type and promoted.

Argument evaluation order within each call is not established. No new supported-family check precedes these calls.

Consequently, changed outputs can change dependency arguments without changing retained resource locations. Conditional on termination and normal dependency returns, F-dbb1e1e8544e is called once per additional chunk plus once afterward. F-4cf587a7e13d is called once.

## Side effects

The routine writes all four descriptor words and all five resolved resource locations. These locations are not established to be distinct. Aliasing can change later reads or overwrite earlier results; independent-field interpretations require suitable nonoverlapping storage.

Visible effects include:

- Clearing configuration before writing its constructed value.
- Installing the descriptor address even when no additional chunks are needed.
- Clearing the descriptor-pointer location only on the no-additional-chunks path.
- Leaving descriptor contents in place unless a dependency changes them.

No allocation, deallocation, ownership transfer, rollback, or restoration is visible. Ignored dependency returns do not trigger recovery.

Validity, bounds, alignment, index suitability, and chunk-limit suitability are unchecked. The described access sequence is the order represented by the supplied static evidence, not a guarantee of hardware transactions, atomicity, or memory barriers.

Dependencies may affect globals, resources, descriptors, or escaped output slots. Their preservation, retention, and release behavior must not be assumed.

## Dependencies

| Opaque function ID | Established invocation contract |
|---|---|
| F-cbaa20924752 | Called once with six fixed scalars and two writable 32-bit output-slot addresses; return ignored. |
| F-dbb1e1e8544e | Called after every additional descriptor preparation and once after terminating chunk processing; arguments freshly evaluated each time; return ignored. |
| F-4cf587a7e13d | Called after the final F-dbb1e1e8544e call, with another fresh evaluation of selection outputs; return ignored. |

Submission, blocking, synchronization, completion, pointer retention, ownership, release, and external error reporting are **documented_only**.

## Acceptance examples

These are conditional static deductions, not runtime observations. Unless an example states otherwise, it assumes valid nonoverlapping storage, defined arithmetic, stable shared observations, and normally returning dependencies that do not interfere.

### Selection

A stable family output whose low byte is 3 produces `0x13`, with no selected-resource or descriptor writes by this routine and no later dependency calls.

A stable family output of `0x101` selects family 1. Selector side effects remain possible in both cases.

### Address relationships

With retained index 2 and matching address-pair base observations, the second-input location is base plus 64 and the first-input location is base plus 68.

If only the second base observation increases by `0x100`, the first-input location is `0x104` bytes beyond the second-input location.

With matching control bases and both converted indices equal to 2, configuration is four bytes beyond retained control.

Equality of those observations is not required. A retained control base of `0x1000` with retained index 2 produces control location `0x1040`. A configuration base of `0x0fe0` with fresh index 3 produces configuration location `0x1044`. Both the base and index differ, but their contributions compensate.

If instead the control bases match and only the fresh index increases from 2 to 3, configuration is 36 bytes beyond retained control. None of these relationships causes later resource-location recalculation.

### Chunk processing

For count 4 and limit 64, the extent and initial chunk are 16. The descriptor is initialized and installed, then the descriptor-pointer location is cleared. There are no additional iterations. Each final dependency is called once.

For count 10 and limit 16, the extent is 40. The initial chunk is 16; additional chunks are 16 and 8. Their first-address fields derive from the original first input plus 16 and plus 32, respectively, with unsigned wrapping before masking. The first additional descriptor self-links; the last has a zero link. F-dbb1e1e8544e is called three times, followed by F-4cf587a7e13d once.

### Separate observations

If the saved mask is zero but the initial-expression mask includes `0x1000`, a corresponding set bit in the initial-expression control read survives into the initial control. Additional controls cannot preserve that bit through the saved-mask contribution, and their 12-bit count contribution cannot set it.

If resource resolution uses family 1 and index 2, but the output slots become family 2 and index 3 before a later call evaluates them, that call receives 2 and 3. Retained resource locations do not change.

A change made by the final F-dbb1e1e8544e call can affect arguments subsequently evaluated for F-4cf587a7e13d.

### Encoding and zero boundaries

A chunk of 20 with shift 2 contributes 5 to control. A chunk of `0x1000` with shift zero contributes zero after the 12-bit mask, without explicit rejection.

A zero count still performs selection, initial programming, descriptor initialization, pointer clearing, and both final calls.

With zero limit and positive unsigned extent, every chunk is zero. Under normally returning, noninterfering dependencies, remaining extent never decreases. The loop does not terminate; the final calls and zero return are not reached.

## Uncertainties

- **Established:** Access widths, descriptor offsets, represented observation order, separate configuration-index conversion, retained resource locations, unsigned chunk comparisons, separate mask observations, fresh dependency arguments, and explicit returns.
- **Established:** Matching control bases and indices guarantee four-byte configuration spacing; compensating differences can produce the same spacing.
- **Inferred intent:** Chunked transfer-like resource programming. Device identity and direction remain unestablished.
- **documented_only:** Plain-character signedness and out-of-range narrowing behavior.
- **documented_only:** Behavior following signed multiplication overflow or invalid address calculations.
- **documented_only:** Valid indices, allocation sizes, alignment, aliasing requirements, and acceptable limits.
- **documented_only:** Complete outcomes under changing globals, output slots, or addressed storage. The contract identifies distinct observations but cannot supply missing external behavior.
- **documented_only:** Memory attributes, concurrency rules, atomicity, barriers, dependency ownership, and completion semantics.
- **documented_only:** External meanings of `0x08000000`, `0x8001`, self-links, and zero-length programming.

## Evidence

This contract derives from the supplied static representation of F-9aaca90ad93d and the supplied opaque dependency mapping.

The evidence establishes separate family-base observations, two index conversions during resource resolution, retained calculated locations, descriptor accesses, separate preservation-mask observations, repeated output-slot evaluation, arithmetic, branches, and explicit returns.

The configuration-spacing relationship follows from the two address expressions. Equal base and index observations are sufficient for four-byte spacing, while offsetting differences provide an independently valid case.

No dependency implementations, memory map, hardware specification, runtime trace, or concurrency model were supplied. No runtime tests have been performed. Acceptance examples describe conditional consequences, not observed execution.
