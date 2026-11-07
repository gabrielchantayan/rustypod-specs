<!-- Trusted provenance: F-aea884b214ba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aea884b214ba

## Purpose

F-aea884b214ba performs a guarded operation using an object returned by a lookup dependency. It checks two classification results, requires a signed-positive return from F-10b760faca36, calls F-6bbddce1b930 with a freshly read subordinate field and literal zero, obtains a candidate, and conditionally invokes a six-argument operation.

The evidence does **not** establish that F-10b760faca36 returns a count or that the zero supplied to F-6bbddce1b930 denotes an index. Those interpretations are inferred intent only.

On normal completion, the routine returns one of three 32-bit values:

| Result | Established condition |
|---|---|
| 0 | The final operation was invoked and returned exactly 1. |
| 3 | The final operation was invoked and returned something other than 1. |
| 7 | A prerequisite check failed or candidate validation failed, so the final operation was not invoked. |

These descriptions do not establish application-level error names. In particular, result 3 is not the outcome for failed prerequisite checks.

The routine also calls designated dependencies for non-null lookup and candidate results before returning. Their placement resembles cleanup, but their ownership and resource-management effects are unknown.

## Interface

### Inputs and result

The routine accepts six 32-bit argument words, designated **A1** through **A6**.

| Argument | Established use |
|---|---|
| A1 | Initializes the first word of a temporary argument area. |
| A2 | Initializes the second word of that area and is separately passed to lookup. |
| A3 | Initializes the third word of that area and is passed to the final operation. |
| A4 | Initializes the fourth word of that area and is passed to the final operation. |
| A5 | Is passed to the final operation. |
| A6 | Is passed to the final operation. |

No semantic types, units, ranges, or ownership rules are established for these words. They may carry values interpreted by dependencies as handles, addresses, flags, or other data; the routine does not visibly convert their representations.

The result is a 32-bit word. The values 0, 3, and 7 describe normal-return paths only. The evidence does not establish a returned result for invalid memory access or a dependency that fails to return normally.

### Temporary argument area

Before the first dependency call, the routine initializes a local area represented as four consecutive 32-bit words:

| Byte offset | Initial value |
|---|---|
| +0 | A1 |
| +4 | A2 |
| +8 | A3 |
| +12 | A4 |

A pointer to the first word is passed to F-e2f82ac49d6f. The area belongs to the current invocation; no heap allocation for it is visible.

The permitted access extent, mutation rules, and retention rules of that dependency are unknown. The local area does not establish storage valid after this invocation returns.

The later final-operation arguments are the routine’s A3 and A4 arguments, not explicit reloads from this temporary area. The evidence therefore does not establish a copy-back mechanism from lookup’s argument area into those forwarded arguments.

### Accessed memory

Addresses and pointer-bearing fields are represented by 32-bit words. All offsets below are byte offsets.

| Descriptive object | Fields read |
|---|---|
| Lookup object | +16: classification input; +20: subordinate address |
| Subordinate object | +8: dependency input; +20: nested-object address |
| Nested object | +16: classification input |
| Candidate | +0: discriminator; +12: associated-object address or word |
| Candidate-associated object | +16: classification input |

These labels describe access relationships, not established application types. Full object sizes and alignment requirements are unknown.

For normal execution, every reached memory access must be valid. The routine explicitly checks the lookup result, one subordinate-address read, and the candidate result for zero. It does not explicitly check the nested-object address or candidate-associated address before dereferencing them.

## Behavior

### Initial preparation and lookup

The prospective result starts at 3, and the retained candidate address starts at zero. The routine initializes the four-word temporary area, then calls F-5fc3a5236553 without explicit arguments. Any return from that call is visibly unused.

It then calls F-e2f82ac49d6f with these arguments, in order:

1. Literal zero.
2. The address of the temporary area’s first word.
3. A2.

The returned word is treated as the lookup-object address and retained for the later lookup-cleanup call.

No behavior of the initial no-argument dependency is established beyond its call position. Neither it nor lookup may be assumed to preserve global or shared state.

### Lookup-object eligibility

The initial conditions are evaluated in short-circuit order:

1. If lookup returned zero, eligibility fails without reading lookup-object fields.
2. Otherwise, the routine reads lookup +16 and passes that word to F-2eb743f270d1. Eligibility fails unless the return is exactly 0x16, decimal 22.
3. If the first classification matches, the routine reads lookup +20 and checks that word for zero. Zero fails eligibility.
4. If that check passes, the routine reads lookup +20 again, follows subordinate +20, reads nested +16, and passes that word to F-2eb743f270d1. Eligibility fails unless this second return is exactly 0x15, decimal 21.

Any failed condition sets the prospective result to 7. A null lookup result skips lookup cleanup; a non-null result proceeds to that cleanup even when eligibility fails.

The subordinate address checked for zero is not a permanent validated snapshot of the object graph. Later accesses reread lookup +20. No synchronization or stability guarantee is visible.

There is no explicit zero check for subordinate +20 before the nested +16 access. A non-null subordinate does not by itself establish that this access is safe.

### Signed-positive gate and zero-valued argument

After both classifications pass, the routine follows the then-current lookup +20 address and reads subordinate +8. It passes that word to F-10b760faca36.

The return is tested as a signed 32-bit integer:

- Strictly positive: continue.
- Zero or negative: set the prospective result to 7 and skip both F-6bbddce1b930 and candidate acquisition.

Thus, a nonzero return is insufficient. For example, the 32-bit bit pattern 0x80000000 fails the signed-positive condition.

When the condition passes, the routine again follows lookup +20 and rereads subordinate +8. It calls F-6bbddce1b930 with:

1. This newly read word.
2. Literal zero.

The new word is not guaranteed to equal the value supplied to F-10b760faca36. That dependency may have changed the object graph or field, and the routine performs no equality check.

F-6bbddce1b930’s return value is not visibly inspected. Normal return from it leads directly to candidate acquisition.

This establishes a signed-positive gate followed by a call with a zero-valued second argument. It does not establish a count, an index, a successful selection, or any bound relationship between the two calls.

### Candidate acquisition and validation

The routine calls F-82d6fbbbf739 without explicit arguments and retains its returned candidate address.

Candidate validation is short-circuited in this order:

1. The candidate address must be nonzero.
2. The 32-bit word at candidate +0 must equal 6.
3. The routine reads candidate +12, follows that address, and reads associated-object +16.
4. It passes the last word to F-ae3bc1bfd2a8, whose return must equal 0x400, decimal 1024.

Failure of any condition sets the prospective result to 7 and skips the final operation.

A null candidate causes no candidate-field reads and no candidate-cleanup call. A non-null candidate remains subject to candidate cleanup even if its discriminator or classification fails.

There is no explicit zero check for candidate +12 before the associated-object +16 access.

### Final operation and result choice

After candidate validation succeeds, the routine calls F-7464b1422118 with six arguments, in order:

1. 0x40, decimal 64.
2. A3.
3. A4.
4. A5.
5. A6.
6. A newly read word from candidate +12.

The sixth argument is read after F-ae3bc1bfd2a8 returns. It need not equal the address used to obtain that dependency’s input. There is no second validation of a changed value before forwarding it.

If F-7464b1422118 returns exactly 1, the prospective result becomes 0. Every other returned value leaves it at 3. Other nonzero values are not normalized to success.

There is no visible retry or alternate final-operation path.

### Cleanup and return

On normal execution, every non-null lookup result is passed to F-92f52adf4077.

After that call returns, every acquired non-null candidate is passed to F-55d57b266fd4. Consequently, whenever both calls occur, lookup cleanup precedes candidate cleanup.

The routine retains the lookup and candidate addresses for these calls. It does not reacquire them from the object graph.

Neither cleanup return value is visibly inspected. The already selected result is returned after the applicable calls complete. If lookup cleanup does not return normally, the evidence does not guarantee that candidate cleanup occurs.

## Side effects

The routine directly initializes local storage, reads object fields, invokes dependencies, and selects its returned status. It shows no direct write into the lookup object, subordinate object, nested object, candidate, or candidate-associated object.

Dependency effects are not constrained by those observations:

- Lookup receives an address into local storage.
- Arguments forwarded to dependencies may themselves encode writable addresses.
- Dependencies may modify global, thread-local, or shared state.
- Object fields reread after dependency calls may have changed.
- No-argument calls are not established as pure.
- Cleanup dependencies may affect resources or other observable state.

No ownership transfer for A1–A6 is established. No explicit allocation, reference increment, rollback, or restoration of any dependency-managed state is visible.

The routine calls F-6bbddce1b930 before candidate acquisition, and lookup cleanup before candidate cleanup. These are established sequencing constraints even though the dependencies’ meanings remain unknown.

There is no visible mechanism guaranteeing cleanup following an invalid access or non-normal dependency exit. Status 7 must not be treated as a demonstrated catch-all for such failures.

## Dependencies

| Opaque ID | Established callsite contract |
|---|---|
| F-5fc3a5236553 | Called before lookup without explicit arguments; no returned value is visibly used. |
| F-e2f82ac49d6f | Receives zero, the temporary-area pointer, and A2; returns the lookup address. |
| F-2eb743f270d1 | Receives classification inputs; the first required return is 0x16 and the second is 0x15. |
| F-10b760faca36 | Receives subordinate +8; continuation requires a signed 32-bit return greater than zero. |
| F-6bbddce1b930 | Receives a freshly read subordinate +8 word and zero; its return is not visibly tested. |
| F-82d6fbbbf739 | Called without explicit arguments; returns the candidate address. |
| F-ae3bc1bfd2a8 | Receives candidate-associated +16; continuation requires exactly 0x400. |
| F-7464b1422118 | Receives the six final-operation arguments; exactly 1 produces status 0, otherwise status 3. |
| F-92f52adf4077 | Receives each non-null lookup result on normal completion. |
| F-55d57b266fd4 | Receives each acquired non-null candidate on normal completion, after lookup cleanup. |

These are callsite observations, not dependency implementations or ownership contracts.

## Acceptance examples

The following are conditional static acceptance criteria, not executed tests. Unless stated otherwise, all reached addresses are valid and every invoked dependency returns normally.

1. **Null lookup result.**  
   The routine returns 7. Neither classification call occurs. Neither cleanup call occurs. The initial dependency and lookup still occur.

2. **First classification returns 21 rather than 22.**  
   The routine returns 7. The subordinate-presence check and second classification are skipped. Lookup cleanup occurs.

3. **First classification passes, but lookup +20 is zero at its explicit check.**  
   The routine returns 7 without the second classification or signed-positive gate. Lookup cleanup occurs.

4. **Both classifications pass; F-10b760faca36 returns zero or a negative value.**  
   The routine returns 7. F-6bbddce1b930 and candidate acquisition are skipped. This includes a return bit pattern of 0xFFFFFFFF.

5. **F-10b760faca36 returns positive and changes subordinate +8.**  
   F-6bbddce1b930 receives the subsequently read value, not a guaranteed copy of the earlier input, followed by zero. This does not imply either value is a count or zero is an index.

6. **F-6bbddce1b930 returns normally with an otherwise arbitrary return value.**  
   Candidate acquisition follows. Its return value alone does not select status 7.

7. **Candidate acquisition returns zero.**  
   The routine returns 7 without reading candidate fields. Lookup cleanup occurs; candidate cleanup does not.

8. **A non-null candidate has discriminator 5.**  
   The routine returns 7 without candidate-associated classification or the final operation. Both cleanup calls occur, lookup first.

9. **Candidate discriminator is 6, but its classification returns 0x401.**  
   The routine returns 7. The final operation is skipped, but both cleanup calls occur.

10. **All gates pass and the final operation returns 1.**  
    The routine returns 0 after both cleanup calls complete.

11. **All gates pass and the final operation returns 0, 2, or a negative value.**  
    The routine returns 3, not 7, after both cleanup calls complete.

12. **Candidate +12 changes during candidate-associated classification.**  
    If classification returns 0x400 and execution remains valid, the final operation receives the newly read candidate +12 word. The routine does not reclassify that replacement.

13. **The subordinate is non-null, but its nested-object address is zero.**  
    The routine attempts the nested +16 access without an explicit null guard. No returned status or cleanup completion is established for an invalid access.

## Uncertainties

- **documented_only — Argument meanings:** Semantic types, units, valid ranges, and ownership of A1–A6 are unknown.
- **documented_only — Dependency meanings:** A signed-positive return does not establish a count. A literal-zero argument does not establish an index or selection operation.
- **documented_only — Object contracts:** Only accessed offsets are supported. Full layouts, alignment, lifetimes, and synchronization requirements are unknown.
- **documented_only — Temporary storage:** Lookup’s permitted reads, writes, and retention of the temporary-area pointer are unknown.
- **documented_only — Hidden state:** Dependencies may communicate through state not represented by explicit arguments.
- **documented_only — Cleanup semantics:** The designated calls may release resources, but deallocation, reference counting, and ownership transfer are not established.
- **documented_only — Abnormal execution:** Invalid accesses and non-normal dependency exits have no established result or cleanup guarantee.
- **Inferred intent:** The signed-positive gate, subsequent zero-valued argument, and candidate acquisition suggest an initial-item workflow. This interpretation is not required by the observed contract.
- **Inferred intent:** The three statuses likely distinguish success and different unsuccessful outcomes. Application-level names remain unknown.

## Evidence

Established behavior comes solely from static inspection of the supplied representation: short-circuit conditions, field offsets, repeated reads, argument order, literal comparisons, dependency sequencing, and return selection.

The representation supports 32-bit argument words, pointer-bearing words, and field accesses, together with a signed 32-bit positive comparison for F-10b760faca36’s return. That return is not visibly incremented, scaled, used as a bound, or passed as an index. Field addressing uses the stated fixed offsets; no additional bounds or overflow checks are shown.

No dependency bodies, external type definitions, caller contracts, or runtime observations were supplied. No runtime tests have been performed. The acceptance examples describe conditional consequences of the visible control flow, not observed executions.
