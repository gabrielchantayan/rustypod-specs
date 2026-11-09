<!-- Trusted provenance: F-aab9ae3f38fb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aab9ae3f38fb

## Purpose

F-aab9ae3f38fb selects one of seven externally stored formatting descriptors according to an unsigned quantity, resolves that descriptor through dependencies, and either:

- formats into temporary local storage, passes that storage to an output dependency, and returns the formatter’s result; or
- invokes an indirect operation associated with the supplied output object and returns that operation’s result.

The thresholds are powers of 1,024. This supports an inferred intent of presenting a size-like quantity using binary-scaled units, with separate formatting variants below and at a scaled value of 100. The actual units, text, precision, localization, and rounding are not established because descriptor contents and dependency implementations are absent.

This contract describes the visible static behavior. It does not claim successful execution, valid output text, or memory safety for any particular invocation.

## Interface

### Inputs

The recovered interface contains five arguments:

| Position | Width | Established role |
|---|---:|---|
| 1 | 32 bits | No explicit use is visible in the recovered body. |
| 2 | 32 bits | No explicit use is visible in the recovered body. |
| 3 | Unsigned 32 bits | Low word used for threshold tests and remainder arithmetic. |
| 4 | Unsigned 32 bits | High word used for threshold tests and range qualification. |
| 5 | Pointer-sized value in an apparent 32-bit layout | Passed to the output dependency on one path; dereferenced to locate an indirect operation on the other. |

For describing branch selection, let:

- **L** be the unsigned value of argument 3.
- **H** be the unsigned value of argument 4.
- **N** be the mathematical value \(L + 2^{32}H\).

The combination into **N** is a specification convenience. It exactly expresses the visible threshold decisions without requiring the implementation to perform a native 64-bit calculation.

Arguments 1 and 2 must not be declared semantically irrelevant beyond the recovered explicit operations. The formatter receives a stack-based argument reference whose contents are not established; that prevents a complete account of which incoming values might be consumed indirectly.

### Output

The function returns one 32-bit value:

- On the formatting path, the value returned by F-063ea81091b3.
- On the fallback path, the value returned by the indirect operation.

No return from F-151ab4bfe553 replaces the saved formatter result.

The return value’s semantic type is unknown. In particular, the evidence does not establish that it is a byte count, success flag, error code, or pointer.

### Preconditions and layout

The supplied output reference must satisfy the requirements of whichever path executes:

- On the formatting path, it must be acceptable to F-151ab4bfe553.
- On the fallback path, its first 32-bit field must provide a usable address for a table-like structure. The 32-bit entry at byte offset 12 from that address must identify a callable operation.

The fallback layout is established by the visible address operations. Calling it a conventional object dispatch table is an interpretation, not a confirmed type declaration.

All selected descriptor storage, dependency code, and stack-based formatter inputs must be accessible as required by the underlying execution environment. No local validation enforces those conditions.

## Behavior

### 1. Select a descriptor class

Descriptor selection depends on **N**, with the following exact, half-open intervals:

| Quantity interval | Selected descriptor class | Preliminary helper |
|---|---|---|
| \(0 \le N < 1{,}024\) | Unscaled | None |
| \(1{,}024 \le N < 102{,}400\) | Small 1,024-scale | F-1e0edc6e0bc9 |
| \(102{,}400 \le N < 1{,}048{,}576\) | Large 1,024-scale | None |
| \(1{,}048{,}576 \le N < 104{,}857{,}600\) | Small 1,048,576-scale | F-1e0edc6e0bc9 |
| \(104{,}857{,}600 \le N < 1{,}073{,}741{,}824\) | Large 1,048,576-scale | None |
| \(1{,}073{,}741{,}824 \le N < 107{,}374{,}182{,}400\) | Small 1,073,741,824-scale | F-1e0edc6e0bc9 |
| \(107{,}374{,}182{,}400 \le N \le 2^{64}-1\) | Large 1,073,741,824-scale | None |

These seven classes identify seven distinct global storage locations. They do not imply that the values stored there are distinct.

There is no additional larger-scale class. Even the maximum representable pair of input words selects the large 1,073,741,824-scale class.

### 2. Perform the conditional preliminary call

Only the three “small” scaled classes invoke F-1e0edc6e0bc9.

For such a class, let **U** be its scale: 1,024, 1,048,576, or 1,073,741,824. The helper receives two 32-bit words:

- First word: \(L \bmod U\).
- Second word: zero.

The helper’s return value is not explicitly consumed by the recovered body.

This remainder description accounts for the widths and wraparound of the visible arithmetic. For each scale, the rounded-down multiple of **U** cannot exceed **L**, so the displayed low-word subtraction cannot borrow. The high-word expression cancels to zero under unsigned 32-bit arithmetic. Consequently, the helper does not receive **H** as a nonzero second word, even when the selected quantity is above \(2^{32}-1\).

The branch decision is already made before this call. The helper is not known to be pure: it may alter global state or other accessible storage. Its lack of an explicitly used return value does not establish that the call is dispensable.

### 3. Acquire context, then read the selected descriptor

Every descriptor class calls F-9d0ecc5f6adf exactly once.

If a preliminary helper call is required, it precedes this call. After F-9d0ecc5f6adf returns, the function reads the descriptor value from the global storage location selected by the earlier branch.

The ordering is significant:

1. Select the class using the input words.
2. Invoke F-1e0edc6e0bc9 if required.
3. Invoke F-9d0ecc5f6adf.
4. Read the selected global descriptor value.

Thus the descriptor value is not snapshotted before either dependency runs. A dependency’s mutation of the selected storage can affect the value subsequently read. No assumption that either dependency preserves globals is part of this contract.

### 4. Resolve the descriptor

F-c0b020cd3882 receives:

1. The 32-bit result of F-9d0ecc5f6adf.
2. The descriptor value read from the selected global storage location.

Its result is treated as a 32-bit value and tested against zero. The visible control flow distinguishes only zero from nonzero; it does not test positivity or classify negative signed values as errors.

A zero result selects fallback dispatch. Any nonzero bit pattern selects formatting.

### 5. Nonzero resolution: format and pass the temporary buffer

For a nonzero resolver result, the function calls F-063ea81091b3 with four visible inputs:

- A pointer to local temporary storage.
- A size-like argument equal to 512.
- The resolver’s nonzero result.
- A reference to a stack-based argument area.

The temporary storage is represented in the supplied listing as a 508-byte local region. That differs from the 512 passed to the formatter. The evidence does not establish whether this discrepancy reflects an inaccurate recovered stack layout, additional usable storage, or a real size mismatch.

After the formatter returns, the function invokes F-151ab4bfe553 with:

1. The caller-supplied output reference.
2. The pointer to that same temporary storage.

This call occurs regardless of the formatter’s returned bit pattern. There is no visible check for formatting failure, truncation, empty output, or termination before the temporary storage is passed onward.

After F-151ab4bfe553 returns normally, the function returns the previously obtained formatter result.

### 6. Zero resolution: indirect fallback

For a zero resolver result, the function does not call either the formatter or F-151ab4bfe553.

Instead, it:

- Reads the first 32-bit field through the output reference.
- Uses that value as the base address of a table-like structure.
- Reads a callable entry at byte offset 12.
- Invokes that entry.
- Returns the entry’s 32-bit result.

No explicit arguments to this indirect operation are recovered. An implicit receiver, register-based parameters, or other calling-convention details remain unknown.

## Side effects

### Observable call and read order

The established sequence is:

1. Optional preliminary helper call.
2. Context-acquisition call.
3. Read of the selected global descriptor.
4. Resolver call.
5. Either:
   - formatter call, then output call; or
   - output-object field read, table-entry read, then indirect call.

The output-object dispatch fields are read only after the resolver returns zero. Their contents must not be assumed to match values they held before earlier dependencies ran.

### Storage and ownership

The function supplies local temporary storage to the formatter and then to the output dependency. Its lifetime does not extend beyond this invocation.

No ownership transfer is established. In particular:

- The function does not explicitly allocate heap storage.
- It does not explicitly release the output object, descriptor, context result, or resolved value.
- It does not establish whether any dependency allocates or releases memory.
- It does not establish whether F-151ab4bfe553 copies, consumes synchronously, or retains the temporary-buffer pointer.

If safe behavior requires the output dependency not to retain that pointer, that requirement must come from the dependency contract; it is not proven here.

### Errors and exceptional behavior

The only explicit local alternative is the zero resolver result, which triggers fallback rather than an immediate error return.

There are no visible local guards for:

- Invalid output references or dispatch entries.
- Invalid descriptor or context values.
- Formatter failure or insufficient buffer capacity.
- Unterminated formatted data.
- Dependency exceptions, faults, or non-return.

If a dependency does not return normally, no subsequent local call or return is established.

## Dependencies

| Opaque ID | Established interaction | Not established |
|---|---|---|
| F-1e0edc6e0bc9 | Called only for the three small scaled classes, with a remainder word and a zero word. Its return is not explicitly consumed. | Purpose, global effects, memory effects, and any implicit result channel. |
| F-9d0ecc5f6adf | Called once before reading the selected descriptor. Its 32-bit result becomes the resolver’s first argument. | Whether the result is a handle, pointer, identifier, or status; ownership and mutation behavior. |
| F-c0b020cd3882 | Receives the acquired value and selected descriptor. Zero selects fallback; nonzero selects formatting. | Lookup semantics, result lifetime, ownership, and failure taxonomy. |
| F-063ea81091b3 | Receives temporary storage, 512, the resolver result, and a stack-based argument reference. Its result becomes the outer return value if subsequent output completes. | Formatting grammar, argument layout, bounds guarantees, termination, and return semantics. |
| F-151ab4bfe553 | Receives the output reference and formatted-storage pointer after the formatter returns. | Copying, retention, output destination, ownership, and error reporting. |
| Indirect operation | Loaded from byte offset 12 in the table addressed by the output object’s first field; called on zero resolution. | Stable function identity, complete call signature, side effects, and return meaning. |

## Acceptance examples

These are conditional static acceptance examples, not executed tests. They specify branch selection and visible call behavior without inventing descriptor text.

1. **Zero quantity:** With both quantity words zero, select the unscaled descriptor and skip F-1e0edc6e0bc9.

2. **Last unscaled quantity:** With **H** zero and **L** equal to 1,023, remain unscaled. At 1,024, select the small 1,024-scale descriptor and invoke the preliminary helper with words 0 and 0.

3. **Nonzero small-scale remainder:** At **N** = 1,537, select the small 1,024-scale descriptor and invoke the helper with words 513 and 0.

4. **First scaled cutoff:** At **N** = 102,399, select the small 1,024-scale descriptor and pass remainder 1,023 to the helper. At 102,400, select the large 1,024-scale descriptor and omit that helper.

5. **Next scale transition:** At **N** = 1,048,576, select the small 1,048,576-scale descriptor and invoke the helper with zero remainder. At 104,857,600, select its large counterpart and omit the helper.

6. **Nonzero high word:** With **H** = 1 and **L** = 7, **N** is \(2^{32}+7\). Select the small 1,073,741,824-scale descriptor and invoke the helper with words 7 and 0.

7. **Largest small-class boundary:** With **H** = 24 and **L** = 4,294,967,295, **N** is one below 100 times 1,073,741,824. Select the small largest-scale descriptor and pass remainder 1,073,741,823. With **H** = 25 and **L** = 0, select its large counterpart and omit the helper.

8. **Descriptor mutation:** If F-9d0ecc5f6adf changes the selected global descriptor and returns normally, the resolver receives the value read after that change, not a value captured before the call.

9. **Nonzero resolver and formatter result:** If resolution returns nonzero and formatting returns a particular 32-bit value, invoke F-151ab4bfe553 with the temporary buffer. If that output call returns normally, return the formatter’s value unchanged.

10. **Zero resolver:** If resolution returns zero, skip formatting and the direct output dependency. Read and invoke the offset-12 dispatch entry, returning its result.

## Uncertainties

The following are **documented_only**:

- Actual descriptor values, text, unit labels, precision, and rounding.
- The semantic role of the first two arguments.
- The contents and validity of the formatter’s stack-based argument area.
- The relationship between the 508-byte recovered local region and the formatter’s 512 argument.
- Complete calling conventions, especially the indirect fallback operation.
- Dependency ownership, retention, global mutation, and error contracts.
- Whether formatted data is always terminated or safe for the output dependency.
- Whether the preliminary helper establishes hidden state used later.

The inferred size-formatting intent must not override these unknowns or impose unobserved output requirements.

## Evidence

Established behavior comes solely from the supplied static listing: unsigned comparisons and shifts, 32-bit arithmetic, seven descriptor-selection locations, dependency-call order, the zero/nonzero resolver branch, formatter and output arguments, and fallback dispatch through byte offset 12.

The interval table and remainder descriptions are mathematical reductions of those visible operations. Size presentation and table-based object dispatch are interpretations, explicitly separated from established behavior.

No dependency implementations, descriptor contents, runtime traces, or external behavioral documents were provided. No runtime tests have been performed.
