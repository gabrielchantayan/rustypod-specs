<!-- Trusted provenance: F-88efe4fe0e59; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-88efe4fe0e59

## Purpose

F-88efe4fe0e59 conditionally updates state associated with one object. An eligibility dependency gates all subsequent processing. An initial count of zero selects an embedded-object operation driven by external text storage. Any nonzero initial count selects current-value acquisition, optional secondary-object updates, and a backward indexed search. A changed search result triggers notification before being recorded.

The supplied static representation establishes these operations and their ordering. Descriptions such as “text,” “size,” “position,” and “notification” identify observed roles or inferred intent; they do not establish application-level semantics.

## Interface

### Input and result

The function accepts one object address and has no explicit return value.

The supplied representation uses 32-bit integers, addresses, and dispatch-table entries. Signedness matters for count and position calculations; the current value and indexed lookup results are compared as unsigned 32-bit values.

There is no visible input-address validation. Every object, dispatch table, external storage location, and indirect target reached during execution must remain usable for its actual access or invocation. These requirements are path-dependent.

### Referenced object layout

Offsets are in bytes from the input object address.

| Offset | Width and established use |
|---|---|
| `+0x18` | 32-bit argument read for eligibility and, on the populated path, read again for current-value acquisition. |
| `+0x1c` | 32-bit argument reloaded for each indexed lookup. |
| `+0x20` | Signed 32-bit count-like value. Read for the initial branch and again before searching. |
| `+0x28` | 32-bit secondary-object value. Tested against zero, passed to dependencies, and used as an address base. |
| `+0x30` | Signed 32-bit lower-position value. |
| `+0x38` | Signed 32-bit upper-position value. |
| `+0x3c` | Signed 32-bit size limit and arithmetic input, read at multiple stages. |
| `+0x40` | 32-bit argument to an arithmetic dependency; semantic type unknown. |
| `+0x44` | Start of an embedded polymorphic object. Its leading 32-bit word supplies a dispatch-table address. |
| `+0xac` | Signed 32-bit stored search result, compared and conditionally overwritten. |

Only the reached fields need be accessible. Eligibility rejection does not reach the count or later fields. A normally completing populated path reaches the stored-result field.

### External storage

The body references three external storage roles:

1. **Text storage.** Its first byte selects the empty-count operation. Its address is passed to a text-related dependency and, conditionally, to an indirect transfer operation.
2. **Notification reference.** A 32-bit value obtained through the external notification reference is passed to F-b11d9c4161cd when the search result differs from the stored result.
3. **Transfer-target storage.** A global location supplies the indirect transfer call target. This target is read only after the embedded dispatch at table offset `+0x08` returns a nonzero result, and before invoking that target.

The transfer target is not established as a fixed function. Neither the preceding text dependency nor the embedded dispatch may be assumed to preserve its global storage.

No text encoding, text-storage capacity, ownership rule, initialization protocol, or valid-target set is established.

## Behavior

### 1. Eligibility gate

The function reads `+0x18` and passes that 32-bit value to F-e234529174b6.

A zero return ends execution immediately. The function itself performs no subsequent count, text, secondary-object, or stored-result access on that path. This restriction does not constrain accesses performed inside the eligibility dependency.

A nonzero return permits a signed 32-bit read of `+0x20`. Zero selects the text path; every nonzero value, including a negative value, selects the populated path.

### 2. Zero-count text path

This path operates on the embedded object at `+0x44`. It does not acquire a current value, update a secondary object, search entries, notify a result change, or directly write `+0xac`.

The function first reads the initial byte of external text storage.

#### Zero initial byte

The embedded object’s leading word supplies a dispatch-table address. The function invokes the entry at table byte offset `+0x0c`, passing the embedded object address, then returns.

A reset or clear operation is plausible intent, not established behavior. In particular, releasing storage is not established.

#### Nonzero initial byte

The following sequence applies:

1. F-5cb57bbb2e0e receives the text-storage address.
2. Its 32-bit result is incremented by one.
3. The embedded object’s dispatch-table pointer is read after that dependency returns.
4. The entry at table byte offset `+0x08` is invoked with the embedded object address, the incremented result, and zero.
5. A zero dispatch result ends execution.
6. For a nonzero dispatch result, the function reads the indirect call target from transfer-target storage.
7. That target is invoked with the nonzero dispatch result and the text-storage address.
8. The function returns.

The global transfer target is therefore obtained after both F-5cb57bbb2e0e and the embedded dispatch have completed. A target change made by either dependency is observable through this later read. The body does not read the target on the zero-dispatch-result path.

The initial text byte is not checked again before transfer. The same storage address is passed onward, but its contents need not remain unchanged across the intervening calls.

This sequence resembles requesting destination storage and then copying text. It does not establish length measurement, allocation, copying, termination, or ownership semantics. Adding one to the first dependency’s result is visible arithmetic, not proof of a terminator requirement. Overflow of that increment remains unresolved.

### 3. Populated-path setup

The function reads `+0x18` again and passes the new value to F-d93d7d546d3b. Its result is retained as an unsigned 32-bit current value.

F-0588aaff39a5 is then called with the input object address.

The retained current value is used for subsequent arithmetic and search comparisons. It is not reacquired after intervening dependencies, even if they modify `+0x18`.

After F-0588aaff39a5 returns, the function reads `+0x28`. Zero skips all secondary-object work and proceeds to the search. Nonzero enables the following operations.

### 4. Optional secondary-object update

#### Preparation and measurement

The function calls F-d7be9cfb4000 with the retained current value and `1000`, retaining its 32-bit result for later arithmetic.

It then calls F-70446cb9c03e with the input object address and retained current value.

After that call, the function:

- Reads and retains the secondary-object value from `+0x28`.
- Calls F-44362605ab79 with the embedded object address at `+0x44`.
- Calls F-a0576cc39408 with the retained secondary-object address plus `0xa4`, and the result of F-44362605ab79.

The secondary-object base used for the `+0xa4` argument is captured before F-44362605ab79. A mutation of `+0x28` by that dependency does not change this particular argument.

#### Effective size

The signed 32-bit result of F-a0576cc39408 is compared with the signed field at `+0x3c`.

If the field is smaller, the effective size is taken from a further read of that field. Otherwise, the dependency result remains the effective size. With stable storage, this selects the signed minimum of the two values.

There is no lower clamp.

A strictly positive effective size causes a fresh read of `+0x28` followed by F-d25031d00f00 with that value and the effective size. Zero and negative sizes skip this call, but do not skip position calculation.

#### Position calculation

The retained result of the first F-d7be9cfb4000 call is multiplied by a fresh signed read of `+0x3c`. The product and the 32-bit value at `+0x40` are passed to a second F-d7be9cfb4000 call.

The arithmetic performed inside this dependency is unknown. Neither call may be specified as division or scaling without further evidence.

After the second call returns, the function reads and retains the signed lower-position field at `+0x30`. The candidate position is the second arithmetic result plus this lower value, minus half the effective size, plus two. The visible signed division of effective size by two truncates toward zero.

The constraints are applied in this order:

1. If the candidate is signed-less-than the retained lower value, the candidate becomes that lower value.
2. If the signed upper-position field at `+0x38` is less than candidate plus effective size, the candidate becomes a further read of the upper-position field minus effective size.

The lower constraint is not reapplied after the upper constraint. An oversized effective size can consequently produce a final position below the lower value.

Finally, F-5cf6facacac5 receives a fresh read of `+0x28`, the resulting position, and zero.

The original nonzero secondary-object check does not protect later reads against mutation. Later calls may receive a changed or zero secondary-object value. No visible revalidation occurs.

### 5. Backward search

After optional secondary-object processing, the function reads `+0x20` again as a signed 32-bit search starting value. This read is independent of the initial branch decision.

The search decrements the index before examining an entry. A negative decremented value stops the search without a lookup. Otherwise, F-9f4049cee92f receives a fresh read of `+0x1c` and the index.

Its result is treated as unsigned 32-bit:

- If the retained current value is strictly smaller than the lookup result, the search decrements again.
- Otherwise, the search stops at that index.

Equality stops the search. A lookup result with its high bit set is a large unsigned value, not a negative value.

For a positive search count, normal dependency completion, and no relevant arithmetic overflow, the search stops at the highest index encountered whose returned value is no greater than the retained current value, or at `−1` if none qualifies. No sorted-order precondition is needed for that operational description.

If the reloaded count is zero, the result is `−1` without lookup. Ordinary negative counts produce their decremented negative value, not necessarily `−1`. The minimum signed 32-bit count presents an unresolved overflow boundary.

### 6. Notification and stored-result update

The final index is compared with the signed field at `+0xac`.

If equal, execution ends without notification or an explicit stored-result write.

If different, the function obtains the notification argument through the external notification reference and calls F-b11d9c4161cd. Only after that call returns does it write the computed index to `+0xac`.

There is no post-notification comparison. A change made to `+0xac` by the notification dependency is overwritten by the subsequent store, provided the object remains valid.

## Side effects

The only explicit direct write to the input object is the conditional store at `+0xac`.

Other effects are delegated:

- Embedded-object operations occur on the zero-count path.
- F-0588aaff39a5 runs on every populated path that reaches it, regardless of secondary-object presence.
- Optional preparation, measurement-like, sizing-like, and positioning-like calls may affect the input object, secondary object, or other state.
- A changed search result invokes notification before recording the result.
- The nonzero text-dispatch path reads the global transfer target immediately before the indirect invocation, after the dispatch has returned.

No local allocation, release, or ownership transfer is established. The nonzero embedded-dispatch result is forwarded to the indirect target but is not visibly released by this body.

There is no explicit error result, retry, rollback, or cleanup path. In particular, a zero result from embedded dispatch `+0x08` causes a normal return without transfer or visible error reporting.

Dependencies are not assumed to preserve fields, globals, dispatch pointers, transfer-target storage, object lifetime, or ownership. Statements about subsequent operations are conditional on dependencies returning normally and the reached accesses remaining valid.

## Dependencies

| Opaque dependency or dispatch | Established role |
|---|---|
| F-e234529174b6 | Eligibility decision from `+0x18`. |
| F-d93d7d546d3b | Supplies the retained unsigned current value. |
| F-0588aaff39a5 | Receives the input object before secondary-object presence is checked. |
| F-d7be9cfb4000 | Two arithmetic calls; internal arithmetic and failure behavior unknown. |
| F-70446cb9c03e | Receives the input object and retained current value before measurement preparation. |
| F-44362605ab79 | Receives the embedded object; result forwarded to F-a0576cc39408. |
| F-a0576cc39408 | Supplies the signed measurement-like result. |
| F-d25031d00f00 | Receives a freshly read secondary-object value and positive effective size. |
| F-5cf6facacac5 | Receives a freshly read secondary-object value, position, and zero. |
| F-9f4049cee92f | Supplies unsigned indexed values during descending search. |
| F-b11d9c4161cd | Called before a changed search index is stored. |
| F-5cb57bbb2e0e | Receives the text-storage address; result incremented for embedded dispatch. |
| Embedded dispatch `+0x0c` | Receives the embedded object when the initial text byte is zero. |
| Embedded dispatch `+0x08` | Receives the embedded object, incremented result, and zero. |
| Global-target indirect operation | Target loaded after nonzero embedded-dispatch return; receives that return value and the text-storage address. |

The indirect operation has no supplied opaque function ID identifying a fixed callee. Its identity is determined by the target value read for that invocation.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless a case specifies mutation, relevant storage is stable and dependencies return normally.

### Eligibility rejection

If F-e234529174b6 returns zero, the function performs no subsequent count read, text-byte read, transfer-target read, search, notification, or explicit stored-result write.

### Empty count and zero text byte

With successful eligibility, initial count zero, and initial text byte zero, the embedded dispatch at `+0x0c` is invoked. F-5cb57bbb2e0e, dispatch `+0x08`, and the global-target transfer are not invoked.

### Zero dispatch result

With initial count zero and a nonzero text byte, suppose F-5cb57bbb2e0e returns `7`. Embedded dispatch `+0x08` receives size-like argument `8` and trailing zero.

If it returns zero, the function neither reads transfer-target storage nor invokes a transfer target.

### Transfer target changed during preparation

Suppose the text byte is nonzero. F-5cb57bbb2e0e changes transfer-target storage to target A, then embedded dispatch `+0x08` changes it to target B and returns nonzero value R.

The later target read obtains B. B is invoked with R and the text-storage address. A previously held target is not used merely because it existed before either call.

### Unsigned search and equality

With search count `3`, current value `20`, and lookup results `30` at index `2` and `20` at index `1`, lookups occur at `2`, then `1`. The search stops at `1` without examining `0`.

A stored result of `1` suppresses notification and store. Any different stored result causes notification followed by storing `1`.

### No qualifying entry

With search count `2`, current value zero, and both lookup results equal to unsigned `0xffffffff`, indices `1` and `0` are examined, then the result is `−1`.

### Reloaded count changes

If the initial count is positive but a preceding dependency changes it to zero, the search performs no lookup and produces `−1`.

If the reloaded count is instead `−2`, the decrement produces `−3`; no lookup occurs, and `−3` is compared with the stored result.

### Upper constraint overrides lower constraint

With effective size `20`, lower position `10`, upper position `15`, and candidate position zero, assuming no arithmetic overflow, the lower constraint first selects `10`. The upper constraint then selects `−5`. The final positioning argument is `−5`.

### Negative effective size

If measurement returns `−3` and the size limit is `10`, the effective size remains `−3`. The positive-size dependency is skipped, but positioning still occurs. Half the effective size is `−1` under truncation toward zero.

### Notification mutation

If the computed result differs from `+0xac` and F-b11d9c4161cd changes that field, its change is overwritten by the computed result after it returns.

## Uncertainties

- **documented_only — overflow and address arithmetic:** Visible arithmetic uses 32-bit quantities without explicit widening or checks. Overflow behavior is unresolved for the text-result increment, multiplication, position arithmetic, search decrement, and address additions. Unbounded arithmetic and wraparound are not interchangeable assumptions.
- **documented_only — arithmetic dependency:** F-d7be9cfb4000 does not have established division, scaling, rounding, or zero-argument semantics.
- **documented_only — indirect operations:** Embedded-dispatch behavior and global-target operation behavior are unknown. A nonzero transfer target is not validated here, and validity is not established merely by reading it.
- **documented_only — text and ownership:** Encoding, termination, storage size, allocation, release, and ownership remain unspecified.
- **documented_only — mutation and lifetime:** Callee effects, reentrancy, concurrent mutation, exceptional exits, and object validity across calls are not established.
- **Inferred intent:** The secondary operations resemble sizing and positioning an indicator; the search resembles selecting an active entry. Neither interpretation adds contractual semantics.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping. No runtime tests have been performed.

Established static evidence covers field widths and offsets, branches, visible read/write ordering, dependency arguments, signed and unsigned comparisons, return points, and the late read of the globally stored transfer target.

Callee implementations, ownership rules, application meanings, and target-level overflow behavior were not supplied. They remain separate from the established observations.
