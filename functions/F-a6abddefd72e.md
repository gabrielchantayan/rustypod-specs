<!-- Trusted provenance: F-a6abddefd72e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a6abddefd72e

## Purpose

F-a6abddefd72e transforms an ordered, cyclic list of record references through two successive filtering stages. Each stage can retain existing references and insert references to newly prepared records in context-owned scratch storage.

The first stage applies up to six context-selected predicates. It classifies records through F-b8ecc89bbefe and prepares crossing records through F-a4ea3fe4eb7a and F-ba69a5506d0a.

The second stage applies up to six conditions encoded in record flag bits. At transitions between accepted and rejected records, it selects an endpoint using F-6c5d754a1830, prepares a scratch record through F-b81aafab8a38, and processes that record through F-ba69a5506d0a.

The apparent intent is cyclic polygon clipping or a similar geometric operation. That interpretation is inferred, not required by the established behavior. In particular, the second stage does not visibly interpolate a record: its computed value selects which endpoint is supplied to another dependency.

## Interface

The function has five inputs and a 32-bit unsigned return value:

| Input | Established role |
|---|---|
| Context address | Base address for configuration, predicate descriptors, an indirectly loaded dependency argument, and scratch records. |
| Initial count | Unsigned 32-bit number of references in the initial list. |
| Initial list address | Address of an array of 32-bit record references. |
| Alternate list address | Address of a writable array used as the destination of the first executed pass. Subsequent executed passes alternate the two arrays. |
| Final-list output address | Address of a writable 32-bit slot receiving the final list address on normal completion. |

The return value is the final reference count. An executed pass that produces no references returns zero immediately, without writing the final-list output slot. Consequently, zero does not by itself indicate whether that slot was written.

### Established layouts

All offsets below are byte offsets.

The context contains:

- At offset `0x2F4`: a 32-bit first-stage enable mask. Only bits 0 through 5 select passes.
- Starting at offset `0x294`: six predicate descriptor addresses, separated by `0x10` bytes. The function passes these addresses to F-b8ecc89bbefe; their contents are otherwise opaque here.
- At offset `0xA90`: a 32-bit address. The function follows that address and reads one 32-bit value, which it retains as an argument for F-a4ea3fe4eb7a.
- Starting at offset `0xDA4`: storage used for newly generated records. Each generated record consumes a successive `0x94`-byte slot across both stages.

A referenced record exposes:

- Signed 32-bit fields at offsets `0x10`, `0x14`, and `0x18`, used by the second stage.
- A signed 32-bit field at offset `0x1C`, also used by the second stage.
- A 32-bit field at offset `0x90`. Only its low byte is used for second-stage classification.

The second-stage condition mask of a record is its low byte at offset `0x90`, shifted right by two. Thus conditions 0 through 5 correspond to original flag bits 2 through 7. Bits 0 and 1 of that byte and all higher bytes are ignored by this function’s classification logic.

### Preconditions and storage requirements

The function performs no explicit pointer, count, bounds, or capacity validation.

For defined operation, all accessed addresses must be valid for their observed reads and writes, including dependency accesses. Both list arrays must have enough capacity for every intermediate result, not merely the initial or final count. Scratch storage must accommodate every generated record across all executed passes.

No general safe-aliasing guarantee is established. In particular, overlapping source and destination arrays can let output writes change references that a pass has not yet consumed. Inputs, context storage, scratch storage, and the final-list output slot must not alias in ways that invalidate the required access sequence.

## Behavior

### Entry and shared state

Before inspecting the first-stage enable mask or considering an empty input, the function reads the address at context offset `0xA90`, follows it, and saves the resulting 32-bit value.

It initializes the next scratch-record address to context offset `0xDA4`. The scratch cursor is shared by both stages and is not reset between passes.

List elements are 32-bit record references. Each executed pass treats the source list as cyclic: its initial previous element is the last source element, followed by traversal from index zero upward.

### First stage: context-selected predicate passes

The function initially reads the enable-mask field. If that value is zero, it skips the entire first stage. Otherwise, it visits six possible passes in ascending order, from bit 0 through bit 5.

For each possible pass, it reads the context mask again and executes the pass only if that pass’s bit is set. An initially nonzero mask containing only higher bits therefore enters the six-pass selection loop but selects no pass unless relevant bits subsequently become set.

For an executed pass:

1. It reads the last source reference and classifies it through F-b8ecc89bbefe using the current predicate descriptor.
2. It traverses each source position in ascending order, reading its reference and obtaining another signed 32-bit classification through F-b8ecc89bbefe.
3. It emits zero, one, or two references for that cyclic edge according to the classification rules below.
4. It uses the current record and classification as the previous record and classification for the next edge.

A current record is retained only when its classification is strictly positive.

A crossing record is generated in exactly these cases:

| Previous classification | Current classification | Emission order |
|---|---|---|
| Positive | Zero or negative | Generated record only |
| Negative | Positive | Generated record, then current reference |
| Zero or positive | Positive, without the negative-to-positive case | Current reference only |
| Zero or negative | Zero or negative | Nothing |

Zero is therefore not simply interchangeable with a negative classification. A positive-to-zero edge generates a record, but a zero-to-positive edge does not.

For a crossing, the function compares the absolute magnitudes of the previous and current classifications:

- If the current magnitude is smaller, it calls F-6c5d754a1830 with the previous classification and the difference “previous minus current.” It supplies the previous reference before the current reference to F-a4ea3fe4eb7a.
- Otherwise, including equal magnitudes, it calls F-6c5d754a1830 with the current classification and “current minus previous.” It supplies the current reference before the previous reference.

F-a4ea3fe4eb7a receives the next scratch-record address, those two ordered references, the opaque result from F-6c5d754a1830, and the 32-bit value saved at entry.

The function then calls F-ba69a5506d0a on that scratch record, appends its address to the destination list, and advances the scratch cursor by `0x94` bytes.

When retaining an existing record, it rereads the reference from the source array instead of necessarily writing the previously cached reference. That distinction can matter if a dependency modifies the array.

After completing an executed pass, it swaps the source and destination array roles. If the resulting count is zero, it immediately returns zero without writing the final-list output slot. Otherwise, the next selected pass consumes that result.

Skipped passes do not swap the arrays.

### Second stage: record-condition passes

After the first stage, the function scans the current list and forms the bitwise union of all record condition masks.

It then visits conditions 0 through 5 in ascending order. A condition pass executes only when its bit is set in the currently accumulated union.

For a selected condition, a record is accepted when that condition bit is clear in its record condition mask. A set bit means rejected for that pass.

The pass starts with the last source record’s acceptance state and traverses the list from index zero. For every edge:

- If the acceptance states differ, it emits one generated record.
- If the current record is accepted, it emits the current source reference after any generated record.
- If both records are rejected, it emits nothing.

For a transition, the function identifies the accepted endpoint and the rejected endpoint, independent of traversal direction.

Condition pairs select one signed coordinate field:

| Condition | Coordinate offset | Transformation of offset-`0x1C` field |
|---|---|---|
| 0 | `0x10` | Negate |
| 1 | `0x10` | Unchanged |
| 2 | `0x14` | Negate |
| 3 | `0x14` | Unchanged |
| 4 | `0x18` | Negate |
| 5 | `0x18` | Unchanged |

Let the selected coordinates of the accepted and rejected endpoints be \(a\) and \(r\), and their transformed offset-`0x1C` values be \(s_a\) and \(s_r\). The function passes these two signed quantities to F-6c5d754a1830:

- First argument: \(s_a-a\).
- Second argument: \(((r-a)-s_r)+s_a\), with the subtraction and addition grouping shown here.

It interprets the dependency result as signed for comparison with `0x08000000`:

- A result strictly below `0x08000000`, including a negative result, selects the accepted endpoint.
- A result equal to or above that threshold selects the rejected endpoint.

It calls F-b81aafab8a38 with the next scratch-record address and the selected endpoint, then calls F-ba69a5506d0a on the scratch record.

After those calls, it reads the generated record’s condition mask, incorporates it into the destination union, and appends the scratch address. It advances the scratch cursor by `0x94` bytes.

For a retained source element, it rereads the source-array entry to write the reference, and reads the referenced record’s condition mask to update the destination union.

At the start of every executed condition pass, the accumulated union is reset to zero. It is rebuilt only from emitted generated records and retained references. This rebuilt union determines whether later condition passes execute. Skipped passes preserve the existing union.

An earlier condition is never revisited, even if a generated record subsequently has that earlier condition bit set.

After an executed pass, the arrays exchange roles. A zero result causes the same immediate return-zero behavior as in the first stage: the final-list output slot remains unwritten.

### Normal completion and empty input

After considering all six second-stage conditions, the function writes the current list address to the final-list output slot and returns the current count.

An initially empty list has two materially different cases:

- If no first-stage pass executes, the condition-union scan has no elements, no second-stage pass executes, and normal completion writes the initial list address and returns zero.
- If a first-stage pass executes, the function attempts to read the last source reference despite the zero count. No safe empty-list behavior is established for that case.

## Side effects

Directly observable writes are:

- Generated-record preparation through dependencies at successive scratch addresses.
- Reference writes to the active destination array, in traversal and emission order.
- The final list address written to the final-list output slot only on normal completion.

There is no visible allocation, release, rollback, or clearing of unused array entries or scratch slots. A return of zero does not undo prior writes or dependency effects.

Existing record references can survive into the result unchanged. Generated references point into the context-relative scratch area. No ownership transfer, release obligation, or scratch lifetime beyond the supplied storage’s validity is established.

Dependencies may have additional effects. Their absence cannot be assumed. In particular, later context-mask reads, source-reference rereads, and flag reads can observe dependency modifications. Cached classifications, cached references, and the entry-saved indirect value are not automatically refreshed after a dependency call.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-b8ecc89bbefe | Receives a record reference and predicate-descriptor address; its signed result drives first-stage acceptance and crossing decisions. |
| F-6c5d754a1830 | Receives two 32-bit arithmetic values. Its result is forwarded during first-stage generation and compared against a signed threshold during second-stage endpoint selection. Division, scaling, rounding, and exceptional-input behavior are undocumented here. |
| F-a4ea3fe4eb7a | Receives a scratch destination, two ordered record references, the arithmetic dependency’s result, and the entry-saved 32-bit value. Its written layout and other effects are unknown. |
| F-b81aafab8a38 | Receives a scratch destination and one selected endpoint reference. Copy semantics, write extent, and other effects are not established. |
| F-ba69a5506d0a | Runs after either scratch-record preparation operation. The second stage subsequently reads the destination’s flags. Its full effects are unknown. |

No dependency is assumed to preserve globals, context fields, source arrays, or referenced records. No dependency is assumed to acquire or release memory.

## Acceptance examples

These examples are conditional static expectations, not runtime test results. Unless stated otherwise, they assume valid capacities, ordinary nonoverflowing arithmetic, and dependencies that do not introduce unrelated mutations.

1. **No filtering selected.**  
   With the first-stage mask zero and every input record’s condition mask zero, the result contains the original count and uses the original list address. The alternate array need not receive any writes. The entry-time indirect read still occurs.

2. **A skipped high-bit-only mask.**  
   With a stable first-stage mask containing only bit 6, no first-stage pass executes. Bits above 5 do not select additional predicates.

3. **First-stage zero asymmetry.**  
   For a two-record cyclic list whose classifications are positive and zero, traversal generates one record for the positive-to-zero edge and retains the positive record. The zero-to-positive edge does not generate another record.

4. **First-stage crossing order.**  
   For a cyclic list with classifications positive, positive, negative, starting at the first positive record, the pass emits: a generated record for the closing negative-to-positive edge, the first positive reference, the second positive reference, and a generated record for the positive-to-negative edge.

5. **Equal first-stage magnitudes.**  
   A crossing with classifications of equal absolute magnitude uses the branch that supplies the current reference before the previous reference to F-a4ea3fe4eb7a.

6. **Second-stage endpoint threshold.**  
   For either direction of an accepted/rejected transition, an F-6c5d754a1830 result of `0x07FFFFFF` selects the accepted endpoint. A result of `0x08000000` selects the rejected endpoint. A negative signed result selects the accepted endpoint.

7. **Complete rejection.**  
   If an executed condition pass sees every record as rejected, it generates no transition records and retains nothing. It returns zero without writing the final-list output slot.

8. **Empty normal completion.**  
   With an initial count of zero and no executed first-stage pass, the function writes the initial list address to the final-list output slot and returns zero.

9. **Later-condition selection changes.**  
   If an executed condition pass emits records whose union no longer contains a later condition bit, that later pass is skipped, even if the bit appeared in the initial union.

## Uncertainties

- **documented_only — arithmetic extremes:** Counts, references, offsets, masks, and stored words are 32-bit. First-stage magnitude calculations negate negative signed values; the minimum signed value has no positive signed counterpart. Differences and second-stage expressions can also exceed the signed range. Exact overflow behavior is not established by the supplied representation. Intended geometric behavior must not be extrapolated to those cases.
- **documented_only — address arithmetic:** Element addresses use four-byte scaling, and scratch addresses advance by `0x94`. No overflow or storage-bound checks are visible.
- **documented_only — arithmetic dependency:** Its name-independent contract is unavailable. A fixed-point ratio interpretation is plausible but not established. Zero-denominator behavior is unknown.
- **documented_only — scratch content:** Passing a destination to a dependency does not establish which bytes it initializes. The second stage requires meaningful subsequent flag reads, but this function alone does not prove those bytes are initialized.
- **documented_only — mutation and concurrency:** No synchronization or stable-context guarantee is visible. Arbitrary dependency or concurrent mutation can invalidate simple geometric interpretations.
- **Inferred intent:** The six paired coordinate conditions and cyclic edge processing suggest clipping against geometric boundaries. That inference does not replace the observed endpoint-selection behavior.

## Evidence

The established contract comes solely from static inspection of the supplied function representation: access offsets and widths, branch conditions, loop bounds, dependency argument order, output-write order, and return paths.

No dependency implementations, external type definitions, caller contracts, runtime traces, or capacity guarantees were supplied. No runtime tests have been performed. The acceptance examples describe conditional consequences of the visible control flow, not observed executions.
