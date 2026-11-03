<!-- Trusted provenance: F-e3dbfac98606; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e3dbfac98606

## Purpose

F-e3dbfac98606 performs a handle-based operation with conditional processing, mandatory finalization on the successful-lookup path, and a final status-reporting call.

Established behavior has three outcomes:

- Lookup yields a null pointer: report status **9** and return **0xffffffff**.
- Lookup succeeds and conditional processing is either unnecessary or returns a nonzero value: report status **0** and return **0**.
- Lookup succeeds, conditional processing is required, and that processing returns zero: report status **28** and return **0xffffffff**.

These outcomes describe ordinary completion: each invoked dependency must return normally, and every direct memory access must be valid. The supplied evidence does not establish exception handling, recovery from invalid memory, or behavior when a dependency does not return.

The paired calls surrounding record processing may represent synchronization or resource management. That interpretation is inferred, not established. The contract below specifies their arguments and ordering without assigning them an unproven ownership or locking meaning.

## Interface

### Input

The function receives one opaque **32-bit value**. Its interpretation is delegated to dependencies.

The input is passed:

1. To F-667717e3fb3e as its first argument, with a second argument of zero.
2. To F-b3f16c6113a2 later, but only if lookup returned a non-null pointer.

No direct validation, masking, or transformation of the input is visible. In particular, the supplied evidence does not establish that an input value of zero is invalid. A null lookup result, rather than a particular input value, selects the lookup-failure path.

### Output

The return value is a **32-bit bit pattern**:

| Condition | Return value |
|---|---:|
| Null lookup result | 0xffffffff |
| Non-null lookup result; processing skipped | 0 |
| Non-null lookup result; processing returns nonzero | 0 |
| Non-null lookup result; processing returns zero | 0xffffffff |

Interpreted as a signed 32-bit integer, 0xffffffff is −1. The established interface is the bit pattern; signed interpretation is not required.

Before returning normally, the function calls F-038b46411c0c with a status value of **9**, **0**, or **28**, as determined below. The evidence does not identify this dependency’s destination or prove that it sets a particular global error facility.

### Record layout accessed directly

A non-null lookup result is treated as a record with the following accessible data:

| Location | Access width | Use |
|---|---:|---|
| Record byte offset 0 | 32 bits | Address of an intermediate object |
| Intermediate object byte offset 0 | 32 bits | Base address of another object |
| That object byte offset 120, or 0x78 | 16 bits | Token passed to two dependencies |
| Record byte offset 4 | 16 bits | Flags controlling conditional processing |

The supplied operations model the address-bearing fields as 32-bit quantities. The token occupies two bytes. Only bits 0 and 1 of the flags affect this function’s branch decision.

No record length, allocation layout beyond these accesses, alignment guarantee, or byte order is independently established.

## Behavior

### 1. Lookup

The first dependency invoked is F-667717e3fb3e. It receives the original 32-bit input and a second argument equal to zero.

Its returned pointer determines the initial branch.

If the pointer is null:

- No record fields are read.
- No token-related dependency is called.
- No conditional processing or record finalization is performed.
- F-b3f16c6113a2 is not called.
- F-038b46411c0c receives status 9.
- The function returns 0xffffffff if that final dependency returns normally.

A null lookup result is the only directly checked pointer failure. There are no visible checks for null or invalid addresses within a non-null record’s nested address chain.

### 2. Read the token before invoking record processing

For a non-null result, the function follows the two address-bearing fields described in the interface and reads the 16-bit token at byte offset 120 of the resulting object.

These accesses have a required data-dependent order:

1. Read the record’s first 32-bit field.
2. Read a 32-bit address from the location identified by that field.
3. Read the 16-bit token at the resulting base address plus 120.

The token is saved for subsequent calls. There is no visible reread of its memory location later in this function.

The function then calls F-6443787f9616 with that saved token. Its return value is not used in any visible decision.

The flags are read only **after** this dependency returns. Consequently, the relevant flags are those observed at that later read, not necessarily their contents when lookup returned.

### 3. Decide whether conditional processing is required

The function reads the 16-bit flags field at record byte offset 4 and considers its two least significant bits.

- If both low bits are zero, F-a97e4a8e33df is not called.
- If either low bit is one, F-a97e4a8e33df is called once with the record pointer.

Bits 2 through 15 do not affect this decision. For example, a flags value of 0x8000 does not request processing, whereas 0x8001 does.

When processing is skipped, the selected outcome is status 0 and return value 0.

When processing is called, its result is compared with zero as a 32-bit integer:

- A result equal to zero selects status 28 and return value 0xffffffff.
- Any nonzero result selects status 0 and return value 0.

The test does not distinguish positive from negative nonzero values. There is no visible retry and no second call to the conditional processor.

This inverted-looking relationship—zero selecting failure and nonzero selecting success—is established by the supplied comparison. Conventional meanings of a dependency’s return value must not replace it.

### 4. Finalize every non-null-lookup path

After the conditional decision, the function performs the following calls in this order:

1. F-f323001005c6 with the record pointer.
2. F-b3f16c6113a2 with the original input.
3. F-591e5dd605b9 with the saved 16-bit token.

This sequence occurs whether conditional processing was skipped, returned zero, or returned nonzero, provided preceding operations return normally.

The record is not directly read again after F-f323001005c6 is called. The token passed to F-591e5dd605b9 is the value saved before F-6443787f9616, not a value reread after processing or finalization.

The return values of these three dependencies do not participate in the visible outcome selection. There is no visible branch that converts their returned values into a different status or function result.

### 5. Report status and return

After either the null-lookup path or the complete non-null-path finalization sequence, F-038b46411c0c receives the selected status.

On the non-null path, this reporting call occurs after F-591e5dd605b9 returns. It is not performed between conditional processing and finalization.

If the reporting dependency returns normally, the function returns the previously selected 32-bit result. Its outcome selection does not inspect the reporting dependency’s return value.

## Side effects

### Direct memory effects

The supplied body performs direct reads of the successful lookup record, its nested address chain, the token, and the flags.

No direct write to the record, token storage, flags, or other externally supplied memory is visible.

That does **not** imply that the operation is read-only. Every dependency may have side effects not described by its use here, including effects on shared state, the record, or memory ownership.

### Ordering constraints

For a non-null lookup result, the observable call order is:

1. F-667717e3fb3e.
2. Token-chain reads.
3. F-6443787f9616.
4. Flags read.
5. F-a97e4a8e33df, only when a low flag bit is set.
6. F-f323001005c6.
7. F-b3f16c6113a2.
8. F-591e5dd605b9.
9. F-038b46411c0c.

The function does not visibly establish synchronization before reading the token chain. Any claim that those initial reads are protected requires information outside this body.

### Ownership and lifetime

No allocation or explicit deallocation primitive is directly identified.

F-f323001005c6 and F-b3f16c6113a2 may finalize resources, but their ownership semantics are unknown. Their names here must not be treated as promises to retain, release, close, or free anything.

For ordinary defined execution:

- The successful lookup record and nested address chain must support the initial token reads.
- The record must remain readable at offset 4 after F-6443787f9616 returns.
- Each dependency must receive an argument that is valid under its own contract.

The evidence does not establish who must provide these lifetime guarantees or whether the dependencies enforce them.

The saved token permits its later use without another direct access to the token-bearing object. This does not, by itself, establish that releasing that object earlier is permitted.

## Dependencies

| Opaque function ID | Arguments supplied here | Established role in this function |
|---|---|---|
| F-667717e3fb3e | Original 32-bit input; zero | Returns the pointer whose nullness selects the initial branch |
| F-6443787f9616 | Saved 16-bit token | Called before reading the flags; returned value is not used |
| F-a97e4a8e33df | Lookup record pointer | Conditionally called; zero versus nonzero result selects failure versus success |
| F-f323001005c6 | Lookup record pointer | Called after the processing decision on every non-null path |
| F-b3f16c6113a2 | Original 32-bit input | Called after F-f323001005c6 on every non-null path |
| F-591e5dd605b9 | Saved 16-bit token | Called after F-b3f16c6113a2 on every non-null path |
| F-038b46411c0c | Selected status: 0, 9, or 28 | Final dependency call before normal return |

The two token-taking dependencies receive the same saved 16-bit value. Their complete parameter types and any argument-extension requirements are not established.

No dependency is assumed to preserve globals, retain memory, release memory, or return normally merely because this function ignores its result.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Each assumes valid required memory and normal return from all invoked dependencies.

### Null lookup

Given that F-667717e3fb3e returns null:

- The only subsequent dependency call is F-038b46411c0c with 9.
- No token or flags reads occur.
- The returned bit pattern is 0xffffffff.

### No low flags set

Given a non-null lookup result and a flags read yielding 0x0000:

- The token is read and passed to F-6443787f9616 before the flags read.
- F-a97e4a8e33df is skipped.
- The three non-null-path finalization calls occur in their specified order.
- F-038b46411c0c receives 0.
- The function returns 0.

The same branch behavior applies to a flags value of 0xfffc: high flag bits alone do not request processing.

### Either low flag bit requests processing

Given a non-null lookup result and a flags value of 0x0001, 0x0002, or 0x0003:

- F-a97e4a8e33df is called exactly once with the record pointer.
- No distinction among these three values is made by this function.

If that dependency returns zero, finalization still occurs, status 28 is reported, and the function returns 0xffffffff.

If it returns any nonzero 32-bit value, including 0xffffffff, finalization still occurs, status 0 is reported, and the function returns 0.

### Flags change before their read

Given that the low flag bits were initially zero, but F-6443787f9616 changes the record so that bit 1 is set before returning:

- The subsequent flags read observes that changed value, assuming no intervening change.
- Conditional processing is invoked.

This example does not assert that the dependency actually modifies flags. It establishes which read controls the branch.

### Token storage changes after capture

Given an initially read token value of 0x1234, and assuming later operations change its storage to 0xabcd:

- F-6443787f9616 receives the captured value 0x1234.
- F-591e5dd605b9 also receives the captured value 0x1234.
- There is no direct token reread to obtain 0xabcd.

### Finalization returns ordinary values

Given a selected processing outcome, changing only the ordinary return values of F-f323001005c6, F-b3f16c6113a2, or F-591e5dd605b9 does not change the selected status or result through any visible test in this function.

This case does not assume those calls have no other effects.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning and valid domain of the input.
- The concrete object types represented by the record and its nested addresses.
- The meaning of either low flag bit.
- Whether the token identifies a lock, resource, subsystem, or something else.
- Whether the token-taking dependencies form an acquire/release pair.
- Whether either finalization dependency changes reference counts, invalidates handles, or releases memory.
- The destination and external meaning of statuses 0, 9, and 28.
- Dependency behavior on invalid arguments, internal failure, or non-returning control flow.
- Thread-safety, reentrancy, and protection against concurrent record mutation.
- Alignment, byte order, address-space constraints, and behavior if address addition is invalid.
- Full dependency signatures and token argument extension.

The visible address offsets are byte offsets: 4 for the flags and 120 for the token. No input-dependent integer arithmetic is visible beyond address calculation and the two-bit flags mask. The evidence does not justify adding overflow handling, pointer validation, retries, or alternate cleanup paths.

## Evidence

This contract is based solely on the supplied static function body and opaque dependency mapping.

**Established static evidence:** argument widths represented in the body; direct access widths and offsets; the null-pointer branch; token capture; the post-call flags read; the low-two-bit condition; the zero/nonzero processing-result comparison; dependency ordering; status constants; and returned bit patterns.

**Inferred intent:** lookup, conditional processing, finalization, and paired token operations are descriptive interpretations of how the calls are used. They do not establish the dependencies’ internal semantics.

**Unknown:** dependency implementations, ownership guarantees, global-state effects, runtime validity of referenced memory, and concrete external error semantics.

No runtime tests have been performed. The acceptance examples describe consequences of the static control flow, not observed execution.
