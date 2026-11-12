<!-- Trusted provenance: F-fe9998d8ffdf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fe9998d8ffdf

## Purpose

F-fe9998d8ffdf conditionally prepares an object, evaluates an ordered series of exclusion conditions, requests a state change when a condition is met, and returns a packed 64-bit result.

The return value is **not a direct acceptance or rejection result**. Its low 32 bits report whether a final call to F-83c3056a21b8 returns exactly 1. Its high 32 bits contain either the original object argument or the latest 16-bit criterion value captured during evaluation.

The supplied static evidence establishes the conditions, dependency calls, and data accesses described below. Interpreting those conditions as filtering rules, the criterion regions as patterns, or the state-changing calls as acceptance and rejection operations is inferred intent, not an established dependency contract.

## Interface

The function accepts four 32-bit arguments:

| Argument | Role established by use |
|---|---|
| Configuration address | Base address for criterion fields, byte switches, masks, and an integer selector. |
| Object value | Passed to dependencies and, on processing paths, used as a base address for object fields. |
| Context address | Used for a byte flag field. A zero value causes a replacement to be obtained from F-267f68fe54d6. |
| Auxiliary value | Uninterpreted 32-bit value used to initialize a dependency-output slot. |

There is no explicit argument validation, allocation, or error return.

### Return layout

The result is a 64-bit quantity with:

- Bits 0–31: 1 if the final F-83c3056a21b8 result equals 1; otherwise 0.
- Bits 32–63: a 32-bit tracking value, initially equal to the original object argument.

The tracking value changes only when a criterion comparison is reached. Each such update zero-extends a 16-bit field to 32 bits. It does not accumulate multiple criterion values.

Consequently, a result can carry an object address or object token in its upper half on one path, and a small criterion value on another. Consumers must not interpret that half uniformly without additional context.

### Relevant configuration layout

Offsets are byte offsets from the configuration address.

| Offset | Width | Established use |
|---|---:|---|
| 0x02c | 16 bits | First criterion enable/value |
| 0x02e | Unspecified | First criterion data address |
| 0x22c | 16 bits | Second criterion enable/value |
| 0x22e | Unspecified | Second criterion data address |
| 0x42c | 16 bits | Third criterion enable/value |
| 0x42e | Unspecified | Third criterion data address |
| 0x62c | 16 bits | Fourth criterion enable/value |
| 0x62e | Unspecified | Fourth criterion data address |
| 0x82c | 16 bits | Context-bit-0x10 criterion enable/value |
| 0x82e | Unspecified | Corresponding criterion data address |
| 0xa2c | 16 bits | Context-bit-0x20 criterion enable/value |
| 0xa2e | Unspecified | Corresponding criterion data address |
| 0xc2c | 16 bits | Final data criterion enable/value |
| 0xc2e | Unspecified | Corresponding criterion data address |
| 0xe2c | 32 bits | Integer selector; all-one bits disable this check |
| 0xe30 | 32 bits | Required-overlap mask |
| 0xe34 | 32 bits | Forbidden-overlap mask |
| 0xef9 | 8 bits | Enables exclusion when object-detail bit 0x02 is clear |
| 0xefa | 8 bits | Enables exclusion when object-detail bit 0x02 is set |

The 16-bit fields are tested for nonzero and later read as unsigned values. Their signed interpretation does not affect the enabling test. Values with bit 15 set are not rejected merely for appearing negative under a signed interpretation.

The spacing between criterion data addresses does not establish their lengths, encodings, or termination rules.

### Other layouts

On paths that reach the relevant accesses:

- Object +0x04 contains a 32-bit value passed to two extraction dependencies.
- Object +0x1d contains a byte whose bit 0 is tested.
- Object +0x20 contains a 32-bit value passed to four extraction dependencies and also used as an address for a byte at +0x15.
- Context +0x1ac contains a byte whose bits 0x10 and 0x20 select alternate criterion paths.
- An address returned by F-875f8fbbb5eb may expose:
  - A 32-bit flag word at +0x1c.
  - A 32-bit mask at +0xbc.
  - A 32-bit address at +0x10; that address leads to a 32-bit integer at +0x20.

Valid storage is required for every access actually reached. The function does not guard all pointer chains against null or invalid addresses.

## Behavior

### Entry gate and initialization

Two local 32-bit output slots initially contain the original auxiliary argument and the original context argument. A separate tracking value initially contains the object argument.

The first observable dependency call is F-83c3056a21b8 with the object and configuration.

- If its result is nonzero, all preparation and condition evaluation are skipped.
- If its result is zero, processing continues.

This entry test differs from later tests: entry requires **zero** to proceed, whereas individual checks generally require **exactly 1**.

When processing continues and the context argument is zero, F-267f68fe54d6 is called with the configuration. Its result becomes the context address used by subsequent context-dependent operations. The local output slot initialized from the original context remains unchanged by this substitution.

Preparation then calls, in order:

1. F-e03d34f0b7e4 with operation 1, the resolved context, and the configuration.
2. F-455544658bc7 with operation 1, the object, and the configuration.

The function next calls F-875f8fbbb5eb with the object. If that result is nonzero, it calls the dependency again and reads bit 0x4000 from the 32-bit word at offset 0x1c of the **second** result. If the bit is set, it invokes F-455544658bc7 with operation 2 and the object; the supplied call form does not show a third argument.

The first nonzero result does not establish that the second result is nonzero. This path therefore requires the second result to permit the field access.

This early operation-2 call does not terminate evaluation.

### Common criterion comparison behavior

Seven criterion paths share the following observable structure:

- Call F-83c3056a21b8 anew.
- Continue with that criterion only if the result is exactly 1 and its 16-bit enable field is nonzero.
- Invoke the criterion’s extraction dependency, giving it a source value and addresses of the two local output slots.
- After that dependency returns, reread the 16-bit criterion field and assign its zero-extended value to the tracking value.
- Call F-aa53e7865743 with:
  - A leading zero.
  - The current auxiliary output-slot value.
  - The current context output-slot value.
  - The criterion data address.
- A nonzero comparison result transfers control to the common exclusion action.
- A zero comparison result continues to the next applicable stage.

The output slots are reused, not reset between extraction calls. Their initialized values and any prior modifications can therefore matter if a dependency writes only one slot or neither slot.

The tracking-value read occurs after the extraction call. A dependency-induced change to configuration memory can make that value differ from the nonzero value that originally enabled the criterion, including making it zero.

### First four criteria

The first four criterion paths run in this order:

| Order | Enable/value offset | Data offset | Extraction dependency | Source |
|---|---:|---:|---|---|
| 1 | 0x02c | 0x02e | F-05f08d4e44ad | Object field at +0x20 |
| 2 | 0x22c | 0x22e | F-c99cba5deb0d | Object field at +0x20 |
| 3 | 0x42c | 0x42e | F-16b769eefd01 | Object field at +0x20 |
| 4 | 0x62c | 0x62e | F-7445c028e81c | Object field at +0x20 |

Each stage makes its own applicability call. A skipped or zero-result criterion does not imply that later applicability results remain unchanged.

### Object-detail bit exclusions

After the first four criteria:

1. If a fresh applicability result is exactly 1, configuration byte +0xef9 is nonzero, and bit 0x02 of the byte at object-detail +0x15 is clear, take the common exclusion action.
2. Otherwise, make another applicability call. If it returns exactly 1, configuration byte +0xefa is nonzero, and the same detail bit is set, take the common exclusion action.

The object-detail address is loaded from object +0x20. Its flag byte is read only when the preceding applicability and configuration-byte conditions permit it.

With stable inputs and both configuration bytes enabled, either value of the detail bit triggers one of these exclusions. Stability is a condition of that inference, not a property guaranteed for dependencies.

### Context-selected criteria and fallback object flag

Next, a fresh applicability result and the resolved context’s byte at +0x1ac determine the first context branch.

- If applicability is exactly 1 and bit 0x10 is set, select the criterion at +0x82c/+0x82e. Its extraction dependency is F-631937468565, using the object field at +0x04.
- Only when that first branch is not selected is applicability queried again for the bit-0x20 branch.
- If the second applicability result is exactly 1 and bit 0x20 is set, select the criterion at +0xa2c/+0xa2e. Its extraction dependency is F-4e666fff06ef, also using object +0x04.

A selected branch whose 16-bit criterion is zero proceeds to the mask checks without a comparison. It does **not** fall through to the other context branch or the fallback object flag.

When neither context branch is selected, bit 0 of the byte at object +0x1d is tested. A set bit takes the common exclusion action; a clear bit continues.

Thus, under stable applicability and context values, bit 0x10 has priority over bit 0x20, including when its associated criterion is disabled.

### Mask exclusions

Two mask checks follow, each with a fresh applicability call.

**Required overlap:** If applicability is exactly 1 and configuration word +0xe30 is nonzero, obtain an address from F-875f8fbbb5eb. A zero address skips exclusion. Otherwise read its mask at +0xbc. Exclude only when that mask is nonzero and has no bits in common with the configuration mask read for the overlap test.

**Forbidden overlap:** If applicability is exactly 1 and configuration word +0xe34 is nonzero, obtain another address from F-875f8fbbb5eb. A zero address skips exclusion. Otherwise read its mask at +0xbc. Exclude only when that mask is nonzero and overlaps the configuration mask read for this test.

A missing returned object or a zero returned mask is not itself an exclusion in either check.

The configuration masks used in the bitwise operations are read after the dependency call; the earlier enabling reads are not cached substitutes.

### Final criterion and integer selector

The final data criterion uses configuration +0xc2c/+0xc2e and F-c62946793ec4. Unlike the other extraction calls, this dependency receives the object argument itself. Comparison and tracking-value behavior are otherwise as described above.

Finally, applicability is queried again:

- A result other than exactly 1 skips the integer check.
- Configuration +0xe2c equal to 0xffffffff also skips it.
- Otherwise, call F-875f8fbbb5eb.
- A zero result skips exclusion.
- For a nonzero result, read the current configured selector, follow the address at returned object +0x10, and read the 32-bit integer at the resulting address +0x20.
- Unequal 32-bit values take the common exclusion action; equal values do not.

There is no null check on the nested address at +0x10.

### Exclusion action and return

Every common exclusion path calls F-455544658bc7 with operation 2, the object, and the configuration. Remaining conditions are skipped.

Both excluded and non-excluded paths then call F-83c3056a21b8 once more. The final result is packed using the exact-equality Boolean from that call and the current tracking value.

No cached applicability result determines the return.

## Side effects

Established direct effects are:

- Dependency calls in the conditional order described above.
- Reads from argument-referenced and dependency-returned storage.
- Writes to local output slots by dependencies that receive their addresses.
- Updates to the internal tracking value.

No direct store into configuration, object, or context storage is shown. Dependencies may nevertheless mutate those objects or other state. Their operation values must not be assigned undocumented meanings.

The early operation-2 call and a later common exclusion call can both occur in one invocation.

No allocation, deallocation, ownership transfer, or release is explicit. Passing a value or an output-slot address does not establish ownership semantics. The lifetime and ownership of extracted values and dependency-returned addresses remain documented_only.

There is no explicit recovery from an invalid memory access, dependency failure, or invalid returned address.

## Dependencies

| Opaque ID | Role visible at this boundary |
|---|---|
| F-267f68fe54d6 | Supplies a replacement context when the context argument is zero. |
| F-e03d34f0b7e4 | Receives operation 1, context, and configuration during preparation. |
| F-455544658bc7 | Receives operation 1 or 2 for the object; one operation-2 call has only two explicit arguments. |
| F-83c3056a21b8 | Repeated gate query; zero controls entry and exactly 1 enables later checks and the returned Boolean. |
| F-875f8fbbb5eb | Supplies addresses used for preparation flags, masks, and an integer selector comparison. |
| F-aa53e7865743 | Compares extracted values with criterion data; nonzero triggers exclusion. |
| F-05f08d4e44ad | Extracts values for the first criterion. |
| F-c99cba5deb0d | Extracts values for the second criterion. |
| F-16b769eefd01 | Extracts values for the third criterion. |
| F-7445c028e81c | Extracts values for the fourth criterion. |
| F-631937468565 | Extracts values for the bit-0x10 context criterion. |
| F-4e666fff06ef | Extracts values for the bit-0x20 context criterion. |
| F-c62946793ec4 | Extracts values for the final data criterion. |

These roles describe call-site behavior, not complete contracts for the dependencies. No purity, stability, memory preservation, or ownership guarantees are established.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Entry bypass:** If the initial gate result is nonzero, preparation and all criteria are skipped. If the final gate returns 1, the low half is 1; otherwise it is 0. The high half remains the original object value.

2. **Exact equality matters:** If a later criterion’s gate returns 2, that criterion is skipped. A final gate result of 2 also produces a low half of 0.

3. **First criterion excludes:** Suppose entry returns zero, preparation completes, the first criterion’s gate returns 1, its enable field is nonzero, extraction leaves that field equal to 0x8001, and comparison returns nonzero. The common operation-2 call occurs, later criteria are skipped, and the high half is 0x00008001. The low half still depends on the final gate.

4. **Latest comparison wins:** If the first and second comparisons both return zero, and their post-extraction values are 3 and 9, the tracking value becomes 9. If no later comparison is reached, the returned high half remains 9 even if a later mask excludes the object.

5. **Disabled priority branch:** With qualifying gates and stable context bits 0x10 and 0x20 both set, a zero criterion at +0x82c skips comparison and proceeds to mask checks. The +0xa2c criterion and fallback object flag are not evaluated.

6. **Missing mask object:** If a mask check is enabled but its address-producing dependency returns zero, that check does not exclude. Subsequent stages remain eligible.

7. **Zero detail mask:** With a nonzero returned address but a zero mask at +0xbc, neither the required-overlap nor forbidden-overlap rule excludes on that mask.

8. **Mutation during extraction:** If a criterion is enabled with value 5 but its extraction dependency changes that field to zero, comparison still occurs. The tracking value becomes zero because it is read after extraction.

9. **Selector mismatch:** With a qualifying final selector gate, a selector other than 0xffffffff, valid returned and nested addresses, and unequal selector values, the common exclusion action occurs. That mismatch alone does not establish the returned low half.

## Uncertainties

The following remain documented_only:

- Argument-domain names, storage ownership, and complete object layouts.
- The actual meanings of preparation, operation 1, operation 2, and gate results.
- Criterion encoding, size, and the interpretation of the two extraction outputs.
- Dependency error behavior, reentrancy, and preservation of caller-visible state.
- Whether repeated address-producing calls normally return identical addresses.
- The ABI significance of the two-argument operation-2 call.
- Complete return-width declarations for dependencies whose supplied representation exposes wider results. The described dereferences use their low 32-bit address portions; no independent behavioral use of an upper return half is established on those paths.
- Byte order and failure behavior for invalid or misaligned storage.

No additional validation or fallback behavior should be inferred to fill these gaps.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes the branch conditions, field offsets and widths, short-circuit read order, dependency-call order, local output-slot reuse, tracking-value updates, and 64-bit return construction.

Filtering terminology and suggested data roles are inferred intent. Dependency internals, production invariants, and runtime behavior are not established.

No runtime tests have been performed.
