<!-- Trusted provenance: F-8ebfdde835aa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8ebfdde835aa

## Purpose

F-8ebfdde835aa converts an unsigned input into an unsigned result constrained to the range 0–255 on every normal return. A field in a caller-supplied object selects between two behaviors:

- A distinguished value of 255 selects threshold subtraction, including an early zero result and a non-returning dependency call for an excessive adjusted value.
- Every other field value selects input capping followed, except for a zero-field early return, by an arithmetic dependency call.

**Established behavior:** the branch selection, field accesses, unsigned comparisons, arithmetic expressions, dependency invocation conditions, and final low-byte extraction are visible in the supplied static representation.

**Inferred intent:** the second behavior resembles scaling a capped input to a byte-sized range. That interpretation depends on the arithmetic dependency behaving like division, which is not established here. Neither this interpretation nor any particular application domain is part of the unconditional contract.

## Interface

### Inputs

The function accepts two arguments:

1. **Object address.** An integer-represented address used to access two unsigned fields at fixed byte offsets:
   - Offset `0x2D4`: the mode-or-limit field.
   - Offset `0x2D8`: the threshold field.

2. **Input value.** An unsigned integer compared with, capped by, or reduced by those fields.

The supplied representation uses unsigned integer values for both fields, the input, dependency arguments and result, and the function result. This contract uses a **32-bit unsigned model** for those values, with range 0–4,294,967,295 and arithmetic modulo 4,294,967,296. The precise machine ABI and native type definitions were not supplied; confirmation that these recovered unsigned types correspond to 32-bit machine values remains `documented_only`.

The object address is represented by a signed integer in the supplied interface. That representation does not establish that negative addresses are accepted, that integer overflow during address formation is meaningful, or that arbitrary integers denote valid storage. The caller must supply an address for which the accesses actually taken are valid.

### Required storage

The function directly accesses unsigned fields at the stated offsets. Under the 32-bit model, each occupies four bytes:

| Offset | Width | Role |
|---|---:|---|
| `0x2D4` | 32 bits | Selects threshold mode when equal to 255; otherwise supplies the cap and second arithmetic-dependency argument |
| `0x2D8` | 32 bits | Threshold used only in threshold mode |

These are unsigned fields, not one-byte fields. The distinguished value 255 does not imply that the mode-or-limit field is stored in a byte.

The supplied material does not establish byte order, required alignment, the complete object layout, or whether the surrounding object contains other meaningful fields. Validity of the threshold-field access is required only when the mode-or-limit value initially read is 255.

### Output

Every normal return is an unsigned integer between 0 and 255 inclusive. This is a numeric range guarantee; it does not establish an eight-bit return ABI.

There is no visible error-status output, output pointer, or explicit object update. Some inputs invoke a dependency marked as non-returning rather than producing a normal result.

## Behavior

### Initial observation and mode selection

The function first reads the mode-or-limit field at offset `0x2D4`.

That observed value governs the remainder of the call. The function does not visibly reread this field, including after calling a dependency. A later change to the object therefore does not alter the value already used for branch selection, capping, or the second arithmetic-dependency argument.

The field value 255 selects threshold mode. All other unsigned values, including zero and values greater than 255, select capped arithmetic mode.

### Threshold mode

When the initially observed mode-or-limit value is 255:

1. The function reads the threshold field at offset `0x2D8` and compares the input against it using an unsigned comparison.
2. If the input is less than or equal to that observed threshold, the function returns zero immediately.
3. Otherwise, the threshold field is read again, and this second observed value is subtracted from the original input.
4. If the resulting unsigned difference exceeds 255, the function invokes F-a817d8d58c10. The supplied representation marks this dependency as non-returning.
5. If the difference is at most 255, the function returns its low eight bits. In this case, the masking does not change the numeric value.

The two threshold reads are distinct evaluation points in the supplied representation. The first establishes whether the function returns early; the second supplies the subtrahend. A specification must not silently replace them with one captured threshold value.

Under stable ordinary storage, a call that reaches subtraction has an input greater than the threshold, so the difference is positive. Its possible normal results are then 1–255.

If the two reads can legally observe different values, the subtraction is instead based on the second value. Under the 32-bit model, subtraction wraps modulo 4,294,967,296. A second threshold equal to the input can produce zero after the earlier comparison allowed execution to continue. A second threshold greater than the input can produce a wrapped difference, potentially invoking the non-returning dependency.

These changing-value cases describe consequences of distinct reads, not a guarantee that unsynchronized concurrent mutation is supported. The source representation alone does not establish volatile access, atomicity, memory-mapped I/O semantics, or a concurrency model.

### Capped arithmetic mode

When the initially observed mode-or-limit value is not 255:

- If the input exceeds the observed field value, the working input is reduced to that field value.
- Otherwise, the working input remains the original input.
- If the observed field value is zero, the function returns zero without calling an arithmetic dependency.
- For every other field value in this mode, the function invokes F-d7be9cfb4000 with:
  - First argument: the capped input multiplied by 255.
  - Second argument: the initially observed mode-or-limit value.
- If that dependency returns, the function returns the low eight bits of its result.

All capping comparisons are unsigned. A value with its high bit set is a large unsigned value, not a negative input.

The zero-field check depends on the field, not on the capped input. Consequently, an input of zero with a nonzero, non-255 field still calls F-d7be9cfb4000, passing zero as its first argument.

The multiplication occurs before the dependency call. Under the 32-bit model, it is reduced modulo 4,294,967,296. It is not established as a widened multiplication, a saturating multiplication, or an exact unbounded product.

The largest capped input whose product with 255 fits in 32 bits is 16,843,009. For larger capped inputs, the first argument can wrap. For example:

- A capped input of 16,843,009 produces 4,294,967,295.
- A capped input of 16,843,010 produces 254 after 32-bit unsigned wraparound.

The final mask extracts the dependency result’s low eight bits. It does not clamp that result to 255. For example, a dependency result of 256 produces zero, whereas 511 produces 255.

No unconditional division, rounding, proportionality, monotonicity, or endpoint-scaling guarantee follows without the contract of F-d7be9cfb4000.

## Side effects

### Direct memory effects

The function visibly performs no stores through the object address and no other explicit memory writes.

Its direct object reads are:

| Path | Reads, in evaluation order |
|---|---|
| Non-255 mode | Mode-or-limit field once |
| Threshold mode, early return | Mode-or-limit field once, then threshold field once |
| Threshold mode, subtraction reached | Mode-or-limit field once, then threshold field twice |

This ordering describes the supplied static representation. It does not establish an exact machine-load count or externally observable access sequence in an executable not supplied for inspection.

### Dependency effects

The absence of direct writes does not make the function globally side-effect-free:

- F-a817d8d58c10 may have effects before its non-returning transfer.
- F-d7be9cfb4000 may modify accessible state before returning.
- Neither dependency’s global-state, memory, I/O, or ownership behavior is supplied.

The function performs no visible object-field reads after either dependency invocation. Effects of a returning arithmetic dependency therefore do not cause this function to recompute its earlier branch or arguments.

### Ownership and lifetime

There is no visible allocation, deallocation, retention of the object address, or explicit ownership transfer.

The object address is not explicitly passed to either dependency. This does not prove that the dependencies cannot access the object through other state, nor that they preserve or release any particular storage. Their ownership behavior remains unknown.

Invalid storage, unsuitable alignment, and inaccessible addresses have no visible recovery path. Their consequences are outside the established normal-return contract.

## Dependencies

### F-a817d8d58c10

- Invoked only in threshold mode after the unsigned adjusted value exceeds 255.
- Receives no explicit arguments in the supplied call representation.
- Marked as non-returning.
- Has no supplied behavioral contract.

It is not established whether this dependency terminates execution, raises an exception, transfers control elsewhere, waits indefinitely, or performs another non-returning action. It must not be described specifically as an assertion, process abort, or diagnostic emitter.

Normal-return behavior for a contrary case in which this dependency returns is not established.

### F-d7be9cfb4000

- Invoked only when the initially read field is neither zero nor 255.
- Receives two unsigned arguments: the potentially wrapped scaled input and the initially read field value.
- Its returned value supplies the final low-byte result.

Its second argument is nonzero at every visible call site in this function. That call-site property does not establish what operation the dependency performs.

The supplied material does not establish its arithmetic semantics, exceptional behavior, termination, memory effects, or global-state preservation.

## Acceptance examples

These are conditional contract examples derived from static evidence. No runtime tests have been performed.

### Stable threshold cases

Assume valid storage, a mode-or-limit value of 255, and a threshold that remains 100 throughout the call:

| Input | Required behavior |
|---:|---|
| 99 | Return zero; no dependency call |
| 100 | Return zero; no dependency call |
| 101 | Return 1; no dependency call |
| 355 | Return 255; no dependency call |
| 356 | Invoke F-a817d8d58c10; no normal return is established |

For the first two rows, the threshold is evaluated once. For the remaining rows, subtraction requires a second threshold evaluation.

### Distinct threshold observations

Assume an execution model in which two threshold evaluations can legally observe different values:

- With mode 255 and input 200, a first threshold observation of 100 allows subtraction. A second observation of 200 yields a difference of zero and a normal result of zero.
- With the same mode, input, and first observation, a second observation of 201 yields 4,294,967,295 under 32-bit subtraction and invokes F-a817d8d58c10.

These cases must not be used to infer support for data races.

### Zero field

With mode-or-limit zero, every unsigned input returns zero without invoking either dependency. The threshold field is not accessed.

### Capping and conditional arithmetic results

With mode-or-limit 100 and input 150:

- The arithmetic dependency receives 25,500 and 100.
- If it returns 255, the function returns 255.
- If it returns 256, the function returns zero.
- The actual dependency return value is not established here.

With mode-or-limit 100 and input zero, the arithmetic dependency is still invoked, with arguments zero and 100.

With mode-or-limit 256 and input 256, capped arithmetic mode applies rather than threshold mode. The dependency receives 65,280 and 256.

### Multiplication-width boundary

Under the 32-bit model:

- With mode-or-limit and input both 16,843,009, the arithmetic dependency receives 4,294,967,295 and 16,843,009.
- With mode-or-limit and input both 16,843,010, it receives 254 and 16,843,010.

In either case, a normal function result is the low eight bits of whatever the dependency returns.

## Uncertainties

The following remain `documented_only`:

- The exact machine ABI, unsigned integer width, address width, byte order, and alignment requirements.
- Whether the supplied field evaluations correspond to separate observable machine reads.
- Whether field mutation during a call is supported under the actual memory model.
- The operation, result width, exceptional behavior, and side effects of F-d7be9cfb4000.
- The mechanism and effects of the non-returning F-a817d8d58c10 call.
- Object ownership rules, full layout, valid address range, and surrounding synchronization requirements.
- Whether callers impose smaller field or input ranges that exclude multiplication wraparound.

The apparent byte-scaling purpose is inferred intent, not a substitute for these missing contracts.

## Evidence

The evidence is limited to the supplied static function representation and the opaque dependency mapping.

It directly supports:

- Field offsets and unsigned treatment.
- Initial mode selection by equality with 255.
- The early-return conditions.
- Two threshold evaluations on the subtraction path.
- Input capping in the alternative mode.
- Multiplication by 255 before the arithmetic dependency call.
- The conditional non-returning dependency call.
- Low-eight-bit extraction on the final return path.
- The absence of explicit stores, allocation, or release operations in this function.

The wraparound examples are arithmetic deductions under the stated 32-bit model. Dependency semantics, executable behavior, and application intent have not been independently verified. No runtime tests have been performed.
