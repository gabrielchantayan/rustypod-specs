<!-- Trusted provenance: F-ed5938e2c6ce; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ed5938e2c6ce

## Purpose

F-ed5938e2c6ce conditionally processes a fixed table of 17 records for a selected signed 32-bit value. It visits records in ascending index order, dispatches eligible records to index-specific dependencies, and invokes a failure dependency if any dispatched operation returns a nonzero result.

Two gated record types may instead be disabled by clearing their leading byte. Processing then continues without dispatching that record.

The function returns a packed 64-bit result. Its lower and upper halves have distinct behavior; they should not be interpreted as a single conventional success or error code.

**Established static behavior:** table filtering, dispatch arguments, local state propagation, direct record disabling, and result packing are visible in the supplied source representation.

**Inferred intent:** the table appears to hold pending operations associated with a selector. Neither operation names nor the meaning of the selector are established.

## Interface

### Inputs

The function accepts four 32-bit inputs:

| Input | Interpretation established here |
|---|---|
| Context | Opaque 32-bit value forwarded unchanged to operation dependencies. Its pointer or handle semantics are unknown. |
| Selector | Signed 32-bit value. Controls whether scanning occurs and which records qualify. |
| Initial auxiliary value | Signed 32-bit value retained as mutable local state and potentially forwarded by index 14. |
| Initial carried value | Signed 32-bit value retained as mutable local state and potentially forwarded by index 14. |

The signed interpretation of the selector matters: the scan runs when the selector is **less than 3**, not merely when it is one of 0, 1, or 2. Negative selectors therefore enter the scan.

The context has no visible validation or interpretation within this function.

### Return value

The return is a 64-bit bit pattern composed of:

- **Upper 32 bits:** a mutable tracking value, initially equal to the selector.
- **Lower 32 bits:** initially 9 when scanning is bypassed, or 0 when scanning begins.

Consequently:

- For selectors greater than or equal to 3, normal return contains the selector’s 32-bit representation in the upper half and 9 in the lower half.
- For selectors less than 3, every normal return has a lower half of 0. Its upper half depends on the qualifying operations executed.

A negative upper-half value retains its 32-bit representation. No signed 64-bit numerical interpretation is established.

### Table layout and preconditions

The scan uses a shared table base and exactly 17 records, numbered 0 through 16 inclusive. The stride is 16 bytes.

| Byte offset within a record | Visible use |
|---|---|
| 0 | One-byte enabled marker. Zero means disabled; any nonzero byte permits processing if the selector matches. |
| 1–3 | No direct access shown. |
| 4–7 | Signed 32-bit selector field. |
| 8–11 | First 32-bit payload field. Interpretation depends on the record index. |
| 12–15 | Second 32-bit payload field. Used by indices 4, 10, and 16. |

The table must support the reads required by the selected paths. Records 4 and 16 must additionally support a one-byte write at offset 0 when their availability check returns zero.

No bounds, null-pointer, alignment, object-validity, or synchronization checks are visible. Calls through the availability object also require a valid callable target. Behavior outside these preconditions is not specified by this contract.

## Behavior

### Entry and scan selection

The tracking value starts as the selector.

If the selector is at least 3, the function returns immediately with lower half 9. No table access or dependency call is shown on that path.

Otherwise, the lower result state starts at 0, the shared table base is obtained, and the function visits all 17 record positions in ascending order unless an invoked dependency prevents continuation.

The table cursor advances by 16 bytes after each completed or skipped iteration. The scan does not visibly reload the shared table base on each iteration.

### Eligibility and observable read ordering

For each record:

1. The signed 32-bit selector field at offset 4 is read and compared with the input selector.
2. Only if those values match is the enabled byte at offset 0 read.
3. Dispatch occurs only if the enabled byte is nonzero.

A selector mismatch therefore does not require an enabled-byte read. An ineligible record does not directly change any local carried state and does not trigger an operation dependency.

Eligibility is evaluated when each record is reached. There is no visible snapshot of all record contents before scanning. Dependencies may therefore affect later eligibility or payload reads if they modify shared storage.

The chosen operation is determined by the **record index**, not by a separate opcode field in the record.

### State carried between records

Three local values influence later behavior:

- **Tracking value:** initially the selector. Index 4 assigns it the selector after a successful availability check. Indices 10 and 16 assign it zero on their dispatch paths.
- **Auxiliary value:** initially the third input. Indices 10 and 16 assign it the selector on their dispatch paths.
- **Carried value:** initially the fourth input. Each payload-specific operation assigns its return value to this state.

The generic operation used by several indices does not replace the carried value.

All payload-specific operation results are immediately checked. If one returns nonzero, the failure path is taken before the next record. Thus, after any such operation that permits normal continuation, the carried value is zero.

Index 14 receives the current values of all three states. Their values are not necessarily the original inputs.

### Dispatch contract

“Low byte” below means the least-significant eight bits of the stored 32-bit value, forwarded as a value from 0 through 255. It does not mean that only one byte is read from the record.

Argument order is significant. The listing describes argument values, not an otherwise unspecified order of evaluating separate payload reads.

| Record index | Dispatch behavior |
|---|---|
| 0, 1, 2, 3, 5, 9, 12, 15 | Call F-0c4d2e445922 with the context and the record index. The index undergoes signed-byte conversion before integer use; every reachable index remains its ordinary nonnegative value. Store the returned 32-bit value in the lower-result state. |
| 4 | Perform the availability check described below. If available, set the tracking value to the selector and call F-a6d44bb3a0a1 with context, low byte of the first payload, low byte of the second payload, and zero. Replace the carried value with the result. |
| 6 | Call F-3867c434fcca with context, the full first 32-bit payload, zero, and selector. Replace the carried value with the result. |
| 7 | Call F-0b77559fa811 with context, low byte of the first payload, zero, and selector. Replace the carried value with the result. |
| 8 | Call F-adf871004f14 with context, low byte of the first payload, zero, and selector. Replace the carried value with the result. |
| 10 | Perform the conversion-and-dispatch sequence described below. |
| 11 | Call F-adb56aabcc86 with context, low byte of the first payload, zero, and selector. Replace the carried value with the result. |
| 13 | Call F-d9032d50340b with context, a Boolean indicating whether the full first 32-bit payload is nonzero, zero, and selector. Replace the carried value with the result. |
| 14 | Call F-280d7b6ddabc with context, low byte of the first payload, zero, selector, current tracking value, current auxiliary value, and current carried value. Replace the carried value with the result. |
| 16 | Perform the availability check described below. If available, set the tracking value to zero and the auxiliary value to the selector. Call F-6ae62d4262ef with context, low byte of the first payload, zero, low byte of the second payload, zero, and selector. Replace the carried value with the result. |

For index 13, Boolean conversion uses all 32 payload bits. A payload whose low byte is zero can still produce Boolean true.

### Availability checks at indices 4 and 16

Each eligible gated record independently:

- Calls F-5fd97c30efdc to obtain an object reference.
- Obtains a dispatch-table reference from the beginning of that object.
- Invokes the callable entry at byte offset 20 within that dispatch table.

No explicit arguments are shown for the indirect call. Whether the calling convention conveys an implicit receiver is not established.

The indirect result is tested as a 32-bit value:

- **Zero:** write zero to the current record’s enabled byte, skip its operation, and continue scanning.
- **Nonzero:** proceed with the corresponding payload-specific operation.

The gate result is not itself treated as an operation error. In particular, a nonzero gate result permits dispatch rather than invoking the failure dependency.

The function does not recheck the record’s selector or enabled marker after the gate call. On the available path, the payload reads occur after that call.

### Index 10 conversion and dispatch

For an eligible index-10 record:

- The full first 32-bit payload is captured before invoking the conversion dependency.
- F-d7be9cfb4000 is called with the full second 32-bit payload and the constant 60.
- Its 64-bit return is interpreted as two 32-bit halves.
- The tracking value is set to zero.
- The auxiliary value is set to the selector.
- F-55290c3145ab is called with:
  - context;
  - low byte of the previously captured first payload;
  - low byte of the conversion result’s lower 32-bit half;
  - the conversion result’s complete upper 32-bit half, interpreted as a signed 32-bit value.
- The operation result replaces the carried value.

There is no visible error check on the conversion dependency’s return. Its arithmetic meaning is unknown; the constant 60 does not establish division, time conversion, or any particular quotient/remainder convention.

The first payload capture is significant: modifications to that field during the conversion call do not change the value forwarded to F-55290c3145ab.

### Operation failures and completion

After every dispatched operation, its 32-bit result is checked for exact equality with zero.

- Zero allows scanning to continue.
- Any nonzero bit pattern causes a call to F-a817d8d58c10.

The supplied representation identifies F-a817d8d58c10 as non-returning. Under that property, an operation failure prevents later records and prevents a normal return from this function. Whether the dependency terminates execution, raises an exception, or transfers control another way is unknown.

For scanning paths that finish normally, the lower result half is always zero: it starts at zero, and any generic-operation result that would make it nonzero instead invokes the non-returning failure path.

## Side effects

The only explicit shared-memory write performed directly by this function is clearing the enabled byte of an eligible index-4 or index-16 record after its availability check returns zero.

No direct successful-operation path clears the record’s enabled marker. Repeated invocations may therefore dispatch the same record again unless dependencies or other actors alter shared state.

Other direct effects are dependency calls and local state changes. The local updates to the auxiliary and carried values do not, by themselves, write back through the input arguments.

Dependency side effects are unspecified. In particular, this contract does not assume that any dependency:

- Preserves table contents or other globals.
- Preserves the shared table-base value.
- Leaves the availability object valid after returning.
- Allocates, owns, retains, or releases memory.
- Preserves records that have not yet been visited.

The function shows no explicit allocation, deallocation, ownership transfer, or synchronization operation. These absences do not constrain dependency behavior.

## Dependencies

| Opaque ID | Role visible to this function |
|---|---|
| F-0c4d2e445922 | Generic operation for eight record indices; returns a checked 32-bit result. |
| F-5fd97c30efdc | Supplies an object used for an indirect availability call. |
| F-a6d44bb3a0a1 | Index-4 operation. |
| F-3867c434fcca | Index-6 operation. |
| F-0b77559fa811 | Index-7 operation. |
| F-adf871004f14 | Index-8 operation. |
| F-d7be9cfb4000 | Converts two 32-bit inputs into a 64-bit result used by index 10. |
| F-55290c3145ab | Index-10 operation. |
| F-adb56aabcc86 | Index-11 operation. |
| F-d9032d50340b | Index-13 Boolean-payload operation. |
| F-280d7b6ddabc | Index-14 operation receiving accumulated local state. |
| F-6ae62d4262ef | Index-16 operation. |
| F-a817d8d58c10 | Failure dependency identified as non-returning. |

The indirect availability callable has no supplied opaque function ID. Its target identity and implementation are unknown.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless a mutation is explicitly part of an example, assume dependencies do not alter table data relevant to that example.

1. **Scan bypass.** With selector 3, the function performs no table accesses or dependency calls and returns upper half 3, lower half 9. The same bypass applies to any larger signed 32-bit selector.

2. **Negative selector.** With selector −1 and no eligible records, the scan visits all 17 positions and returns upper half `0xFFFFFFFF`, lower half zero. A negative selector is not rejected.

3. **Eligibility short-circuit.** If a record’s selector differs from the input selector, its enabled byte need not be read. If its selector matches but its enabled byte is zero, no operation is dispatched.

4. **Byte truncation.** For an eligible index-7 record with first payload `0x123456AB`, F-0b77559fa811 receives 171 as its second argument, followed by zero and the selector.

5. **Full-width Boolean conversion.** For an eligible index-13 record with first payload `0x00000100`, F-d9032d50340b receives Boolean true, despite the low byte being zero.

6. **Unavailable gated record.** If index 4 is eligible and its indirect availability result is zero, its enabled byte is cleared. F-a6d44bb3a0a1 is not called, the carried local state is unchanged by this branch, and scanning continues.

7. **Conversion result partitioning.** Suppose eligible index 10 has first payload `0x123456AB`, and F-d7be9cfb4000 returns upper half `0x80000001` and lower half `0x123456CD`. F-55290c3145ab receives context, 171, 205, and the signed 32-bit interpretation of `0x80000001`. If it succeeds, the tracking value is zero and the auxiliary value is the selector.

8. **Initial state reaching index 14.** If index 14 is eligible and no earlier payload-specific operation dispatches, it receives the original selector as tracking value, the third input as auxiliary value, and the fourth input as carried value. Earlier successful generic operations do not replace those states.

9. **Updated state reaching index 14.** If index 10 dispatches successfully before an eligible index 14, index 14 receives tracking value zero, auxiliary value equal to the selector, and carried value zero.

10. **Failure stops the scan.** If an eligible index-6 operation returns any nonzero value, F-a817d8d58c10 is invoked before index 7 is examined. Assuming that dependency is non-returning, there is no packed return value.

11. **Final gate preserves or replaces tracking state.** If index 16 is eligible but unavailable, it is disabled without changing the tracking value. If available and its operation succeeds, normal return has upper half zero and lower half zero.

## Uncertainties

The following are **documented_only**:

- Domain meanings of selectors, record indices, payloads, and packed return halves.
- Actual table allocation, initialization, lifetime, and synchronization rules.
- Dependency implementations, error-code meanings, and side effects.
- Ownership and lifetime of the availability object.
- The indirect availability call’s implicit arguments and calling convention.
- The conversion dependency’s arithmetic semantics.
- Relative evaluation order of distinct payload reads passed to the same call, where the representation does not explicitly sequence them.
- Behavior if the dependency identified as non-returning unexpectedly returns.
- Safety under concurrent mutation, reentrancy, or dependencies invalidating referenced storage.

The apparent pending-operation interpretation is inferred intent only. It is not a requirement on dependency behavior.

## Evidence

This contract is based solely on the supplied static source representation. No runtime tests, dependency inspections, or independent implementation validation have been performed.

Established evidence includes the signed scan threshold, 17-record bound, 16-byte stride, field widths and offsets, short-circuited eligibility checks, per-index dispatch arguments, state propagation, two direct enabled-byte writes, checked operation results, and 64-bit result composition.

The non-returning failure behavior relies on the supplied static annotation. Unknown dependency behavior has not been assumed to preserve globals, maintain object lifetimes, or implement any particular ownership policy.
