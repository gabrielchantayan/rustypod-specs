<!-- Trusted provenance: F-a2c46f03f6cf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a2c46f03f6cf

## Purpose

F-a2c46f03f6cf selects between two construction paths using the zero/nonzero result of F-3ec7d07fef90. That dependency receives an address two bytes into the input and a fixed comparison-token address.

- **Zero result:** requests 144 bytes, initializes selected fields, calls F-41bb5880f4ae, and returns the allocation address.
- **Nonzero result:** requests 164 bytes, initializes a header, passes an interior address to F-576ea5ba1d23, initializes fields relative to that dependency’s returned address, invokes two further dependencies, and returns an address 88 bytes before that returned address.

These operations are established by the supplied static representation. Their interpretation as particular object types, formats, or ownership transfers is not established.

## Interface

### Input

The function accepts one address presented as a pointer to 16-bit units.

Two input positions are directly relevant:

| Position | Established use |
|---|---|
| Input+0 | A 16-bit value read directly on the nonzero branch, after that branch’s initial field writes. |
| Input+2 | Passed to F-3ec7d07fef90 and, on the nonzero branch, F-151ab4bfe553. |

The two-byte displacement results from advancing the input pointer by one 16-bit element. It is not a one-byte displacement.

The dependencies’ input-access requirements are unavailable. Consequently, the complete required input extent, encoding, termination, and interpretation are unknown. Passing Input+2 does not itself establish string semantics.

### Output

The return value is presented as a pointer to 32-bit units. Its established meaning is an address:

- On the zero branch, the allocation result.
- On the nonzero branch, the result of F-576ea5ba1d23 minus 88 bytes.

There is no explicit status result, error code, or checked failure return.

### Address and value notation

All offsets in this contract are byte offsets.

- **A:** the allocation result for the selected branch.
- **Q:** the address returned by F-576ea5ba1d23 on the nonzero branch.
- **B:** Q−88.
- **G1:** the first fixed 32-bit global value read on the zero branch.
- **G2:** the second fixed 32-bit global value read on the zero branch.
- **G3:** the fixed 32-bit global value read on the nonzero branch.

These labels identify addresses and observed values for explanation; they do not introduce stored fields or additional operations.

The visible accesses have widths of 8, 16, or 32 bits. The representation uses a 32-bit integer form for some address calculations. Exact pointer representation, address-overflow behavior, alignment requirements, byte order, and invalid-access behavior are not independently established.

The 16-bit all-ones value is 65,535. The 32-bit all-ones value is 4,294,967,295 as an unsigned bit-pattern interpretation; its signed interpretation is not needed to specify the writes.

### Preconditions for normal completion

Normal completion requires:

- All invoked dependencies to return normally.
- The input and other accessed addresses to support the executed reads and writes.
- Allocation results to be usable by subsequent operations.
- On the nonzero branch, Q-relative addresses to be valid, including addresses preceding Q.
- Address calculations to be valid under the execution environment’s rules.

The function does not check these conditions. Neither allocation result nor Q is checked before use.

## Behavior

### Branch selection

The first dependency invocation is F-3ec7d07fef90 with:

1. Input+2.
2. A fixed comparison-token address.

Only whether the result equals zero affects control flow. Zero selects the 144-byte path; every nonzero result selects the 164-byte path.

No direct read of Input+0 precedes this call. The dependency may access or modify memory, but its internal behavior is unavailable.

The exact comparison-token bytes and the dependency’s comparison rules are not established. Apparent naming in the supplied material is not proof of token content or semantics.

### Zero-result branch: 144-byte construction

F-8157a180d901 is called with a size argument of 144 bytes. Its result becomes A.

After the allocation call returns, the function reads G1 and retains that value. It then reads G2 and writes G2 at A+0. The retained G1 value supplies both a later field write and an argument to F-41bb5880f4ae; those uses do not involve separate global reads.

The following direct writes occur in order:

| Order | Address | Width | Value |
|---:|---|---:|---|
| 1 | A+0 | 32 bits | G2 |
| 2 | A+92 | 32 bits | 0 |
| 3 | A+88 | 32 bits | Captured G1 |
| 4 | A+96 | 16 bits | All bits set |
| 5 | A+98 | 8 bits | 0 |
| 6 | A+100 | 32 bits | All bits set |
| 7 | A+108 | 8 bits | 0 |
| 8 | A+112 | 32 bits | 0 |
| 9 | A+116 | 32 bits | 0 |
| 10 | A+120 | 32 bits | 0 |
| 11 | A+124 | 32 bits | 0 |
| 12 | A+128 | 32 bits | 0 |
| 13 | A+132 | 32 bits | 0 |
| 14 | A+136 | 32 bits | 0 |
| 15 | A+140 | 32 bits | 0 |

The final write occupies A+140 through A+143, ending exactly at the requested 144-byte boundary.

This is selective initialization, not a full-region clear. No direct writes in this branch initialize:

- A+4 through A+87.
- A+99.
- A+104 through A+107.
- A+109 through A+111.

Their contents may depend on the allocator or other dependencies.

After these writes, the function invokes F-41bb5880f4ae. The supplied representation shows six argument expressions:

1. A.
2. The original input address.
3. A 32-bit all-ones value.
4. Captured G1.
5. An unresolved incoming machine-state value.
6. Another unresolved incoming machine-state value.

The final two expressions have no established semantic role. Their reliable calling-convention interpretation is also unresolved.

The dependency’s return value is unused. If it returns normally, the function returns A without further direct writes or a visible cleanup operation.

The table specifies writes before F-41bb5880f4ae, not guaranteed field contents at function return. That dependency may change the initialized memory or its validity.

### Nonzero-result branch: 164-byte construction

F-8157a180d901 is called with a size argument of 164 bytes. Its result becomes A.

The function then:

1. Reads G3.
2. Writes G3 at A+0.
3. Calls F-576ea5ba1d23 with A+88.
4. Uses that dependency’s result as Q.

The supplied representation does not establish that Q equals A+88. Subsequent field addresses and the returned address derive from Q.

The following direct writes occur in order after F-576ea5ba1d23 returns:

| Order | Address | Width | Value |
|---:|---|---:|---|
| 1 | Q+8 | 32 bits | 0 |
| 2 | Q+16 | 32 bits | 0 |
| 3 | Q+20 | 16 bits | 0 |
| 4 | Q+22 | 16 bits | 0 |
| 5 | Q+24 | 32 bits | 0 |
| 6 | Q+28 | 32 bits | 0 |
| 7 | Q+49 | 8 bits | 0 |
| 8 | Q+52 | 32 bits | 0 |
| 9 | Q+56 | 8 bits | 0 |
| 10 | Q+57 | 8 bits | 0 |
| 11 | Q+60 | 32 bits | 0 |
| 12 | Q+64 | 32 bits | 0 |
| 13 | Q+68 | 32 bits | 0 |
| 14 | Q+72 | 32 bits | 0 |
| 15 | Q−84 | 16 bits | Value read from Input+0 at this stage |
| 16 | Q−82 | 8 bits | 0 |

The direct 16-bit input read occurs after the first fourteen writes. It is not an entry-time snapshot. If an earlier dependency or an overlapping direct write changes Input+0, the copy uses the value observed when this read occurs, subject to the execution environment’s memory semantics.

The last direct **32-bit zero write** occupies Q+72 through Q+75. It is followed by the 16-bit input copy and then the final direct zero write: an 8-bit clear at Q−82.

Next, the function calls:

1. F-151ab4bfe553 with Q and Input+2.
2. F-933d555a9096 with B and zero, where B=Q−88.

Both return values are unused. If both dependencies return normally, the function returns B.

The zero argument to F-933d555a9096 has no established meaning as a flag, mode, or ownership request.

### Conditional relationship between A and Q

If F-576ea5ba1d23 returns exactly the address it receives, then Q=A+88 and B=A. Under that condition:

- The 16-bit input copy targets A+4.
- The final direct byte clear targets A+6.
- The Q-relative positive-offset writes occupy selected fields between A+96 and A+163.
- The last direct 32-bit zero write occupies A+160 through A+163.
- The function returns A.

The write at A+160 is not the final direct zero write overall. The later byte clear at A+6 follows the input copy.

These relationships are conditional. They do not establish that F-576ea5ba1d23 normally or necessarily returns its argument.

## Side effects

### Direct effects and ordering

The function directly:

- Calls the selector before either allocation.
- Requests one allocation on the selected path.
- Reads the selected branch’s fixed global values after allocation.
- Writes selected fields in the stated order and widths.
- Reads Input+0 only on the nonzero path, at the specified late stage.
- Calls further dependencies after direct initialization.
- Returns an address without an explicit local release operation.

No direct write specifically targets the input as an input parameter. This does not guarantee input immutability: dependencies receive input-related addresses, and aliasing may cause field writes to overlap input memory.

Global reads are not entry-time snapshots. Changes made by the selector or allocator may affect the values subsequently read.

### Dependency effects and final contents

The visible caller does not bound dependency effects. This contract does not assume that any dependency:

- Preserves global values.
- Preserves input contents.
- Preserves previously written fields.
- Retains the allocation or leaves it valid.
- Acquires or releases no resources.
- Returns the address it receives.

The direct-write tables describe observable operations in the supplied representation. They are not unconditional final-memory snapshots.

### Ownership and failures

Allocation and initialization suggest construction intent. They do not establish who owns the resulting memory, whether ownership transfers to a dependency or caller, or how the memory must be released.

There is no visible local handling for:

- Allocation failure.
- An unusable Q result.
- Invalid input or output memory.
- Dependency failure or abnormal exit.
- Partially completed initialization.

An abnormal exit or invalid access may interrupt execution after earlier side effects have occurred. No rollback, recoverable failure result, or exception-safety guarantee is established.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-3ec7d07fef90 | Receives Input+2 and a fixed token address; zero/nonzero result selects the branch. |
| F-8157a180d901 | Receives 144 or 164 as a size argument; returned address is used without a local validity check. |
| F-41bb5880f4ae | Runs after zero-branch initialization; receives A, the original input, an all-ones value, captured G1, and two unresolved machine-state expressions. |
| F-576ea5ba1d23 | Receives A+88; returned address determines subsequent nonzero-branch field locations and return-address calculation. |
| F-151ab4bfe553 | Receives Q and Input+2 after the nonzero-branch direct writes. |
| F-933d555a9096 | Receives Q−88 and zero after F-151ab4bfe553 returns. |

No direct recursive invocation of F-a2c46f03f6cf is visible. Indirect invocation through dependencies is not established or excluded.

## Acceptance examples

These are conditional acceptance criteria from static evidence, not executed tests.

### Zero selector result

Given a zero result from F-3ec7d07fef90 and normal completion:

- The allocation request is 144 bytes.
- The zero-branch writes occur in their listed order before F-41bb5880f4ae.
- The value written at A+88 and the fourth represented argument to F-41bb5880f4ae come from the same captured G1 read.
- F-576ea5ba1d23, F-151ab4bfe553, and F-933d555a9096 are not directly called on this path.
- The returned address is A, irrespective of the unused return value from F-41bb5880f4ae.

If F-41bb5880f4ae preserves the initialized fields and their validity, their listed values remain observable at return. Without that condition, final contents are unknown.

### Any nonzero selector result

A result represented as one and a result represented as negative one both select the 164-byte path. The branch distinction is solely zero versus nonzero.

Given normal completion, the return address is Q−88.

### Helper returns its input address

Given Q=A+88 and valid accesses:

- The function copies the 16-bit value observed at Input+0 to A+4.
- Its last direct **32-bit zero write** occupies A+160 through A+163, reaching but not exceeding the requested allocation boundary.
- That 32-bit write precedes the input copy.
- After the input copy, the function performs its final direct zero write: an 8-bit clear at A+6.
- The function eventually returns A if the remaining dependencies return normally.

Final contents still depend on the subsequent dependencies.

### Helper returns a different valid address

Given Q different from A+88, with every required Q-relative access valid:

- Subsequent direct field writes use Q-relative addresses.
- F-151ab4bfe553 receives Q.
- F-933d555a9096 receives Q−88.
- The function returns Q−88, not necessarily A.

Unconditionally replacing that return address with A would not match the established behavior.

### Input changes before the copy

If an earlier dependency changes Input+0 and no intervening operation replaces that value, the nonzero branch copies the value observed at its late 16-bit read, not necessarily the value present on entry.

An entry-time input snapshot is therefore not equivalent in general.

### Allocation failure or abnormal dependency exit

If the allocation dependency returns null, the function has no visible early null-return check. Successful completion is not established.

If a later dependency exits abnormally, this contract does not promise that the function returns or that earlier writes and resource effects are undone.

## Uncertainties

### Inferred intent

The allocation requests, initial header writes, and subsequent dependency calls suggest two related construction paths. Interpreting the header values as type descriptors, dispatch metadata, or any particular object model remains inference.

### Documented-only unknowns

The following remain **documented_only**:

- Exact token bytes and comparison semantics.
- Dependency-internal reads, writes, errors, and resource operations.
- Numeric values and semantic roles of G1, G2, and G3.
- Whether Q ordinarily equals A+88.
- The reliable signature interpretation and values of the final two represented arguments to F-41bb5880f4ae.
- Ownership, lifetime, destruction, and release requirements.
- Required input extent, encoding, and termination.
- Contents of bytes not directly initialized.
- Final contents and validity after dependencies execute.
- Machine byte order, alignment rules, address-overflow behavior, and invalid-access behavior.
- Any externally guaranteed non-aliasing relationships.

Conventional allocator, comparison, or constructor behavior must not be used to fill these gaps.

## Evidence

Established behavior comes from the supplied static function representation: branch conditions, dependency-call order, allocation-size arguments, explicit memory-access widths, address displacements, constants, global-read placement, and returned-address expressions.

The nonzero path places the 32-bit zero write at Q+72 before the 16-bit input copy at Q−84 and the final byte clear at Q−82. Under Q=A+88, those locations become A+160, A+4, and A+6 respectively. This ordering distinguishes the last direct 32-bit zero write from the final direct zero write overall.

Dependency implementations, global definitions, token contents, and runtime observations are unavailable. Names and the prior draft are not independent proof of semantics.

No runtime tests have been performed. The acceptance examples describe conditional static requirements, not observed test results.
