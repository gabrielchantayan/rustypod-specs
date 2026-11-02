<!-- Trusted provenance: F-834e12b6dfde; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-834e12b6dfde

## Purpose

F-834e12b6dfde conditionally selects an address derived from a caller-supplied state block and advances its leading signed 32-bit value. A rejected request returns null, provided execution completes normally. Rejection may first invoke F-4fc89ec41a6d and then set a notification marker.

The established behavior is address calculation and state mutation, not allocation or initialization. Interpreting the leading value as a record count, the returned address as a reserved record, or the marker as a warning-suppression flag is inferred intent.

The control value at byte offset 648 does not select a second successful range. Every signed value of at least 13 selects rejection. Smaller values defer the decision to the leading state value.

The cursor test admits every negative signed 32-bit value. This does **not** establish that every admitted value supports successful completion: the address calculation includes a signed 32-bit multiplication by five, which is not representable for sufficiently negative cursors. Behavior when that multiplication overflows remains unresolved.

## Interface

### Input and state layout

The function receives one pointer to a state block. The supplied representation establishes four-byte integer spacing and the following signed 32-bit accesses:

| Byte offset | Width | Contract name | Direct use |
|---|---:|---|---|
| 0 | 32 bits | Cursor | Read on the cursor-controlled path; incremented after supported successful address calculation |
| 648, or 0x288 | 32 bits | Control | Read first; selects immediate rejection or cursor evaluation |
| 700, or 0x2BC | 32 bits | Notification marker | Read on rejection; written as 1 after a qualifying dependency call returns |

These names identify fields for this contract; they do not establish application-level meanings.

The input must support every access reached during execution. The function contains no explicit null check, state-size check, alignment check, or lifetime validation. Readable storage alone is insufficient for paths that also write.

### Output

On ordinary rejection, the result is null.

On the success-selected path, the function calculates an address using the originally read Cursor, then stores Cursor plus one at byte offset 0. The address calculation has these relevant components:

- A **signed 32-bit Cursor-times-five intermediate**.
- A pointer advance by that many four-byte integer positions.
- A further pointer advance by one integer position.

Only when the multiplication and both pointer additions are supported may the resulting byte displacement be described mathematically as **4 + 20 × Cursor**.

This formula is not a complete specification for overflowing integer multiplication or unsupported pointer arithmetic. Neither mathematical evaluation beyond signed 32-bit range nor machine wraparound is established for those cases.

The function does not directly read or write through the calculated result address.

### Ownership and lifetime

The function performs no direct allocation, deallocation, record copying, or record initialization. A normally returned non-null address is derived from the input; no separate allocation or ownership transfer is established.

The caller must establish the returned address’s validity for any subsequent use. The effects of F-4fc89ec41a6d on ownership, globals, or memory lifetime are unknown.

## Behavior

### Decision rules

The function first reads Control as a signed 32-bit value.

- **Control ≥ 13:** select rejection without reading Cursor.
- **Control ≤ 12:** read Cursor as a signed 32-bit value:
  - **Cursor ≥ 32:** select rejection.
  - **Cursor < 32:** select the address-calculation path.

There is no lower-bound test on either field. Negative Control values remain cursor-controlled. Negative Cursor values satisfy the success-selection predicate.

The represented subtraction and overflow information establish signed comparisons. A test based solely on the sign of a wrapped subtraction would not describe these decisions correctly, particularly at extreme negative inputs.

### Integer arithmetic and address calculation

The arithmetic used to select a branch must be distinguished from the arithmetic used to calculate an address.

The branch decision admits Cursor values from **−2147483648 through 31**, inclusive. For each admitted value, Cursor plus one is representable as a signed 32-bit integer. In particular, the largest admitted Cursor increments from 31 to 32, and the minimum signed value increments to −2147483647.

However, the address calculation first multiplies Cursor by five as a signed 32-bit intermediate. That product is representable only for Cursor values from **−429496729 through 429496729**, inclusive. Intersecting that range with the success-selection range gives:

| Cursor range | Branch selection | Cursor-times-five intermediate |
|---|---|---|
| −2147483648 through −429496730 | Address-calculation path | Not representable as signed 32-bit |
| −429496729 through 31 | Address-calculation path | Representable as signed 32-bit |
| 32 through 2147483647 | Rejection, when Control permits Cursor evaluation | Not evaluated for the result address |

The first row is not an ordinary rejection case. It selects the address-calculation path, but the supplied evidence does not settle the behavior of the overflowing multiplication. This contract therefore does not assign a wrapped result address, guarantee the later cursor store, or guarantee normal return for that row.

Even when the multiplication is representable, both pointer additions must be supported. The final mathematical displacement alone does not establish that the intermediate pointer operation is valid.

For a negative Cursor with representable multiplication and supported pointer operations, the calculated address is before the input pointer. Without those conditions, that address-direction conclusion is not established.

### Normally completing success path

Subject to supported integer and pointer arithmetic, the order is:

1. Read Control.
2. Read Cursor because Control is at most 12.
3. Select the address-calculation path because Cursor is less than 32.
4. Calculate the result address from the saved, originally read Cursor.
5. Write that Cursor plus one to byte offset 0.
6. Return the calculated address.

The result uses the old Cursor, not the stored incremented value.

The Notification marker is not read or written on this path. F-4fc89ec41a6d is not called. A previously nonzero marker is not cleared.

The address calculation precedes the cursor store. Thus representability of the increment alone does not guarantee that the store is reached when earlier arithmetic is unsupported.

### Rejection path

After selecting rejection, the function reads Notification marker.

If the marker is nonzero:

- F-4fc89ec41a6d is not called.
- The function makes no direct state write.
- The function returns null.

Every nonzero value suppresses the call, including negative values and values other than 1. The field is not normalized.

If the marker is zero:

1. Invoke F-4fc89ec41a6d with the five positional values **3, 116, 109, 0, 0**.
2. If that invocation returns normally and execution can continue, write the signed 32-bit value 1 to Notification marker.
3. Return null.

The marker write occurs after the dependency call. There is no intervening marker reread. A marker value installed by the dependency can therefore be overwritten with 1.

Rejection performs no direct Cursor write. When Control is at least 13, rejection does not read Cursor either.

### Dependency interference and abnormal completion

The dependency receives no explicit state-block pointer among the five visible arguments. This does not establish that the state is unreachable through globals, aliases, or other means.

Changes made by the dependency to Control or Cursor do not cause the rejection decision to be reconsidered. Normal return leads directly toward the marker write without rechecking those fields.

If the dependency does not return normally, the later marker write and null return are not established. If it invalidates the state storage, the function still has a subsequent marker-write operation with no lifetime revalidation. Successful completion of that access is not guaranteed.

## Side effects

### Direct state accesses

| Selected path | Direct reads, in order | Direct writes on ordinary completion |
|---|---|---|
| Control ≥ 13 | Control, Notification marker | Marker becomes 1 only if its zero value triggered a normally returning dependency call |
| Control ≤ 12 and Cursor ≥ 32 | Control, Cursor, Notification marker | Same marker rule |
| Control ≤ 12 and Cursor < 32, with supported arithmetic | Control, Cursor | Cursor becomes its originally read value plus one |

For success-selected inputs with overflowing multiplication or unsupported pointer arithmetic, the final row does not establish completion of its write.

There is no direct write to Control and no direct initialization of storage at the returned address. Dependency effects are additional and unknown.

### Notification and synchronization

The marker suppresses dependency invocation only when it is observed as nonzero on rejection. This is not an unconditional lifetime-wide “once only” guarantee.

The marker remains unwritten when the dependency is entered. Reentrant activity can therefore encounter a zero marker and trigger another invocation before the outer call writes 1.

No locking or atomic check-and-set is established. No distinct-address guarantee for concurrent callers follows from the cursor read and later write. Caller-provided synchronization, if any, is outside this contract.

### Failure reporting

Null is the ordinary rejection result. The marker-controlled dependency invocation is the only other visible reporting action.

Both rejection reasons use the same return value and the same dependency arguments. The function itself does not expose which threshold caused rejection.

Invalid storage, unsupported arithmetic, and dependency failure have no explicit recovery path in the supplied representation.

## Dependencies

### F-4fc89ec41a6d

- **Invocation condition:** Rejection has been selected and Notification marker was read as zero.
- **Visible positional arguments:** 3, 116, 109, 0, 0.
- **Return handling:** No returned value is used.
- **Post-call operation:** After normal return, attempt to write 1 to Notification marker, without rereading it.
- **Direct invocation count:** At most one invocation in a single ordinary execution of F-834e12b6dfde. Invocations caused by reentry, recursion, or other executions are separate.
- **Unknown contract:** Declared parameter types, return type, reporting semantics, global accesses, ownership effects, storage-lifetime effects, and exceptional or nonreturning behavior.

No preservation or memory-ownership guarantee is assumed for this dependency.

## Acceptance examples

These are conditional static acceptance criteria, not runtime observations. All examples require valid storage for reached accesses. Examples asserting normal successful completion additionally require supported intermediate integer arithmetic, both pointer additions, and the cursor write.

1. **First nonnegative Cursor**
   - Control is 12 and Cursor is 0.
   - The product is 0; the result is the input address plus 4 bytes.
   - Cursor becomes 1.
   - Notification marker is neither read nor changed, and the dependency is not called.

2. **Last nonnegative Cursor accepted**
   - Control is 0 and Cursor is 31.
   - The product is 155; the result is the input address plus 624 bytes.
   - Cursor becomes 32.
   - A subsequent call with unchanged Control and Cursor selects rejection.

3. **Cursor threshold triggers notification**
   - Control is 12, Cursor is 32, and Notification marker is 0.
   - The dependency receives 3, 116, 109, 0, 0.
   - If it returns normally and the marker write remains valid, the marker becomes 1 and the result is null.
   - Cursor is not directly changed.

4. **Control threshold overrides an available Cursor**
   - Control is 13 and Notification marker is 1.
   - Cursor is not read, regardless of its stored value.
   - The dependency is not called.
   - The result is null, with no direct memory write.

5. **Noncanonical marker suppresses notification**
   - Rejection is selected and Notification marker is −1.
   - The dependency is not called.
   - The marker remains unchanged by this function, and the result is null.

6. **Extreme negative Control remains cursor-controlled**
   - Control is −2147483648 and Cursor is 0.
   - Cursor evaluation selects the address-calculation path.
   - With supported pointer operations, the result is the input address plus 4 bytes and Cursor becomes 1.

7. **Small negative Cursor**
   - Control is 12 and Cursor is −1.
   - The multiplication produces −5, which is representable.
   - If both pointer additions are supported, the result is 16 bytes before the input pointer and Cursor becomes 0.
   - This does not independently authorize a later access through that address.

8. **Representable multiplication boundary**
   - Control is 12 and Cursor is −429496729.
   - The multiplication produces −2147483645, which is representable.
   - Pointer arithmetic remains a separate precondition; valid input-field accesses alone do not establish it.
   - Only if the address calculation and write are supported does Cursor become −429496728 and the function return the calculated address.

9. **First overflowing negative multiplication**
   - Control is 12 and Cursor is −429496730.
   - The branch selects address calculation rather than rejection.
   - The mathematical product, −2147483650, is outside signed 32-bit range.
   - No particular result address, wraparound rule, cursor write, or normal return is specified.

10. **Minimum Cursor does not imply a safe success**
    - Control is 12 and Cursor is −2147483648.
    - The cursor comparison selects address calculation.
    - Cursor plus one would be representable, but Cursor times five is not.
    - Representability of the increment does not establish execution of the later store.

11. **Dependency changes the marker**
    - Rejection occurs with Notification marker initially zero.
    - The dependency changes the marker to 7.
    - If it returns normally and the subsequent write remains valid, this function overwrites 7 with 1 and returns null.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of the three state fields.
- Whether callers enforce a nonnegative Cursor invariant.
- Whether the 20-byte spacing represents complete records, subobjects, or another layout.
- Total storage size and validity of result addresses for consumers.
- Pointer width and the supported domain of each pointer addition.
- Behavior when the signed 32-bit Cursor-times-five intermediate is not representable.
- Whether unsupported arithmetic prevents, alters, or otherwise affects later observable operations.
- Dependency parameter types, reporting semantics, global effects, ownership effects, lifetime effects, and abnormal completion.
- Caller-provided synchronization or reentrancy exclusion.
- Environmental handling of invalid accesses.

In particular, overflowing multiplication is not resolved as either unbounded mathematical arithmetic or machine wraparound. The conditional displacement formula does not remove this uncertainty.

Record reservation and warning suppression remain inferred intent. They do not establish bounds safety, initialization, ownership transfer, atomic reservation, or exactly-once notification.

## Evidence

This contract is based solely on the supplied static representation and opaque dependency mapping.

Established static evidence includes:

- Signed 32-bit state accesses at offsets 0, 648, and 700.
- Signed threshold decisions with the represented subtraction-overflow information accounted for.
- A signed 32-bit Cursor-times-five intermediate before pointer additions.
- Four-byte integer spacing and the additional one-position pointer advance.
- Address calculation before the cursor increment store.
- Null rejection, marker-controlled dependency invocation, fixed argument values, and the post-call marker write.

The multiplication bounds follow from signed 32-bit representability. The displacement formula follows only when the intermediate multiplication and pointer operations are supported.

No runtime tests or executions have been performed. No dependency implementation has been inspected, and unresolved arithmetic behavior has not been experimentally characterized.
