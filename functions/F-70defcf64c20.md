<!-- Trusted provenance: F-70defcf64c20; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-70defcf64c20

## Purpose

F-70defcf64c20 conditionally reallocates storage or performs an in-place sequence of element transfers, then passes a supplied 32-bit value to an assignment-like dependency.

**Inferred intent:** the routine inserts one value at a supplied position in a packed-bit sequence. The object appears to contain a two-word beginning position, a two-word ending position, and a storage-limit address. The 32-bit rounding of allocation size and the use of position/proxy dependencies support this interpretation.

**Established behavior:** the routine chooses its branch by comparing two object words, not by consulting a separately established capacity or size contract. It invokes different dependency sequences on the two branches. The apparent insertion, copying, and iterator meanings remain conditional on the dependencies’ contracts.

This specification distinguishes the directly visible control flow and accesses from those inferred meanings. In particular, it does not assume that dependencies preserve object state, perform successful allocation, copy a particular range, or release storage.

## Interface

The routine has four logical arguments:

| Argument | Width and form | Visible use |
|---|---|---|
| Object | Pointer to storage containing at least five consecutive 32-bit words | Supplies state to dependencies and receives direct state updates |
| Position component A | 32 bits, presented as a signed integer | Combined with component B into a two-word temporary position |
| Position component B | Opaque 32 bits | Combined with component A; forwarded to a transfer-like dependency |
| Value | Opaque 32 bits | Passed to F-bcb6380f47f7 |

There is no explicit return value or explicit success/error result.

The object’s visible layout is:

| Byte offset | Width | Established use | Inferred role |
|---|---:|---|---|
| 0 | 32 bits | Read as a dependency argument and release-like argument; replaced after the growth path | Beginning storage address |
| 4 | 32 bits | Read with offset 0; set to zero after growth | Beginning bit-position component |
| 8 | 32 bits | Compared with offset 16; used with offset 12; writable through dependency calls | Ending storage address component |
| 12 | 32 bits | Paired with offset 8 | Ending bit-position component |
| 16 | 32 bits | Compared with offset 8; used in size arithmetic; replaced after growth | Storage-limit address |

These names describe the inferred representation, not independently verified field contracts. In particular, the branch comparison ignores the word at offset 12.

The supplied position is carried as an ordered pair: component A followed by component B. Several dependencies receive addresses of two-word temporary records or addresses into the object. Those records must be treated as opaque unless the relevant dependency contract establishes their representation.

### Preconditions

For defined, useful execution:

- The object must support all five 32-bit accesses and any additional accesses made by dependencies.
- Addresses interpreted by dependencies must satisfy those dependencies’ validity and alignment requirements.
- The supplied position must satisfy the comparison, traversal, transfer, and assignment dependencies’ requirements.
- Arithmetic used for growth and release must remain within the implementation’s supported domain.
- Dependencies producing temporary records or reference-like results must initialize everything subsequently consumed.

The routine contains no explicit checks establishing these conditions. It also contains no explicit null-object check, position-range check, allocation-failure check, or arithmetic-overflow check.

## Behavior

### Branch selection

The routine reads the object words at offsets 8 and 16 and compares them for equality.

- Equality selects the **growth path**.
- Inequality selects the **in-place path**.

No visible capacity query precedes this decision. Neither offset 4 nor offset 12 participates in the comparison.

### Growth path: size selection and allocation

The routine calls F-43fb2e0768db with the object.

- If the result is zero, the nominal growth quantity is 32.
- Otherwise, it calls F-43fb2e0768db a second time and doubles the **second** result.

The two calls are observably distinct. The second result is not required by this routine to equal the first result. If the first call returns nonzero and the second returns zero, the nominal growth quantity becomes zero.

The routine then converts the nominal quantity to a count of 32-bit units by adding 31 as unsigned 32-bit arithmetic and shifting right by five bits. It requests four times that unit count from F-871e64ddc6de, with a second argument of zero.

Important arithmetic properties:

- The addition of 31 is unsigned 32-bit arithmetic and can wrap.
- The right shift after that addition is a logical unsigned shift.
- Doubling the second query result is expressed as a left shift of a signed 32-bit value. The source does not establish a safe range for that operation. Negative values or values whose doubling is not representable are not given portable, defined behavior by this specification.
- For a valid, nonnegative nominal quantity without addition overflow, the requested size is four times its rounded-up count of 32-bit units.
- The visible unit-count calculation yields at most 134,217,727 units, so its subsequent multiplication by four fits within unsigned 32 bits. That does not establish validity of the resulting address arithmetic.

The allocation-like result is retained separately for the final object updates. The routine does not test it for failure.

### Growth path: dependency sequence

After obtaining the allocation-like result, the routine initializes temporary state containing:

- The supplied two-word position.
- The allocation-like result with a zero companion word.
- Another pair consisting of the allocation-like result and zero.

Not all of these initialized values appear as explicit arguments to the subsequent calls. They do not, by themselves, establish an additional dependency parameter or a proven destination-range contract.

The visible sequence is:

1. Call F-2f14eb415922 with:
   - An output temporary record.
   - The object.
   - The object word at offset 0.
   - The object word at offset 4.

2. Capture both words of that output record.

3. Call F-af9bc0871b42 with the address of the same output record.

4. Form another two-word temporary from the pair captured **before** that call.

5. Call F-4808cb187ddd with an output temporary and the address of the captured pair.

6. Call F-bcb6380f47f7 with that output temporary and the supplied value.

Thus, the value-assignment-like call uses a reference-like result derived from the pre-update pair, not directly from the possibly updated record passed to F-af9bc0871b42.

Next, the routine stages temporary pairs containing the current object words at offsets 8 and 12 and the current words of the first output record. It then calls F-2f14eb415922 again, this time with:

- The address of the object’s word at offset 8 as the output location.
- The object.
- The supplied position component A.
- The supplied position component B.

This second call can modify the object’s ending-position words through its explicit output pointer. Its exact output, transferred range, and relationship to the newly requested allocation are not established by this routine alone.

### Growth path: release-like call and final writes

After the second F-2f14eb415922 call, the routine calls F-ff1bdc376dcd with:

1. The object word currently at offset 0.
2. The current difference between the words at offsets 16 and 0, shifted right by two bits.
3. Zero.

These object values are read **after** the preceding dependency calls. They are not an immutable snapshot of the entry state.

The difference is formed from signed 32-bit integer representations, and the source expresses its division-like operation as a signed right shift. The representation is consistent with a byte-distance converted to four-byte units, but that intended meaning requires valid addresses, supported subtraction, and appropriate signed-shift behavior.

Only after F-ff1bdc376dcd returns does the routine directly perform these stores, in this order:

1. Offset 16 receives the allocation-like result plus four times the unit count.
2. Offset 0 receives the allocation-like result.
3. Offset 4 receives zero.

The routine does not subsequently overwrite offsets 8 or 12 directly. Their resulting values depend on earlier dependency effects.

If F-ff1bdc376dcd releases storage, that release precedes the direct replacement of the object’s beginning and storage-limit fields. The routine itself supplies no rollback or recovery behavior.

### In-place path

The routine first constructs a two-word temporary from the supplied position.

It calls F-7c7f55b590bd with:

- An output temporary record.
- The address of the object word at offset 8.
- The constant one.

Only after this call does it read the object words at offsets 8 and 12 into another two-word temporary. Consequently, those reads observe any changes F-7c7f55b590bd made to the object.

The routine then repeatedly calls F-d5d7c92d36b2 with the supplied-position temporary and the first output record. A zero result ends the loop. Any nonzero result enters one transfer iteration.

Each iteration has this observable dependency order:

1. F-0090defb153a receives the temporary initialized from the object’s ending-position pair.
2. F-4808cb187ddd receives a destination temporary and the result of that call.
3. F-0090defb153a receives the comparison-side temporary originally produced by F-7c7f55b590bd.
4. F-4808cb187ddd receives a second destination temporary and that result.
5. F-d9f8aea71acf receives the second destination temporary.
6. F-bcb6380f47f7 receives the first destination temporary and the result of F-d9f8aea71acf.

The next comparison occurs only after all six calls return.

**Inferred intent:** the two position temporaries move backward, and each iteration copies one logical value from the preceding position into the following position, making room for insertion. The visible routine does not itself decrement either record. Progress and direction depend on the callees.

Once the comparison returns zero:

1. F-4808cb187ddd derives a reference-like temporary from the supplied-position pair.
2. F-bcb6380f47f7 receives that temporary and the supplied value.
3. F-af9bc0871b42 receives the address of the object word at offset 8.

There is no explicit iteration limit. If dependency behavior never makes the comparison return zero, the routine does not reach the final assignment or final position update.

## Side effects

Established direct effects are limited to temporary-state construction and the three growth-path object stores described above.

Dependency-mediated effects can include:

- Mutation of the object, including its ending-position pair.
- Mutation of temporary position or reference records.
- Reads and writes to underlying sequence storage.
- Allocation-like and release-like activity.
- Changes to globals or other externally reachable state.
- Failure, exceptional control transfer, or nontermination.

The visible routine does not establish preservation of globals across any dependency call.

The value argument is forwarded as a full 32-bit quantity. Any conversion to a bit, Boolean normalization, masking, or truncation is a dependency behavior, not a visible operation here.

No ownership transfer is proven solely by the allocation-like and release-like names used in this specification. The routine retains the allocation-like result for object installation and passes the object’s then-current storage description to another dependency. Whether either dependency acquires, retains, or releases ownership is documented_only.

No explicit error handling, cleanup-on-failure, or transactional state restoration is present. Partial dependency effects may precede an interruption.

## Dependencies

| Opaque ID | Established interaction | Inferred role or limitation |
|---|---|---|
| F-43fb2e0768db | Receives the object; returns a 32-bit quantity used for growth | Size-like query; queried twice when the first result is nonzero |
| F-871e64ddc6de | Receives requested byte count and zero; returns an address-like 32-bit value | Allocation-like; failure and ownership contract unknown |
| F-2f14eb415922 | Called twice on growth, with an output record, object, and two 32-bit position components | Transfer-like; exact range and destination semantics unknown |
| F-af9bc0871b42 | Receives a mutable position-record address | Advancement-like; exact mutation unknown |
| F-4808cb187ddd | Receives an output temporary and a position/reference-like input | Reference construction-like |
| F-bcb6380f47f7 | Receives a reference-like temporary and a 32-bit value | Assignment-like; conversion and storage semantics unknown |
| F-ff1bdc376dcd | Receives current storage base, shifted distance, and zero | Release-like; actual ownership effect unknown |
| F-7c7f55b590bd | Receives an output record, object ending-position address, and one | Position-adjustment-like |
| F-d5d7c92d36b2 | Compares two temporary records; zero terminates iteration | Equality or inequality interpretation not independently established |
| F-0090defb153a | Receives a mutable temporary record and returns a reference-like result | Backward traversal-like |
| F-d9f8aea71acf | Receives a reference-like temporary and returns a 32-bit value | Value extraction-like |

## Acceptance examples

These are conditional static acceptance criteria, not observed runtime results.

### Initial growth quantity

If the entry comparison selects growth and the first F-43fb2e0768db result is zero:

- There is no second query.
- The nominal quantity is 32.
- F-871e64ddc6de receives a byte count of four and a second argument of zero.

### Distinct query results

If the first query returns 10 and the second returns 17:

- The second query controls doubling.
- The nominal quantity is 34.
- The unit count is two.
- The requested byte count is eight.

An implementation that reuses the first result and requests four bytes would not preserve this visible behavior.

### Successful growth-path completion

Assume the allocation-like result is a valid address, arithmetic is supported, and every dependency returns normally. The final direct writes install the computed limit first, the retained allocation-like result second, and zero at offset 4 last. No direct end-position update follows them.

This criterion does not establish which elements were copied or whether the old allocation was released.

### In-place insertion with no transfers

If the first F-d5d7c92d36b2 result is zero, none of the six-call transfer iterations occurs. The routine still derives a reference from the supplied position, forwards the supplied value to F-bcb6380f47f7, and finally calls F-af9bc0871b42 on the object’s ending-position record.

### One transfer iteration

If the comparison returns nonzero once and then zero, exactly one extraction-and-assignment sequence occurs before the supplied-value assignment. The object-ending update call occurs after both assignments.

### Dependency mutation

If an earlier growth-path dependency changes the object word at offset 0, the release-like call uses that changed word. A version that always passes the entry-time word would differ from the visible access order.

### Allocation failure

If F-871e64ddc6de returns a failure value, no dedicated failure branch is taken. Subsequent calls still occur unless control flow is interrupted elsewhere. Successful insertion, safe cleanup, and object validity are not acceptance guarantees for that case.

## Uncertainties

The following remain **documented_only**:

- Exact logical container type and element representation.
- Whether position components are an address and bit index.
- Exact transfer ranges and destinations for F-2f14eb415922.
- The significance of staged temporary records not passed as explicit arguments.
- Allocation failure representation and ownership rules.
- Release semantics and required size units.
- Iterator direction, comparison semantics, and termination guarantees.
- Value normalization or truncation.
- Aliasing restrictions and behavior for invalid positions.
- Global-state preservation by any dependency.
- Behavior outside the supported signed-arithmetic and address ranges.

The packed-bit insertion interpretation is useful explanatory intent, not a substitute for these missing contracts.

## Evidence

This contract is based solely on the supplied static routine: its visible branches, argument construction, dependency-call order, 32-bit arithmetic, temporary-record uses, and direct object accesses.

Dependency bodies, runtime traces, external type declarations, and allocation contracts were not provided. No runtime tests have been performed. The acceptance examples are consequences of the visible control flow under their stated conditions, not execution evidence.
