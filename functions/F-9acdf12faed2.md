<!-- Trusted provenance: F-9acdf12faed2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9acdf12faed2

## Purpose

F-9acdf12faed2 conditionally returns a 32-bit word through either delegation or an indexed memory lookup.

A call to F-3b037d560480 gates both paths. If that dependency returns zero, the function returns zero immediately. Otherwise, an object-associated unsigned 16-bit quantity determines the path:

- A zero quantity causes delegation to F-f8ec0897aceb.
- A nonzero quantity permits a table lookup, subject to a nonzero initial selector and an unsigned index bound.

The lookup path exposes invocation-local selector and control storage to F-3dfc1640191e. Their resulting values affect slot selection and index reversal. A missing table pointer causes one call to F-616f7d92d3b9 before another availability check.

The selector is evaluated again after that call. Because F-3dfc1640191e receives its address, it may retain access to the still-live selector storage, and F-616f7d92d3b9 may subsequently modify it through that retained address. The post-call check is therefore not guaranteed to address the slot checked before the call.

“Object,” “descriptor,” “slot,” and “table” describe observed memory relationships, not established application-level types. Dependency names do not establish their purpose, ownership rules, or side effects.

## Interface

### Inputs

The supplied representation exposes four 32-bit inputs:

| Input | Representation | Use |
|---|---|---|
| Object | Address-like 32-bit value | Null check, dependency arguments, and base for memory accesses |
| Selector | Signed 32-bit value | Initial eligibility check and subsequent slot-address calculation |
| Control | Unsigned 32-bit value | Low-byte delegation argument or low-byte reversal decision |
| Index | Unsigned 32-bit value | Unsigned bounds check and entry selection |

Selector and control are copied into separate invocation-local 32-bit storage locations. F-3dfc1640191e receives their addresses. Writes through these addresses affect the copies, not the caller’s original argument storage.

A nonzero object is not sufficient to establish valid memory. Descriptor pointers and selector-derived addresses are dereferenced without comprehensive validation.

### Output

The result is one 32-bit word:

- The result of F-f8ec0897aceb, returned without further transformation.
- A 32-bit word loaded from the selected table entry.
- Zero when the gate rejects the operation, lookup eligibility fails, or the availability check after F-616f7d92d3b9 observes a zero pointer.

Zero is not a distinct error code. A delegated result or valid table entry may also be zero.

### Observed layout

| Location | Access width | Observed role |
|---|---:|---|
| Object + `0x40` | 32 bits | Descriptor address |
| Descriptor + `0x2c` | 16 bits, unsigned | First range quantity |
| Descriptor + `0x2e` | 16 bits, unsigned | Second range quantity |
| Object + `0x224` + four times the current selector | 32 bits | Table-pointer slot |
| Table address + `0x10` + four times the effective index | 32 bits | Result entry |

The entry region begins 16 bytes after the table address. This function does not read the preceding four words or establish their meaning.

Byte order, alignment requirements, allocation sizes, and the validity of extreme address calculations remain undocumented.

## Behavior

### 1. Gate the operation

The first dependency call is F-3b037d560480, with no explicit arguments.

A zero result produces an immediate zero return. This function then performs no explicit object, descriptor, slot, or entry reads.

A nonzero result permits further processing. The dependency is not established to preserve globals or object-associated memory. Subsequent reads observe memory after this call, not a guaranteed entry-time snapshot.

### 2. Obtain the first range quantity

If the object is zero, the first quantity is taken as zero without dereferencing the object.

Otherwise, the function:

1. Reads a 32-bit descriptor address at object offset `0x40`.
2. Reads an unsigned 16-bit quantity at descriptor offset `0x2c`.
3. Zero-extends that quantity for later arithmetic.

There is no explicit descriptor-null check before the 16-bit read.

### 3. Delegate when the first quantity is zero

A zero first quantity causes a call to F-f8ec0897aceb with five arguments, in this order:

1. Original object.
2. Selector copy.
3. Low eight bits of the control copy, as a value from 0 through 255.
4. Original index.
5. Original object again.

The delegated result is returned directly. There is no second quantity read, selector/control adjustment call, table-pointer check, or entry read by this function afterward.

A zero object reaches this path when the gate succeeds. Object nullness alone therefore does not imply a zero result.

The selector and control copies have not yet been exposed by address to F-3dfc1640191e on this path.

### 4. Establish lookup eligibility

When the first quantity is nonzero, the function obtains the second unsigned 16-bit quantity from descriptor offset `0x2e`.

It first rereads the descriptor address at object offset `0x40`. The two quantity reads therefore need not use the same descriptor if that memory changes between reads.

The second quantity is read before rejecting a zero selector.

For a nonzero selector, the lookup bound is the sum of the two zero-extended quantities. Each quantity is at most 65,535, so the sum is at most 131,070 and cannot overflow 32-bit unsigned arithmetic. On this path, the first quantity is nonzero, so the bound is at least one.

Lookup continues only when:

- The selector copy is nonzero.
- The original index is strictly less than the bound, under an unsigned comparison.

Failure of either condition returns zero without calling F-3dfc1640191e or F-616f7d92d3b9. A zero selector prevents evaluation of the comparison requiring the bound.

These checks establish index arithmetic eligibility, not the validity or capacity of any eventual table.

### 5. Expose and adjust selector and control

For an eligible lookup, F-3dfc1640191e receives:

1. Original object.
2. Address of the invocation-local selector copy.
3. Address of the invocation-local control copy.

No return value from this dependency is used.

The function subsequently uses the local copies, including modifications made through these addresses. It does not repeat the selector-nonzero test or the original index-bound check.

Consequently:

- A selector changed to zero is permitted to select the slot at object offset `0x224`.
- A negative selector is not explicitly rejected.
- A selector outside valid object storage can produce an invalid access.
- Descriptor changes do not update the already calculated bound.
- The dependency receives all 32 control bits, not merely the low byte.

The addresses remain addresses of live local storage during the later F-616f7d92d3b9 call. Nothing in the supplied evidence prevents F-3dfc1640191e from retaining them and making them available to that dependency. The selector is therefore not established to remain fixed after F-3dfc1640191e returns.

### 6. Determine the effective index

After F-3dfc1640191e returns, the function examines the low eight bits of the current control copy:

- If those bits are zero, the effective index is the original index.
- Otherwise, the effective index is the bound minus the original index minus one.

Any nonzero low-byte value enables reversal. This is not a single-bit test; high control bits alone do not enable it.

Because the original index already passed the unsigned bound check, reversal neither underflows nor leaves the interval from zero through the bound minus one.

The effective index is determined before F-616f7d92d3b9 can be called. If that later dependency modifies control through a retained address, the function does not repeat the reversal decision. The previously determined effective index remains in use.

The effective index is at most 131,069. Multiplication by four and addition of the `0x10` entry displacement do not overflow a 32-bit unsigned value by themselves. Adding the resulting displacement to a table address still requires valid address arithmetic and accessible storage.

### 7. Check table availability

The first table-pointer read uses the selector value current at that read.

If this read is nonzero, F-616f7d92d3b9 is skipped.

If it is zero:

1. F-616f7d92d3b9 is called with the original object.
2. The selector is evaluated again.
3. The corresponding table-pointer slot is read.
4. A zero result from this post-call read causes a zero return.

F-616f7d92d3b9 is called at most once. Its return value is unused.

The post-call read is not guaranteed to address the initial slot. For example, F-3dfc1640191e may have retained the selector address, and F-616f7d92d3b9 may write a different selector through that address. The second read then checks the slot selected by the new value, without another selector or range validation.

There is no explicit search over slots. Any change in the selected slot follows from reevaluating the selector, not from a scanning operation.

### 8. Load the result

A successful availability check is followed by another table-pointer read. The function again evaluates the selector to locate this final slot read; it does not reuse a pointer cached from the successful check.

The final table address is used to read a 32-bit word at offset `0x10` plus four times the previously determined effective index. That word is returned unchanged.

Successful retrieval therefore involves:

- Two slot reads if the initial availability read is nonzero.
- Three slot reads if the initial read is zero and the post-call availability read is nonzero.

These are counts of slot reads, not a guarantee that every read addresses one fixed slot.

There is no dependency call between the successful availability check and final slot read. Nevertheless, no synchronization or selector/pointer stability guarantee is established. If the final selector or slot contents change through external activity, the final read determines the table address. That address receives no further zero check.

## Side effects

### Direct effects

The function explicitly writes its own local selector and control storage. It exposes their addresses to F-3dfc1640191e.

No explicit writes to object, descriptor, table-slot, or table-entry memory are shown. Depending on the path, explicit external reads include descriptor addresses, unsigned 16-bit quantities, table pointers, and one result word.

No explicit allocation, release, ownership transfer, or cleanup operation is shown.

### Dependency effects and local-address lifetime

Dependency implementations are unavailable. This contract does not assume that they preserve globals, object contents, pointers, allocations, or memory lifetimes.

In particular, F-3dfc1640191e may retain the local addresses it receives. Those storage locations remain live during the subsequent F-616f7d92d3b9 call, allowing that dependency to modify selector or control through retained access.

These possible mutations have different consequences:

- A selector mutation can change which slot is checked after the call and which slot is used for the final pointer read.
- A control mutation after the reversal decision does not trigger another decision or alter the already computed effective index.
- Changes to descriptor quantities do not recalculate the bound.

No ownership or release behavior can be inferred from a dependency’s arguments alone. Valid retrieval requires every subsequently accessed memory region to remain valid through its actual use.

The local addresses are not established to remain valid after this invocation returns. Retention does not establish permission to dereference them after their lifetime ends.

### Errors and abnormal execution

There is no explicit diagnostic output, error object, retry loop, or exception handling.

Invalid pointers, insufficient table storage, unsuitable alignment, and invalid selector-derived addresses are not converted into a defined zero result. Their consequences depend on the execution environment.

The contract’s normal return descriptions assume that dependencies return normally and required memory accesses are valid.

## Dependencies

| Opaque ID | Call condition | Explicit arguments | Result use |
|---|---|---|---|
| F-3b037d560480 | Every invocation | None | Zero rejects; nonzero permits processing |
| F-f8ec0897aceb | Gate succeeds and first quantity is zero | Object, selector, low control byte, original index, object | Returned directly |
| F-3dfc1640191e | Nonzero first quantity, nonzero selector, index below bound | Object, selector-local address, control-local address | Local mutations affect subsequent behavior; return unused |
| F-616f7d92d3b9 | Initial slot read observes zero | Object | Return unused; selector is reevaluated for the following availability read |

At most three dependency calls occur: gate, selector/control adjustment, and conditional availability assistance. The delegation path makes two calls: gate and delegation.

“Adjustment” and “availability assistance” describe call placement. Normalization and table initialization are plausible intent, not established dependency semantics.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless a case states otherwise, dependencies return normally, accessed memory is valid, and relevant values remain stable.

1. **Gate rejection.** A zero result from F-3b037d560480 produces zero, no later dependency call, and no explicit object-associated read.

2. **Zero object.** With a successful gate and object zero, F-f8ec0897aceb receives zero as its first and fifth arguments. Its result becomes the result.

3. **Delegated control truncation.** With first quantity zero, selector 7, control `0x123456ab`, and index 12, delegation receives selector 7, control argument 171, and index 12. The second quantity is not read.

4. **Zero initial selector.** With first quantity 3 and selector zero, the second quantity is still read. The function then returns zero without either lookup-path dependency.

5. **Strict unsigned bound.** Quantities 3 and 2 produce bound 5. Index 5 is rejected; index 4 is eligible with a nonzero selector. Index `0xffffffff` is also rejected.

6. **No reversal.** With bound 5, original index 1, and adjusted control `0x01000000`, the effective index remains 1. A stable valid table supplies the word at offset `0x14`.

7. **Low-byte reversal.** With the same bound and index, adjusted control `0x80` produces effective index 3 and entry offset `0x1c`.

8. **Adjusted selector zero.** If an initially eligible selector becomes zero during F-3dfc1640191e, the initial availability read uses object offset `0x224`; no second nonzero-selector check occurs.

9. **One unsuccessful assistance call.** If the initial slot read is zero, F-616f7d92d3b9 is called once. If the slot selected by the selector value after that call is also zero, the function returns zero without an entry read. This need not be the initially checked slot.

10. **Assistance changes selector.** Suppose adjustment leaves selector 2 and retains its address. The initial read of slot 2 is zero. F-616f7d92d3b9 changes the local selector to 3 through that retained address. If slot 3 is nonzero and remains stable, lookup succeeds through slot 3 even if slot 2 remains zero.

11. **Late control change does not reverse again.** With bound 5 and index 1, adjustment leaves low control byte zero, fixing effective index 1. If F-616f7d92d3b9 later changes that byte to `0x80`, a successful retrieval still uses entry offset `0x14`.

12. **Final pointer reread.** If a successful check observes one nonzero table pointer but the final read observes another valid pointer, retrieval uses the latter. No cached checked pointer is substituted.

13. **Final pointer becomes zero.** If external activity changes the final pointer to zero after a successful check, the function does not take the earlier zero-return path. It attempts the entry access based on the final value; the outcome is not defined here.

## Uncertainties

The following remain **documented_only**:

- Application-level input meanings, selector domains, and result interpretation.
- Dependency implementations, failure modes, global effects, and ownership rules.
- Whether retained local addresses are actually used; the representation permits the described interaction but does not prove it occurs.
- Valid object, descriptor, slot, and table allocation extents.
- Byte order, alignment, concurrency rules, and memory-mapped access semantics.
- Extreme selector-address arithmetic, including signed multiplication and address overflow. Valid derived addresses are a precondition for defined retrieval.
- Whether dependencies return normally.
- Any intended consistency relationship among separately read descriptors, selector values, or table pointers.
- Any synchronization or hardware-level ordering guarantees.

## Evidence

This contract derives from the supplied static representation and opaque dependency mapping.

Established evidence includes the gate, null-object delegation, two unsigned 16-bit quantity reads, delegated argument order, unsigned index comparison, writable local arguments, low-byte reversal decision, conditional F-616f7d92d3b9 call, reevaluation of the selector after that call, repeated table-pointer reads, and direct 32-bit result paths.

Retained-address mutation is a permitted interaction given the exposed live local addresses and unknown dependency effects; it is not an observed execution.

No dependency bodies, runtime observations, executable checks, or tests were supplied or performed. Described read order follows the static representation, not an independently verified hardware memory-ordering contract.
