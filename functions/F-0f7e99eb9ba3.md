<!-- Trusted provenance: F-0f7e99eb9ba3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0f7e99eb9ba3

## Purpose

F-0f7e99eb9ba3 searches 30 fixed-stride record positions for the first record whose first integer word equals the input. On a match, it invokes F-89ee793df58f, clears two four-byte fields using a table address captured before the search, and performs a final integer assignment through an externally held control reference.

That final assignment has **two separate source-level uses of the control reference**: one determines where an integer is read, and the other determines where the result of subtracting one is written. Only when the control reference remains stable during the assignment can this be described as decrementing one resolved word.

If no record matches, the function invokes F-a817d8d58c10, which the supplied static material identifies as non-returning. The mechanism of non-return is not established.

“Removal,” “deregistration,” and “entry-count maintenance” are **inferred intent**, not established semantics. The source does not establish that zero means unused, that the control value counts occupied records, or that either dependency releases resources.

## Interface

### Input and result

- One integer argument supplies the comparison value and the first argument to F-89ee793df58f.
- No return value is declared.
- The matching path returns normally only if the dependency returns and the subsequent memory operations complete.
- The no-match path has no established normal-return result.

The source explicitly establishes four-byte clearing stores and eight-byte record spacing. Integer comparisons, address calculations, and the final arithmetic use the source’s integer type. A 32-bit integer interpretation is consistent with the supplied representation, but no independent target ABI declaration is available.

Accordingly, concrete integer boundary examples below are conditional on a signed 32-bit integer model. Exact integer and address representations remain **documented_only**.

### Referenced state

Two externally held address values participate in the function:

| Reference | Referenced layout | Use |
|---|---|---|
| Table reference | 30 positions at an eight-byte stride | Search addressing and the initial capture of the clearing address |
| Control reference | An integer location at byte offset eight | Separate address determinations for the final read and write |

Each table record has the following established field layout:

| Record-relative offset | Width | Use |
|---|---:|---|
| Zero | Four bytes under the 32-bit integer model | Read for equality; cleared on selection |
| Four | Four bytes | Cleared on selection; not inspected by the search |

The first record begins at offset zero. The final record begins at offset 232, and its second field begins at offset 236. A contiguous allocation containing all 30 complete records therefore occupies 240 bytes.

No length field, occupancy flag, or terminator is consulted. The function explicitly accesses no control-object field other than the integer at offset eight, although the separate reference evaluations need not identify the same object without a stability assumption.

### Preconditions for ordinary completion

The function performs no explicit checks for null references, alignment, bounds, storage lifetime, or access permissions.

Ordinary completion requires:

- Every visited comparison location to support the required integer read.
- Both clearing destinations, calculated from the initially captured table address and selected index, to support four-byte writes after F-89ee793df58f returns.
- The location selected by the final assignment’s read-side control-reference use to support an integer read.
- The location selected by its write-side control-reference use to support an integer write.
- The address calculations and subtraction to have valid semantics in the execution environment.

These locations need not belong to disjoint objects. Their validity is not guaranteed by the function or by any supplied dependency contract.

## Behavior

### Initial capture and search

Before performing the search, the function captures the address held by the table reference. This retained address determines the destinations of both clearing stores if a match is found.

The search considers indices zero through 29 in increasing order. For each visited index, it compares the input with the first integer at that index’s eight-byte offset.

The comparison address uses the table reference appearing in the search expression, rather than the address captured for clearing. Consequently, under a model permitting the table reference to change between relevant evaluations:

- A comparison may address storage different from the captured clearing storage.
- The selected index still determines the offsets of the eventual clearing stores.
- The clearing address is not recaptured after selection or after the dependency call.

These are distinctions between source-level expressions. They do not establish a particular number of physical reference loads, compiler reload behavior, or defined concurrent-mutation semantics.

The search stops at the first matching index. It does not inspect the second field to confirm a match or determine whether a record is occupied.

Zero has no special search meaning. A first field containing zero matches a zero input. It does not terminate a search for a different input.

If all comparisons fail, the search considers exactly 30 record positions and does not inspect index 30.

### Matching path: dependency and clearing

After selecting a matching index, the function performs these ordered operations:

1. Calls F-89ee793df58f with the original input and integer zero.
2. If the dependency returns, writes a four-byte zero to the selected record’s first field using the initially captured table address.
3. Writes a four-byte zero to the selected record’s second field using that same captured address.
4. Performs the final control-reference assignment described below.
5. Returns without searching any later record.

The dependency call precedes both explicit clearing stores. The function therefore does not itself clear the selected fields before invoking F-89ee793df58f. This does not establish what that dependency observes or changes.

Neither the match nor storage validity is rechecked after the dependency returns. If the dependency modifies the selected record, the explicit stores still clear the two destinations. If it changes the table reference, those stores still use the entry-captured address. If it invalidates that storage, ordinary completion is not established.

The second field is cleared without an explicit preceding read of its contents.

### Final assignment: distinct read and write address determinations

The final assignment contains two separate source-level uses of the control reference:

- The **read-side use** determines an address eight bytes beyond its referenced base. The integer at that address supplies the arithmetic input.
- The **write-side use** determines an address eight bytes beyond its referenced base. That address receives the arithmetic input minus one.

The supplied source does **not** establish the relative evaluation order of these two control-reference uses. In particular, it does not establish whether the read-side base or the write-side base is evaluated first.

The assignment follows both clearing stores. Its stored value depends on the integer read and the subtraction. These facts do not resolve the relative evaluation order of the two address determinations.

If the control reference remains stable throughout the assignment, both uses select the same integer location. Subject to valid arithmetic and memory access, the operation then decrements that location by one.

Without that stability assumption, the relationship between the read and write addresses is **documented_only**. The contract must not promise that the destination’s previous value is the value decremented, or that the location supplying the value is itself written.

This distinction establishes neither two physical loads of the control reference nor defined behavior under concurrent mutation. No volatility declaration, synchronization contract, or applicable concurrency model is supplied.

### Effects of mutation and aliasing

Changes made by F-89ee793df58f can affect the final assignment because that assignment uses the control reference after the dependency returns and after both clearing stores.

For example, if the dependency changes the control reference and the new reference then remains stable throughout the final assignment, the assignment reads and writes the integer at offset eight in the newly referenced object.

Aliasing also matters:

- If the stable control reference selects the first cleared field, that field supplies zero to the subtraction.
- If it selects the second cleared field, that field likewise supplies zero.
- Under the signed 32-bit model, either case ends with negative one at the aliased location, assuming no intervening mutation and valid accesses.

These examples require control-reference stability during the assignment. They do not establish an in-place decrement when its two reference uses can resolve differently.

No disjointness rule is enforced. If a clearing destination overlaps storage containing an external reference, a later reference evaluation may be affected. The resulting address interpretation and validity require information not supplied here.

### No-match path

After all 30 comparisons fail, the function invokes F-a817d8d58c10 with no explicit arguments.

Before that invocation, the function has performed no explicit record-clearing store, no final control assignment, and no call to F-89ee793df58f.

The supplied static material identifies F-a817d8d58c10 as non-returning. It does not establish termination, trapping, exception propagation, diagnostic output, or another particular mechanism.

Behavior if that dependency unexpectedly returns is **documented_only**. No status value or recovery path is established.

### Integer and address boundaries

The visited index ranges from zero through 29. Multiplication by the eight-byte stride produces offsets from zero through 232. Including the second-field displacement gives a maximum field-start offset of 236.

These small offsets do not themselves approach a signed 32-bit boundary. Their addition to an address still requires a valid address representation, representable address calculations where required, and accessible storage. Address wrapping is not established.

The final subtraction has no explicit lower-bound check and is not saturating.

Under the signed 32-bit integer model:

- An input value of one produces zero.
- Zero produces negative one.
- A negative value above the minimum produces the next lower integer.
- Subtracting one from negative 2,147,483,648 exceeds the representable signed range.

Wrapping to positive 2,147,483,647 at that last boundary is not established. The boundary behavior remains **documented_only**.

Here, “input value” means the integer obtained by the assignment’s read-side access. It means the destination’s previous value only when the relevant addresses coincide and no intervening mutation changes the value.

## Side effects

On a normally completing matching path, the function explicitly performs:

- One call to F-89ee793df58f.
- Two ordered four-byte zero stores.
- An integer read and a subsequent store of that value minus one, with separate source-level control-reference uses determining their addresses.

The function contains no explicit assignment to either external reference. Nevertheless, aliasing or dependency effects could change reference storage; preservation is not guaranteed.

No explicit allocation or deallocation is established. In particular:

- Clearing a field does not prove that it contained an owned resource.
- Passing the input to F-89ee793df58f does not establish ownership transfer.
- Neither dependency is known to preserve referenced storage or object lifetimes.
- Caller ownership obligations cannot be derived from this body alone.

There is no explicit synchronization, locking, atomic transaction, rollback, or recovery. If a dependency fails to return or a later access cannot complete, earlier effects need not be undone.

## Dependencies

| Opaque function ID | Invocation | Established role and limits |
|---|---|---|
| F-89ee793df58f | Once after finding a match; arguments are the original input and integer zero | Precedes both clearing stores. No returned result is used. Mutation, ownership, lifetime, and failure behavior are unknown. |
| F-a817d8d58c10 | Once after all 30 comparisons fail; no explicit arguments | Identified by the supplied static annotation as non-returning. Its mechanism and other effects are unknown. |

No other direct function dependency appears in the supplied body.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Unless a case says otherwise, assume stable references, valid disjoint storage, representable arithmetic, and a returning F-89ee793df58f that preserves relevant state.

1. **Match at the beginning.**  
   Record zero’s first field equals the input. The dependency receives the input and zero. Both fields of record zero are cleared, the stable control location decreases by one, and the function returns. Later records are not compared.

2. **Match at the final position.**  
   Records zero through 28 fail to match; record 29 matches. All 30 comparisons occur. The captured-table fields at offsets 232 and 236 are cleared before the final assignment.

3. **Duplicate matches.**  
   Records four and nine both match. Record four is selected. The function does not continue to record nine or explicitly clear it.

4. **Zero input.**  
   The first encountered zero first field matches, regardless of the second field. Even if both fields are already zero, the dependency call and final assignment still occur.

5. **No matching record.**  
   All 30 first fields differ from the input. F-a817d8d58c10 is invoked. No explicit clearing or control assignment precedes that invocation.

6. **Dependency changes the table reference.**  
   A match occurs at index six. The dependency redirects the table reference and returns. Both clearing stores still target index six relative to the initially captured address. That original storage must remain writable for ordinary completion.

7. **Dependency changes the control value.**  
   The dependency changes the eventual read-side integer from seven to 12. The control reference then remains stable through the assignment, with no later interference or aliasing. The location ends at 11, not six.

8. **Dependency redirects the control reference.**  
   The dependency selects a different control object and returns. If that reference remains stable during the final assignment, the integer at offset eight in the new object is read and decremented in place. The old object is not the final assignment’s destination under these conditions.

9. **Control location aliases a cleared field.**  
   The stable control reference selects the selected record’s second field. The clearing store writes zero before the final assignment reads it. Under signed 32-bit arithmetic, that field ends at negative one.

10. **Control-reference stability is unavailable.**  
    If the two source-level reference uses can resolve to different bases under an applicable execution model, acceptance must not require an in-place decrement. The established expression relationship is a value read through the read-side address, reduced by one, and assigned through the write-side address. Whether such differing evaluations are defined, and what orders or outcomes are permitted, remains **documented_only**.

11. **Matching-path dependency does not return.**  
    Execution never reaches this function’s explicit clearing stores or final assignment. The dependency’s own effects remain unspecified.

## Uncertainties

The following remain **documented_only**:

- Exact target ABI, integer widths, address representation, alignment requirements, and byte order.
- Semantic meanings of the input, record fields, and control value.
- Whether zero denotes an unused record.
- Ownership, mutation, lifetime, and failure contracts of both dependencies.
- Validity of storage after dependency execution.
- Any invariant connecting the control value to table contents.
- Relative evaluation order of the final assignment’s two control-reference uses.
- Whether their read and write addresses coincide without reference stability.
- Physical load/store counts and compiler treatment of reference expressions.
- Defined behavior under concurrent mutation or reentrancy.
- Signed-minimum subtraction and address-overflow behavior.
- Behavior if the designated non-returning dependency returns.

These unknowns do not justify assuming preserved globals, retained allocation lifetimes, resource release, or atomic execution.

## Evidence

The supplied static body establishes an initial table-address capture, an ascending 30-position search at eight-byte spacing, first-field equality comparisons, a matching-path dependency call, two ordered four-byte clearing stores, and a final assignment containing separate read-side and write-side uses of the control reference. It also establishes a no-match dependency call annotated as non-returning.

The final assignment supports an in-place decrement model only when the control reference is stable during that assignment. Its source form does not establish the relative evaluation order of the two reference uses or two physical reference loads.

No dependency implementations, independent layout declarations, target ABI specification, or runtime observations were supplied. No runtime tests have been performed.
