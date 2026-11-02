<!-- Trusted provenance: F-ea646ab31dfc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ea646ab31dfc

## Purpose

F-ea646ab31dfc conditionally invokes an indirect destination after two ordered byte checks. It calls F-b1ee7d5e02f9, converts the result to a byte address, and returns if the byte read there is zero. Otherwise, it calls F-b1ee7d5e02f9 again, retains that result in a recovered signed integer intermediate, adds one, converts the sum to a byte address, and reads another byte. A nonzero second byte causes an immediate return.

Only when the first examined byte is nonzero and the second examined byte is zero does execution reach a further value-producing dependency and the indirect destination. The destination receives the dependency’s result, the first input shifted left by two positions, and the unchanged second input.

The second address calculation is not established as unconditional pointer-offset arithmetic. Its recovered signed addition and subsequent integer-to-address conversion impose uncertainties separate from memory readability.

**Inferred intent:** if both address-producing calls identify the same stable string and the recovered address calculations have the intended meaning, the checks resemble recognition of a one-byte, zero-terminated string. Those assumptions are not established. The general contract concerns two separate calls and two separate observations.

The function exposes no return value or explicit indication of whether dispatch occurred.

## Interface

### Inputs and output

| Item | Recovered representation | Established use |
|---|---|---|
| First input | Signed integer, treated as 32 bits in this analysis | Shifted left by two positions on the dispatch path; supplied as the second destination argument |
| Second input | Opaque 32-bit quantity | Supplied unchanged as the third destination argument |
| Function result | No exposed value | No status or destination result is returned |

The first input does not select either guard outcome. Its shift is reached only after both byte checks pass.

The second input is not directly dereferenced, validated, or interpreted. Its meaning could be data, an address representation, a handle, or another convention-specific quantity; none is established.

The 32-bit signed interpretation follows the recovered integer representation. Independent target-ABI confirmation is unavailable. In particular, this interpretation does not independently establish pointer width or an integer-to-address mapping.

### Memory and evaluation requirements

Normal completion requires more than readable storage:

- The first dependency result must convert to an address at which one byte can be read.
- If the first byte is nonzero, the second dependency result must be representable in the recovered signed intermediate according to the applicable return and conversion conventions.
- Adding one to that intermediate must have defined behavior under the applicable arithmetic model.
- Converting the resulting integer to a byte address must yield an address suitable for the second read.
- If both guards pass, the later dependency must return normally, the shifted argument must have defined semantics, and the indirect destination must be callable under the actual calling convention.

These are requirements for a defined normal path, not validations performed by the function.

Readable storage at an intended “second byte” does not by itself establish that the signed addition and address conversion used to reach it are defined. No explicit null check, range check, arithmetic-overflow check, or destination-validity check is present.

No larger buffer layout, allocation size, encoding, or alignment guarantee is supplied.

## Behavior

### First guard

F-b1ee7d5e02f9 is invoked without explicit arguments. Its result is converted to a byte address, and the byte at that address is read.

If that byte is zero, F-ea646ab31dfc returns immediately. There is no second address-producing call, second address calculation, second byte read, value-producing call, or indirect invocation.

If the byte is nonzero, execution continues. Every nonzero byte satisfies this condition; no particular character set or printable-character requirement applies.

### Second guard and integer-mediated address formation

F-b1ee7d5e02f9 is invoked a second time, again without explicit arguments. This result is retained in a recovered signed integer intermediate rather than directly reused as a byte pointer.

The visible address formation consists of three distinct operations:

1. Obtain the signed integer intermediate from the second dependency result.
2. Add one to that integer.
3. Convert the resulting integer to a byte address for the read.

The function then reads one byte through that address. A nonzero byte causes immediate return. A zero byte permits dispatch preparation.

Under the recovered signed 32-bit interpretation, adding one is arithmetically representable for intermediate values from −2,147,483,648 through 2,147,483,646 inclusive. At 2,147,483,647, the mathematical sum exceeds the signed 32-bit maximum. Ordinary signed arithmetic does not define a wrapped result for that case.

The supplied representation does not establish whether the actual execution has wrapping arithmetic, some other machine-specific interpretation, or a failure. None is selected by this contract.

Even when the addition is representable, conversion of the resulting integer to a byte address remains dependent on the target’s address and conversion conventions. Arithmetic representability alone does not prove that the converted address is valid or that it corresponds to one byte beyond an address encoded by the original integer.

Consequently, “offset one from the second returned address” is only a conditional interpretation: it applies when the return representation, signed arithmetic, and address conversion support that relationship.

### Relationship between the reads

The first dependency result is not reused for the second check. The two results are not compared.

No byte corresponding to offset zero of the second result is checked. No byte corresponding to offset two is checked. There is no general string scan.

When the second calculation does correspond to an ordinary one-byte address advance, the second guard examines that advanced address regardless of what a byte at the unadvanced address contains. Passing both guards therefore does not establish that either returned region is a nonempty, one-byte string.

The byte comparisons are exact zero tests. Character signedness does not affect whether a byte is zero or nonzero.

### Dispatch preparation

After both guards pass, a dependency without a supplied opaque function ID is invoked without explicit arguments. Its recovered result is an opaque 32-bit quantity.

The indirect destination is then supplied three arguments in this positional order:

1. The result of that value-producing dependency.
2. The first input shifted left by two bit positions.
3. The unchanged second input.

Argument positions are established. The representation does not establish a machine-level order for evaluating and placing every argument or for fetching the indirect destination relative to all preparation operations. The value-producing dependency must produce its result before that result can be consumed by the destination.

The destination is selected through a stored dispatch location. Earlier dependencies are not known to preserve that location. A value observed there before a dependency executes cannot be assumed to remain the destination used later.

The recovered representation treats the indirect transfer as a call followed by return. It does not use any destination result. The underlying transfer’s exact calling and return convention remains unresolved.

### First-input arithmetic

The visible first-input transformation is a left shift by two, not checked multiplication.

Under an ordinary signed 32-bit interpretation, the shift has a defined numeric result for nonnegative inputs from 0 through 536,870,911 inclusive. In that range, the result equals four times the input and is at most 2,147,483,644.

For negative inputs, or positive inputs whose shifted result exceeds the signed range, the supplied evidence does not establish a portable result. Wrapping, saturation, rejection, and checked error reporting are not established.

This uncertainty is separate from the earlier signed addition used for the second read. The addition is reached when the first guard passes; the shift is reached only when both guards pass.

### Errors and abnormal completion

There is no explicit error result, recovery branch, or local fault handling.

Invalid address conversion, invalid memory access, arithmetic outside the defined recovered model, an unusable destination, or abnormal completion of a dependency does not have an established normal-return interpretation.

In particular, signed-addition overflow is not specified as a guard failure. It is not specified to wrap to another address, trap, or return from the function. The same restraint applies to unsupported shift inputs.

A dependency that fails to return normally prevents the remaining local sequence from being guaranteed. Exception, fault, and nonlocal-exit policies are **documented_only**.

## Side effects

The function directly performs up to two byte reads and accesses a stored indirect destination on the dispatch path. It contains no explicit application-memory write, allocation, deallocation, or ownership-transfer operation.

For paths with valid arithmetic, conversions, accesses, and normally returning dependencies:

| Guard outcome | F-b1ee7d5e02f9 calls | Byte reads | Value-producing calls | Indirect invocations |
|---|---:|---:|---:|---:|
| First byte is zero | 1 | 1 | 0 | 0 |
| First byte is nonzero; second byte is nonzero | 2 | 2 | 0 | 0 |
| First byte is nonzero; second byte is zero | 2 | 2 | 1 | 1 |

This table does not prescribe what happens if the second signed addition, either address conversion, or an access lacks defined semantics.

An early return is not necessarily side-effect-free: at least one dependency call has already occurred. The dependencies may change globals, alter the dispatch location, mutate storage, allocate memory, release memory, or perform external operations. Their behavior is not established here.

Each dependency-produced address is used for its immediate read. The function does not explicitly release either result. That absence does not establish borrowed, static, caller-owned, or retained storage, nor continued validity across another dependency call.

The second input is forwarded without an explicit local ownership operation. Any ownership effect imposed by the destination remains unknown.

## Dependencies

### F-b1ee7d5e02f9

Called once unconditionally and a second time only if the first examined byte is nonzero.

The two results have different visible consumption paths:

- The first is converted directly to a byte address for a read.
- The second is retained in a signed integer intermediate, incremented by one, and then converted to a byte address for a read.

The second path requires both defined arithmetic and a usable address conversion. Describing both results simply as interchangeable pointers would omit this distinction.

Not established: return identity across calls, shared storage, string termination, integer-return interpretation, storage lifetime, purity, ownership, or preservation of global state.

### Value-producing dependency — documented_only

No opaque function ID is supplied for this dependency; none is fabricated.

It is reached only after both byte checks pass. It takes no explicit arguments in the recovered representation and produces an opaque 32-bit result used as the first destination argument.

Its semantic meaning, side effects, ownership behavior, and failure modes are unknown.

### Indirect destination — documented_only

No opaque function ID is supplied for this dynamically selected destination.

Its established call-site interface consists of the three argument positions described above. Its actual signature, argument meanings, calling convention, return behavior, and side effects are unresolved.

The stored destination is not assumed immutable across earlier calls.

## Acceptance examples

These examples are conditional specifications, not executed tests. Unless an example expressly examines an uncertainty, required arithmetic and conversions are defined, reads are valid, dependencies return normally, and the destination is callable.

### Immediate return avoids later arithmetic

The first converted address contains a zero byte.

Expected behavior: one F-b1ee7d5e02f9 call and one byte read, followed by return. Neither the second signed addition nor the first-input shift is reached.

This local path is independent of both input values, including first-input values outside the shift’s defined domain.

### Second guard rejects

The first examined byte is `0x41`. The second dependency result has a valid signed representation; adding one is defined; conversion of the sum yields a readable address containing `0x42`.

Expected behavior: two address-producing calls and two byte reads, then return. The value-producing dependency and indirect destination are not invoked.

### Dispatch with a defined shift

The first examined byte is `0xFF`. The second signed addition and address conversion are valid, and the second examined byte is zero. The first input is 7.

Expected behavior: the value-producing dependency is invoked. The destination receives its result as argument one, 28 as argument two, and the original second input as argument three.

The first byte need not be an ASCII character.

### Distinct storage can still pass

The first read observes a nonzero byte in one region. The second dependency result, after valid integer addition and address conversion, identifies a zero byte in another region.

Expected behavior: both guards pass. No equality between the dependency results is required.

Even if the second result has an ordinary address interpretation and its unadvanced address also contains zero, that byte is not checked.

### Maximum signed second intermediate

The first guard passes, and the second result is retained as 2,147,483,647 under the recovered signed 32-bit interpretation.

Adding one exceeds that signed range. This contract specifies no resulting address, second byte value, normal return, or dispatch outcome.

Mapped readable storage at a hypothetical wrapped address does not resolve the arithmetic uncertainty.

### Representable addition is not address validity

The second signed intermediate is 2,147,483,646. Adding one gives 2,147,483,647 without signed overflow.

This establishes only the arithmetic result. A valid second read still requires the target’s integer-to-address conversion to yield suitable readable storage.

### Shift boundary is independent

With passing guards and first input 536,870,911, the shifted argument is 2,147,483,644 under the recovered signed 32-bit interpretation.

With passing guards and first input 536,870,912, no portable shifted result is specified. Valid second-address formation does not resolve this separate uncertainty.

## Uncertainties

The following remain **documented_only**:

- Exact target ABI, pointer width, and independent confirmation of recovered integer widths and signedness.
- How the second dependency result acquires its signed integer representation.
- Actual behavior when adding one exceeds that signed representation’s range.
- Integer-to-address conversion rules, including whether a representable increment corresponds to a one-byte address advance.
- Shift behavior outside the defined signed 32-bit domain.
- Semantic meanings of the inputs and value-producing result.
- Stability, validity, lifetime, mutation, synchronization, and ownership of returned storage.
- Identity and semantics of the value-producing dependency.
- Identity, actual signature, and return behavior of the indirect destination.
- Dependency effects on globals, including the dispatch location.
- Fault, exception, and nonlocal-exit behavior.

The one-byte-string interpretation remains inferred intent only. It requires storage stability and address-representation assumptions not established by the visible operations.

## Evidence

This contract is based solely on the supplied static representation and opaque dependency mapping.

Established static evidence includes two separate F-b1ee7d5e02f9 calls; the first direct byte-address conversion; the second result’s signed integer intermediate, addition by one, and subsequent byte-address conversion; both zero-based guard decisions; the later value-producing call; the left shift by two; and the three destination argument positions.

The recovered indirect transfer does not establish the destination’s full calling contract. Ordinary signed-arithmetic reasoning identifies the addition and shift boundaries under the recovered 32-bit interpretation; it does not independently establish actual machine behavior outside those boundaries.

No runtime execution, tests, memory inspection, implementation access, or independent ABI verification has been performed. The acceptance examples are conditional expectations, not empirical results.
