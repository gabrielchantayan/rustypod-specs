<!-- Trusted provenance: F-2cc6ced0beea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2cc6ced0beea

## Purpose

F-2cc6ced0beea copies a two-word input into a word-reversed destination and writes a separate exponent-like result. When the initially observed input exponent field is zero, it invokes F-9f634613286e and derives the exponent-like result from that dependency’s return value. In that branch, a nonzero mode also causes the returned words to replace the initial destination contents.

The established behavior is a word transformation and conditional dependency call. Interpreting the input as an IEEE 754 binary64 value is plausible because the field extraction excludes the upper word’s sign bit and selects its next eleven bits. That interpretation is inferred intent, not a requirement that the input contain a valid floating-point value.

The function does not itself distinguish zero from a subnormal value, or infinity from a NaN. It does not convert the extracted field into a conventional unbiased binary64 exponent.

## Interface

The function takes four arguments:

| Argument | Required representation | Role |
|---|---|---|
| Input | Address of two consecutive 32-bit unsigned words | Supplies the original words and, in one branch, words passed to the dependency |
| Word destination | Address of two consecutive writable 32-bit words | Receives the initial reversed copy and possibly the dependency result |
| Exponent destination | Address of one writable 32-bit word | Receives the final exponent-like result |
| Mode | 32-bit signed integer | Passed twice to the dependency; zero versus nonzero controls replacement of the word destination |

There is no explicit return value or status result.

For descriptive purposes:

- The input’s first word is the word at byte offset 0.
- The input’s second word is the word at byte offset 4.
- The word destination’s first and second words occupy the corresponding offsets.
- The dependency’s 64-bit result has a low 32-bit half and a high 32-bit half, defined by numeric bit position rather than memory byte order.

The word interpretation places the input’s second word in the high-word role. The initial destination layout reverses those word positions: second input word first, first input word second. This establishes word order, not the byte order within an individual word.

### Preconditions

For defined, successful execution:

- The accessed input words must be readable.
- Both destination words and the exponent destination must be writable.
- Pointer alignment and object access must satisfy the execution environment’s rules for 32-bit word accesses.
- If the dependency is reached, its own preconditions must also hold. Those preconditions are not supplied.

The function performs no explicit null-pointer, length, alignment, or overlap validation.

The supplied body does not establish a non-overlap requirement. Overlapping storage can change the values read after the initial write; consequently, the ordinary reversed-copy description is valid without qualification only for non-overlapping storage.

## Behavior

### Initial reads and writes

The function first reads and retains the input’s second word.

It then performs these destination writes in order:

1. It writes that retained word to the word destination’s first position.
2. It reads the input’s first word and writes the value read to the word destination’s second position.

The second step does not necessarily use the input’s original first word. If the first destination write overlaps that input word, the subsequent read sees the changed storage under the supplied access model.

The retained second input word determines which branch executes. Later writes to input storage do not change that branch decision.

These are the established source-level access dependencies. They do not establish atomicity, synchronization, or an exact machine-instruction trace.

### Initial exponent-field selection

The initial exponent field consists of bits 20 through 30, inclusive, of the retained second input word. Bit 31 is excluded. The resulting field is an unsigned value from 0 through 2047.

Only whether this field is zero affects control flow.

### Nonzero initial exponent field

If the initial field is nonzero:

- F-9f634613286e is not called.
- The mode has no effect.
- No further word-destination writes are performed by this function.
- The selected field is written to the exponent destination.

This includes an all-ones field, which produces 2047. There is no special rejection or alternate handling for encodings that might represent infinities or NaNs.

The function does not subtract a floating-point exponent bias in this branch.

### Zero initial exponent field

If the initial field is zero, the function calls F-9f634613286e with six arguments in this order:

1. The input’s first word, read after both initial destination writes.
2. The input’s second word, also read after those writes.
3. The integer constant 64.
4. The mode.
5. The exponent-destination address.
6. The same mode again.

The two input-word arguments are fresh reads, not necessarily the original input values. Their relative evaluation order is not established by the supplied body, but both occur after the initial destination assignments.

The dependency is called regardless of whether the mode is zero. Zero mode therefore does not mean “skip processing.”

The dependency’s result is treated as a 64-bit value. The function separates its high and low 32-bit halves.

If the mode is nonzero, the function writes:

1. The returned high half to the word destination’s first position.
2. The returned low half to the word destination’s second position.

Any nonzero mode selects these writes, including a negative mode. The mode is not normalized to one before being passed to the dependency.

If the mode is zero, this function performs neither of those replacement writes. The dependency itself may still have side effects; its implementation is unknown.

The function then selects bits 20 through 30 of the returned high half and subtracts 64 using 32-bit unsigned arithmetic. That result is written to the exponent destination.

The masking applied to the returned high half excludes its sign-position bit. Its other selected bits form an eleven-bit field, just as in the initial branch decision.

### Final write and arithmetic boundaries

The exponent-destination write occurs last among this function’s explicit writes on every normally returning path.

In the dependency branch, subtraction wraps modulo \(2^{32}\):

- A returned field of 64 produces 0.
- A returned field of 65 produces 1.
- A returned field of 63 produces 4,294,967,295.
- A returned field of 0 produces 4,294,967,232.
- A returned field of 2047 produces 1983.

These are stored unsigned bit patterns. A consumer may interpret those patterns differently, but this function does not establish a signed exponent interface.

## Side effects

### Direct writes

On a normally returning invocation, the function directly writes both words of the word destination before making its branch decision effective.

It additionally writes both destination words after the dependency call when:

- The initial exponent field was zero; and
- The mode is nonzero.

It finally writes the exponent destination.

The input is not written through the input argument. Its storage can nevertheless change when it overlaps either destination.

### Aliasing and write precedence

Overlapping arguments make access order observable.

If the input and word destination are exactly the same two-word region, the initial first write replaces the input’s first word with its second word. The subsequent read of the input’s first word therefore reads that replacement. Both words become equal to the original second word.

If the exponent destination overlaps either destination word, the final exponent write replaces that word’s preceding value. This holds even when the dependency result was copied there immediately beforehand.

If the exponent destination overlaps input storage, the final write changes that storage. It does not retroactively affect the initial branch decision or the dependency arguments, which were obtained earlier.

Other partial overlaps follow the same ordered-access rules. There is no temporary two-word snapshot protecting the original pair.

### Dependency effects and ownership

F-9f634613286e receives the exponent-destination address. It may modify that storage before this function’s final write. On normal continuation, the final write overwrites the 32-bit value at that address.

No assumption is established about the dependency preserving globals, modifying other reachable storage, retaining the address, or owning or releasing memory. Those matters are documented_only.

F-2cc6ced0beea itself contains no explicit allocation, deallocation, or ownership-transfer operation.

### Failure behavior

There is no explicit validation failure, error code, or recovery path.

Invalid memory accesses are not converted into a defined status result. If the dependency does not return normally, the subsequent result-copy and final exponent write are not established to occur.

The initial destination writes precede the dependency call. There is no rollback if a later operation fails.

## Dependencies

### F-9f634613286e

This is the only explicit dependency.

Its established invocation contract here is:

- Called exactly once if the initially retained second input word has a zero eleven-bit exponent field.
- Not called otherwise.
- Receives two freshly read input words, the constant 64, the mode twice, and the exponent-destination address.
- Supplies a value consumed as a 64-bit result.

The caller uses the returned high half to derive the exponent-like output. It uses both returned halves for destination replacement only when the mode is nonzero.

The dependency’s mathematical operation is unknown. Scaling or normalization is plausible inferred intent, but neither operation is an established contract. No guarantees about its errors, memory effects, or treatment of special floating-point encodings are available.

## Acceptance examples

These examples specify expected behavior from the supplied static evidence. They are not reports of runtime tests. Except where stated otherwise, storage does not overlap and execution returns normally.

### Nonzero field with zero mode

Given input words 0x89ABCDEF and 0x3FF12345, and mode zero:

- The word destination receives 0x3FF12345 followed by 0x89ABCDEF.
- The exponent destination receives 1023.
- The dependency is not called.

Changing only the mode does not change this path’s behavior.

### Sign exclusion

Given input words 0x01234567 and 0xBFF00000:

- The initial destination becomes 0xBFF00000 followed by 0x01234567.
- The exponent destination receives 1023.
- The dependency is not called.

The top bit of the second input word does not contribute to the extracted exponent field.

### All-ones field

Given a second input word of 0x7FF00000:

- The exponent destination receives 2047.
- The dependency is not called.

No additional classification follows from the first input word’s contents.

### Zero field, nonzero mode

Given input words 0x00000001 and 0x00000000, and mode −1, the dependency is called with those words, 64, −1, the exponent-destination address, and −1.

Conditionally, if it returns high half 0x04123456 and low half 0xAABBCCDD:

- The final word destination is 0x04123456 followed by 0xAABBCCDD.
- The returned exponent field is 65.
- The exponent destination receives 1.

The supplied evidence does not predict that dependency return value; the example specifies how that return would be consumed.

### Zero field, zero mode

For the same input and conditional return, but mode zero:

- The dependency is still called, with zero in both mode positions.
- This function does not copy the returned halves to the word destination.
- Its initial destination writes are 0x00000000 followed by 0x00000001.
- The exponent destination receives 1.

The initial destination values remain final only if the dependency has no intervening effects on that storage.

### Unsigned subtraction

If the dependency returns a high half whose exponent field is zero, the exponent destination receives 0xFFFFFFC0. It does not receive a signed-language value of −64 through this unsigned interface.

### Exact input/destination overlap

Given shared input and word-destination storage initially containing 0x11223344 followed by 0x3FF00000:

- The first write replaces the first word with 0x3FF00000.
- The next input read observes that replacement.
- Both words finish as 0x3FF00000.
- A separate exponent destination receives 1023.

### Exponent/destination overlap

Given the first nonzero-field example, but with the exponent destination pointing to the word destination’s first word:

- The reversed-copy writes occur first.
- The final exponent write replaces the first destination word.
- The final pair is 0x000003FF followed by 0x89ABCDEF.

## Uncertainties

The following are documented_only:

- The semantic meaning and preconditions of F-9f634613286e.
- Its memory ownership, address retention, global-state effects, and failure behavior.
- Whether callers intend a signed interpretation of wrapped exponent results.
- Whether the surrounding API permits overlapping arguments.
- Platform byte order and concrete calling-convention details.
- Behavior under concurrent mutation or asynchronous observation.

Binary64 interpretation and normalization intent are inferences, not established guarantees.

## Evidence

Established static evidence comprises two ordered initial destination assignments, extraction of an eleven-bit field from the retained second input word, one conditional dependency call, mode-controlled replacement writes, unsigned subtraction by 64 in the dependency branch, and a final exponent-destination write.

No dependency implementation, caller contract, runtime trace, or external interface documentation was supplied. No runtime tests have been performed.
