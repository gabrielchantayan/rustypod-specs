<!-- Trusted provenance: F-0bfad00ead24; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0bfad00ead24

## Purpose

F-0bfad00ead24 coordinates four dependencies, conditionally checks response integrity, and populates selected fields in a caller-supplied context. It returns a 64-bit result containing a status and a pointer-valued word.

The response layout, fixed value of 512, scaling calculations, and adjacent-byte reversal suggest device-identification processing. That interpretation is **inferred intent**, not an established protocol identity.

This contract describes static behavior visible in the supplied source. Repeated reads remain distinct observations: neither response storage nor global values are generally established to remain stable.

## Interface

### Arguments

| Argument | Width | Role |
|---|---:|---|
| Context | 32-bit pointer | Mutable storage used for validation, dependency calls, and output fields. |
| Response buffer | 32-bit pointer | Storage passed to the final dependency and subsequently parsed. |
| Request word | 32 bits | Initially stored in a temporary descriptor; its low eight bits are later retained. |
| Auxiliary word | 32 bits | Initially stored in the descriptor, then replaced after preparation succeeds. |

All offsets in this contract are byte offsets. Multibyte accesses use the target’s native representation. Byte order, alignment requirements, and invalid-access behavior are not independently established.

The context must support the validation reads at offsets `0x00` and `0x44`, the later read at `0x0C`, and the output accesses described below. Dependencies may require additional storage.

There is no response-length argument. The fixed accesses include a 16-bit read at response offset `0x200`; ordinary contiguous storage therefore needs at least 514 accessible bytes for those accesses. A separate globally selected response offset and dependency behavior may require additional accessible storage.

No explicit response-buffer null, alignment, or extent check occurs.

### Return value

The return is a 64-bit bit pattern:

| Bits | Meaning |
|---|---|
| 31–0 | Status |
| 63–32 | Current first descriptor word, except initial validation failure uses the original response-buffer pointer |

Locally generated statuses are:

- `0`: processing completed successfully.
- `7`: initial context validation failed.
- `0x5B`: the enabled checksum check failed.

A nonzero dependency result is propagated as its 32-bit status bit pattern. Its signed interpretation and error namespace are unknown.

The upper half is not a validated output pointer. F-f60b937b7afa receives the descriptor by address and may modify its first word. Response acquisition and parsing nevertheless use the original response-buffer argument.

### Temporary descriptor

The temporary descriptor consists of three consecutive 32-bit words:

| Offset | Initial contents |
|---|---|
| `0x00` | Original response-buffer pointer |
| `0x04` | Request word |
| `0x08` | Auxiliary word |

It is local temporary storage. Passing its address does not establish whether a dependency retains that address or whether such retention would be safe.

## Behavior

### 1. Validate the context

The checks are short-circuited in this order:

1. Require a nonnull context.
2. Compare the 32-bit context word at `0x00` with a global reference value.
3. Only if that comparison succeeds, require the 32-bit context word at `0x44` to be nonzero.

Failure returns status `7` with the original response-buffer pointer in the upper half. No dependency is called.

These checks do not establish that a nonnull context points to valid storage. No conversion of invalid memory accesses into a status is shown.

### 2. Initialize and prepare the descriptor

After validation, initialize the three descriptor words as specified under Interface, then call F-cf5fba21d449 with the context.

A nonzero result terminates processing immediately, using that result as status and the current first descriptor word as the upper return half.

After a zero result, perform a 32-bit read and a 32-bit write using a global object base plus `0x10`:

- One evaluation of the global base selects the source address.
- A separate evaluation selects the destination address.
- The value read from the source is written to the destination.

The relative order of the two global-base evaluations is not established by the supplied assignment expression. The source read must supply the value used by the destination write, but that does not establish which base evaluation happens first.

Only if the global base remains stable do these accesses read and write the same location. Even then, the operation must not automatically be treated as unobservable; access-sensitive storage and concurrency semantics are undocumented.

Next, revise the descriptor:

- The second word becomes the zero-extended low eight bits of its request value.
- The third word becomes a packed 32-bit value:
  - Bits 31–24: `0xEC`.
  - Bits 23–16: `0xA0` OR the low eight bits of the context word at `0x0C`.
  - Bits 15–0: zero.

The context access supplying this byte is a 32-bit read after F-cf5fba21d449 and after the global-object operation. It is not an entry-time snapshot. The original auxiliary word does not contribute to the replacement value.

### 3. Call the remaining dependencies

Call, in order:

1. F-f60b937b7afa with the context, descriptor address, and zero.
2. F-4a940af17369 with the context.
3. F-79e91facff56 with the context, original response-buffer pointer, and one.

Each call requires every preceding dependency to have returned zero. Any nonzero result immediately terminates processing with that status and the descriptor’s current first word in the upper return half.

The function performs no rollback. Dependency changes to the context, descriptor, response storage, globals, or resource ownership may remain visible on failure.

F-79e91facff56 receives the original response-buffer pointer even if F-f60b937b7afa changed the descriptor’s first word.

### 4. Conditionally check the response checksum

After every dependency succeeds:

1. Read 16 bits at response offset `0x200`.
2. Compare the numeric low eight bits with `0xA5`.
3. If unequal, skip checksum calculation.
4. If equal, read the 512 response bytes at offsets `0x000` through `0x1FF` in ascending address order and accumulate their sum modulo 256.

An enabled checksum passes only when the final eight-bit sum is zero. Otherwise, return status `0x5B` before beginning direct context-field population.

The marker access is 16 bits wide despite only eight numeric bits affecting the decision. The addressed byte corresponding to those numeric bits depends on native byte order. The marker lies outside the summed range.

Checksum success does not establish that later response reads see the bytes previously summed.

### 5. Populate base context fields

The following accesses occur in order:

| Order | Observable access or update |
|---:|---|
| 1 | Write zero to context byte `0x28`. |
| 2 | Read 32 bits from response `0x78`. |
| 3 | If that value is exactly `0x0FFFFFFF`, read 32 bits from response `0xC8` and select that replacement. |
| 4 | Write the selected 32-bit value to context `0x14`. |
| 5 | Write the 16-bit value 512 to context `0x18`. |
| 6 | Read unsigned 16 bits from response `0x02` and retain that value for later arithmetic. |
| 7 | Separately reread 16 bits from response `0x02`, then write this second value to context `0x1A`. |
| 8 | Read unsigned 16 bits from response `0x06` and retain that value for later arithmetic. |
| 9 | Separately reread 16 bits from response `0x06`, then write this second value to context `0x1C`. |
| 10 | Read 16 bits from response `0x0C`, then write that value to context `0x1E`. |

The fallback comparison is exact: `0xFFFFFFFF`, for example, is stored without selecting the fallback.

The values retained for arithmetic need not equal those written to context `0x1A` and `0x1C`. Equality requires stable storage across the respective reads; that stability is not generally established.

### 6. Populate conditional metadata

Read 16 bits from response `0x6A` and inspect bit `0x0004`.

**If the bit is clear:**

1. Write 16-bit zero to context `0x24`.
2. Read 16 bits from response `0x7E` for a zero comparison.
3. If that read is zero, leave context `0x26` and byte `0x29` unchanged.
4. Otherwise, separately reread 16 bits from response `0x7E`, mask this second value with `0x0007`, and write the result as 16 bits to context `0x26`.
5. After that write, write one to context byte `0x29`.

The nonzero decision uses the first read; the stored low three bits use the second. A second read of zero still results in the field write and flag write.

**If the bit is set:**

1. Read 16 bits from response `0xB0`.
2. Write its low five bits as 16 bits to context `0x24`.
3. Write one to context byte `0x29`.
4. Leave context `0x26` unchanged.

Next, read 16 bits from response `0xAE`. If its bits selected by `0xC000` equal `0x4000`, read 16 bits from response `0xAC`, extract bit 10, and write it as zero or one to context byte `0x28`. Otherwise, perform no further write to that byte at this stage. With stable, non-overlapping storage, it remains the zero written earlier.

Finally, process the globally selected response field:

1. Evaluate the global response-field offset.
2. Read 16 bits from the response address selected by that offset.
3. Test bit `0x8000` of this first value.
4. If clear, write 16-bit zero to context `0x2A`.
5. If set, separately reevaluate the global offset, read 16 bits from the newly selected response address, mask this second value with `0x7FFF`, and write the result as 16 bits to context `0x2A`.

The set-bit branch does not reuse either the first offset evaluation or the first field value. The second address may differ, and the second value need not have its high bit set. No second high-bit test occurs.

### 7. Derive scaled values

Use the first reads from response `0x02` and `0x06` as unsigned 16-bit initial quantities, widened to 32 bits.

Start with the first quantity unreduced, the second quantity unscaled, and a scale factor of one. While the current first quantity exceeds 1023 and the current second quantity is below 128:

- Double the scale factor.
- Halve the first quantity by unsigned right shift.
- Double the second quantity.

Afterward, multiply the reduced first quantity by the scale factor and compare with the original first quantity. If unequal, increment the reduced first quantity by one.

Write the resulting first quantity as 16 bits to context `0x20`, followed by the resulting second quantity as 16 bits to context `0x22`.

For the established input widths:

- At most six iterations occur.
- The scale factor is at most 64.
- The reconstruction product fits within 32 bits.
- Each doubled second quantity was below 128 immediately before doubling, so the doubled result is at most 254.
- An initially larger second quantity is not reduced.
- Early termination or final rounding can leave the first output above 1023.

Thus this operation is not an unconditional normalization to a first output of at most 1023.

### 8. Copy the pair-reversed identifier

Copy eight response bytes into context `0x2C` through `0x33`, reversing adjacent pairs.

The source offsets, in destination order, are:

`0x2F`, `0x2E`, `0x31`, `0x30`, `0x33`, `0x32`, `0x35`, `0x34`.

Each source byte read is followed by its corresponding destination byte write before the next source read. This is not an eight-byte snapshot.

Write zero to context byte `0x34`, then return status zero with the current first descriptor word in the upper half.

## Side effects

Direct effects include temporary descriptor writes, the global-base-selected 32-bit write, and selected context writes through byte `0x34`.

There are no direct response-buffer writes through the response pointer. Context or global-object writes can nevertheless alter response storage if the regions alias. Dependencies may also write it.

The function does not establish allocation, deallocation, ownership transfer, or persistent retention of either caller pointer by its own operations. Dependency ownership and retention behavior remain unknown.

The operation is not transactional:

- Validation failure precedes dependency calls.
- Dependency failure may leave dependency effects.
- Checksum failure follows every dependency but precedes direct context-field population.
- Successful population intentionally preserves some fields.
- Faults during population have no shown recovery or rollback.

No non-aliasing constraint is checked. Overlap can make earlier writes affect later response reads, including repeated reads and byte-copy reads. Stable-storage examples below therefore require explicit assumptions.

Concurrency, locking, reentrancy, access-sensitive memory behavior, and memory-ordering guarantees are undocumented.

## Dependencies

| Opaque ID | Observed invocation | Unresolved contract |
|---|---|---|
| F-cf5fba21d449 | First call; context argument | Context/global changes, resource effects, error meanings |
| F-f60b937b7afa | Context, mutable descriptor address, zero | Descriptor changes, pointer retention, ownership effects |
| F-4a940af17369 | Context after descriptor submission succeeds | State changes, resource effects, error meanings |
| F-79e91facff56 | Context, original response pointer, one | Response extent, response guarantees, ownership effects |

No dependency is assumed to preserve globals, context fields, descriptor contents, or ownership merely because its implementation is unavailable.

## Acceptance examples

These are conditional static examples, not executed tests. Unless overridden, assume accessible, stable, non-overlapping storage, stable relevant globals, and dependencies returning zero without additional mutations.

1. **Validation rejection.** A null context produces status `7`, preserves the original response pointer in the upper return half, and invokes no dependency.

2. **Preparation failure.** If F-cf5fba21d449 returns `0x23`, the status is `0x23`. No later dependency, global-object operation, checksum access, or direct context-field population occurs.

3. **Packed descriptor.** A request word of `0x12345678` and a post-preparation context word at `0x0C` ending in `0x05` produce descriptor words `0x00000078` and `0xECA50000` at offsets `0x04` and `0x08`.

4. **Distinct global bases.** If the source-base evaluation selects one object and the destination-base evaluation selects another, the operation transfers the 32-bit value between their respective `0x10` locations. It is a same-location write only when both evaluations select the same address.

5. **Descriptor pointer mutation.** If F-f60b937b7afa replaces the first descriptor word, acquisition and parsing still use the original response pointer. The return’s upper half uses the resulting descriptor word.

6. **Checksum rejection or bypass.** A marker whose numeric low byte is `0xA5`, combined with a summed result of one modulo 256, produces `0x5B` before direct context population. A different marker bypasses that checksum regardless of the sum.

7. **Exact fallback.** A response value of `0x0FFFFFFF` at `0x78` selects the value at `0xC8`; `0xFFFFFFFF` does not.

8. **Arithmetic and stored fields can differ.** Suppose the two reads at `0x02` yield 2047 and 100, respectively, while both reads at `0x06` yield 64. Context `0x1A` receives 100, but scaling uses 2047 and produces outputs 1024 and 128.

9. **Metadata reread.** With bit `0x0004` clear at `0x6A`, suppose the test read at `0x7E` yields one and the subsequent read yields zero. Context `0x26` becomes zero and byte `0x29` becomes one. If the test read itself is zero, neither location is written on that branch.

10. **Global-offset reevaluation.** Suppose the first selected field has bit `0x8000` set, but the second offset evaluation selects another field containing `0x0123`. Context `0x2A` receives `0x0123`; the second field need not have its high bit set.

11. **Scaling boundaries.** Initial arithmetic quantities 2047 and 64 produce 1024 and 128. Quantities 65535 and zero produce 1024 and zero after six iterations and rounding.

12. **Pair reversal.** Stable response bytes `A B C D E F G H` at `0x2E` through `0x35` produce `B A D C F E H G` at context `0x2C`, followed by a zero byte.

## Uncertainties

The following remain **documented_only**:

- Native byte order, alignment requirements, and invalid-access behavior.
- Validation reference value and global-object identity.
- Global response-field offset value, representation, and permitted range.
- Complete context and response layouts and required dependency buffer extent.
- Stability and access semantics of response storage and globals.
- Relative evaluation order of the source and destination global bases in the global-object assignment.
- Dependency internals, errors, resource ownership, cleanup, and pointer retention.
- Required aliasing restrictions and caller-enforced lifetime guarantees.
- Concurrency, reentrancy, and synchronization requirements.
- External protocol identity and interpretation of the derived fields.
- Whether consumers treat the return’s upper half as a pointer, token, or incidental result.

These unknowns do not authorize assuming stable repeated reads, same-location global writes, dependency preservation, safe pointer retention, or automatic cleanup.

## Evidence

**Established static evidence:** the supplied control flow, access widths and offsets, separate repeated reads, separate global-base evaluations, dependency arguments and order, masks, arithmetic bounds, conditional writes, byte-copy order, and 64-bit return construction.

**Inferred intent:** device-identification processing, geometry-like quantities, and identifier-like byte data. These interpretations do not establish an external protocol contract.

**Unknown:** unavailable dependency behavior, global values and stability guarantees, platform memory semantics, and caller obligations not expressed by the function.

No runtime tests have been performed.
