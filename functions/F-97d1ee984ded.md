<!-- Trusted provenance: F-97d1ee984ded; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-97d1ee984ded

## Purpose

F-97d1ee984ded conditionally obtains a referenced record, derives a 32-bit adjustment from that record and a zero/nonzero selector, transforms a local 32-bit value through dependencies, and returns whether a subsequently obtained 32-bit result is nonzero.

The established return contract is Boolean-valued: every visible normal return is exactly 0 or 1. The function does not return the derived adjustment, the transformed local value, or the result word itself.

The names and business meanings of the referenced structures are unknown. An interpretation involving record processing or lookup is inferred intent only; it is not part of the established contract.

## Interface

### Inputs

The function has four apparent 32-bit inputs:

| Input | Established use |
|---|---|
| Base address | Used to form addresses at byte offsets 160 and 132. The function passes these addresses to dependencies. |
| Selector | Tested for equality with zero. Zero selects an increment of 8; every nonzero value selects an increment of 12. |
| Third input | Not referenced by the visible body. |
| Initial value | Copied into a local 32-bit working slot before the first dependency call. The slot’s address is passed to subsequent dependencies. |

The base address is represented by an integer in the supplied representation, but it is used as an address. Its signed numerical interpretation is not established.

The third input has no visible influence on control flow, arguments supplied to dependencies, or the returned value. This statement concerns the visible function body; it is not a claim about undocumented calling conventions or external instrumentation.

### Output

The return value is a 32-bit quantity:

- **1:** the full processing path was reached, and the 32-bit word read from the final dependency-provided address was nonzero.
- **0:** the initial gate was nonzero, the retrieved record address was null, or the final result word was zero.

The interface exposes no distinct status for these three zero-return cases.

### Referenced layouts

The visible accesses establish only the following layout fragments:

| Location | Established interpretation |
|---|---|
| Base address + 160 | Address passed first to F-18dde3a4696c and, conditionally, to F-fc900472ddea. No direct field read at this offset is shown. |
| Base address + 132 | Address passed to F-e5963bb0d688. No direct field read at this offset is shown. |
| Address returned by F-fc900472ddea, offset 0 | A pointer-valued word identifying a record, or null. |
| Record address, offset 0 | A signed 16-bit value, extended to the apparent 32-bit integer argument width before being passed to F-2c6bf41c5b21. |
| Address returned by F-e5963bb0d688, offset 0 | A 32-bit word whose zero/nonzero value determines the successful-path return. |

The apparent target model uses 32-bit integers and pointer-valued words. The explicit four-byte quantities and these accesses support that model, but the supplied representation does not independently specify the complete target ABI, alignment rules, or byte order.

### Preconditions for defined normal execution

Each reached dependency must support the arguments supplied to it. In addition:

- The addresses formed from the base must be valid for the operations performed by the dependencies that receive them.
- F-fc900472ddea must return an address from which a pointer-valued word can be read. Its returned address is not checked for null before that read.
- A nonnull record address must permit reading its first signed 16-bit value.
- F-e5963bb0d688 must return an address from which a 32-bit word can be read. That returned address is also not checked for null.
- Address formation and adjustment arithmetic must have defined behavior under the actual execution model.

These are path-dependent requirements. For example, an immediate nonzero gate result avoids both explicit dependency-result dereferences.

## Behavior

### 1. Initialize the local working value

Before invoking any dependency, the function copies the fourth input into a local 32-bit working slot.

The caller does not pass a pointer to this slot. Consequently, the slot is not an output parameter supplied by the caller. It is, however, subsequently exposed to dependencies by address.

The initial copy also occurs on paths that return before using the slot further.

### 2. Apply the initial gate

The function calls F-18dde3a4696c with the address at base + 160.

- If its returned integer is **nonzero**, the function returns 0.
- If its returned integer is **zero**, processing continues.

Any nonzero gate value has the same visible effect, including negative values under the apparent signed-integer interpretation.

On the nonzero-gate path, no other dependency is called. In particular, there is no call to the dependency later used after processing the local working slot.

### 3. Obtain and inspect the record address

The function calls F-fc900472ddea with base + 160, then reads the pointer-valued word at the returned address.

The distinction between the two addresses is material:

- The address returned by F-fc900472ddea is dereferenced without a preceding null check.
- The record address obtained from that dereference is checked for null.

If the record address is null, the function returns 0. It does not call F-2c6bf41c5b21 or any of the later dependencies.

A null return directly from F-fc900472ddea is therefore not an established clean “no record” case. The visible body attempts to dereference that return.

### 4. Derive the adjustment

For a nonnull record address, the function reads the signed 16-bit value at record offset 0. It passes both the record address and the sign-extended value to F-2c6bf41c5b21.

The dependency’s apparent 32-bit integer result is then adjusted:

- Selector equal to zero: add 8.
- Selector not equal to zero: add 12.

The selector is not otherwise normalized, added, or forwarded. Its magnitude and sign do not affect which increment is selected beyond the zero/nonzero distinction.

The visible body does not check the result from F-2c6bf41c5b21 for an error sentinel, clamp it, or validate its range. In particular, a negative result does not itself terminate processing.

The adjustment addition is performed at the apparent 32-bit integer width; no widening or overflow check is visible. For results where signed 32-bit addition would overflow, the supplied representation is insufficient to establish a portable arithmetic result. Wraparound must not be assumed as an acceptance requirement.

### 5. Process the working slot

The function calls F-f4fd698ba468 with:

1. The address of the local 32-bit working slot.
2. The adjusted value from the preceding step.

No return value from this call is used by the visible body.

The function next calls F-e5963bb0d688 with:

1. Base + 132.
2. The address of the same local working slot.

This is the same slot that initially received the fourth input. If F-f4fd698ba468 modifies its contents, F-e5963bb0d688 receives the address of that modified slot. The body does not restore the initial value between these calls.

The supplied evidence does not establish what either dependency does with the slot or with any value it contains.

### 6. Read the result, invoke the final dependency, and return

The function reads a 32-bit word from the address returned by F-e5963bb0d688 and saves that word locally.

It then calls F-8a5aadac202f with the address of the working slot.

Only after this call returns does the function select its normal return value:

- Saved result word nonzero: return 1.
- Saved result word zero: return 0.

This ordering is significant. The decision uses the word read **before** F-8a5aadac202f, not a fresh read afterward. Changes to the dependency-provided result location during F-8a5aadac202f do not change the saved decision value.

F-8a5aadac202f is called on both zero-result and nonzero-result full-processing paths. If it does not return normally, the visible function does not reach its Boolean return.

## Side effects

### Direct effects

The visible body directly:

- Initializes and maintains local working storage.
- Reads one pointer-valued word after F-fc900472ddea returns.
- Conditionally reads one signed 16-bit record field.
- Conditionally reads one 32-bit result word after F-e5963bb0d688 returns.

No direct write to the base-relative locations, the record, or the dependency-provided result location is shown.

### Dependency-mediated effects

The function exposes:

- Base + 160 to two dependencies.
- The record address to F-2c6bf41c5b21.
- Base + 132 to F-e5963bb0d688.
- The working slot’s address to three dependencies.

Those dependencies may have externally observable effects not specified here. The supplied body does not establish that they preserve global state, preserve the record, avoid allocation, or refrain from modifying reachable memory.

The final dependency’s placement is consistent with finalization or cleanup, but its actual behavior is unknown. It must not be described as releasing ownership or freeing memory without additional evidence.

### Ownership and lifetime

No allocation, ownership transfer, retention rule, or release rule is established directly.

The working slot is local to the invocation. Its address is supplied during the calls, but there is no established contract permitting a dependency to retain and use that address after the enclosing invocation ends.

The returned pointer from F-e5963bb0d688 is dereferenced before the final dependency call and is not visibly used afterward. Its allocation source, owner, and subsequent validity remain unknown.

## Dependencies

The following order is established on the complete normal path:

| Order | Opaque ID | Arguments and observed use |
|---|---|---|
| 1 | F-18dde3a4696c | Receives base + 160. Zero permits continuation; nonzero produces return 0. |
| 2 | F-fc900472ddea | Receives base + 160. Its result is dereferenced to obtain the record address. |
| 3 | F-2c6bf41c5b21 | Receives the record address and its signed 16-bit leading value extended to the apparent 32-bit width. Its integer result is adjusted. |
| 4 | F-f4fd698ba468 | Receives the working-slot address and the adjusted integer. No return value is consumed. |
| 5 | F-e5963bb0d688 | Receives base + 132 and the working-slot address. Its result is dereferenced for a 32-bit word. |
| 6 | F-8a5aadac202f | Receives the working-slot address after the result word has been saved. No return value is consumed. |

Early returns truncate this sequence. No retry, alternate dependency, or repeated call is visible.

Dependencies are not assumed to be pure or independent. Later calls may observe state changed by earlier calls.

## Acceptance examples

These are conditional static examples, not results of executed tests. They assume valid reached memory accesses, normally returning dependencies, and defined arithmetic.

### Gate rejects processing

Given a nonzero result from F-18dde3a4696c:

- Return 0.
- Call no subsequent dependency.
- Do not inspect a record or a result word.

The same outcome applies whether the gate result is 1, −1, or another nonzero value.

### Retrieved record address is null

Given a zero gate result and a valid address from F-fc900472ddea whose first pointer-valued word is null:

- Return 0.
- Do not read a signed 16-bit record field.
- Do not call F-2c6bf41c5b21, F-f4fd698ba468, F-e5963bb0d688, or F-8a5aadac202f.

### Zero selector

Given a valid nonnull record and an F-2c6bf41c5b21 result of 20, with selector zero:

- Pass 28 as the adjustment argument to F-f4fd698ba468.
- Continue through the result read and final dependency call.
- Return according to the saved result word, not according to 28.

### Nonzero selector

Under the same conditions, with selector −7 or 5:

- Pass 32 as the adjustment argument.
- Treat both selector values identically.

### Signed leading record field

Given a record whose leading 16-bit pattern is all ones:

- Pass −1 as the widened integer argument to F-2c6bf41c5b21.
- Do not pass 65535.

This example concerns signed interpretation; it does not specify byte order.

### Negative intermediate result

Given an F-2c6bf41c5b21 result of −8 and selector zero:

- Pass an adjusted value of 0 to F-f4fd698ba468.
- Do not terminate merely because the dependency’s result was negative.

### Final result normalization

Given that F-e5963bb0d688 supplies a readable result word:

- A word of zero leads to return 0 after F-8a5aadac202f returns.
- Any nonzero word leads to return 1 after that call returns.
- A word with only its highest bit set still leads to return 1.

### Result changes during the final call

Given a saved result word of zero, if F-8a5aadac202f later changes the original result location to nonzero:

- Return 0, provided the final call returns normally.

Conversely, a saved nonzero word still produces return 1 if the original location is subsequently changed to zero.

## Uncertainties

The following are **documented_only** unknowns:

- Semantic names and complete layouts of the base object, record, working value, and result object.
- Complete target ABI, byte order, alignment requirements, and address-overflow behavior.
- Actual machine behavior for adjustment additions outside the signed 32-bit range.
- Whether any dependency mutates global state, modifies the record, allocates, releases, retains pointers, or transfers ownership.
- Whether the final dependency is a destructor, release operation, reference-count operation, or something else.
- Dependency failure behavior, including exceptions, traps, nonlocal exits, and nontermination.
- Thread-safety, synchronization requirements, and effects of concurrent mutation.
- Behavior after an invalid address or other violated precondition.

There is no visible error recovery for invalid dependency-returned addresses. That absence does not establish which runtime fault, if any, occurs on a particular platform.

## Evidence

**Established static evidence:** The supplied function body supports the branch conditions, dependency order, address offsets, explicit dereferences, signed 16-bit argument conversion, adjustment constants, local-slot reuse, pre-final-call result capture, and exact normal return values described above.

**Inferred intent:** The routine appears to conditionally process a referenced record using a mutable working value. The final dependency may perform finalization. Neither interpretation establishes additional behavior.

**Unknown:** Dependency internals, broader object contracts, ownership, platform details beyond the apparent width model, and runtime failure semantics are not supplied.

No runtime tests have been performed. The acceptance examples are deductions from the supplied static evidence under their stated preconditions.
