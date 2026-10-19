<!-- Trusted provenance: F-fc2bfe131d26; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fc2bfe131d26

## Purpose

F-fc2bfe131d26 derives an unsigned result from a caller-supplied word and an auxiliary value made available after invoking dependency F-d7be9cfb4000.

The established behavior has three parts: capture the input word, derive a folded value from that captured word, and combine the folded value with a scaled auxiliary value following the dependency invocation.

The arithmetic suggests a compact mixing or reduction operation. That interpretation is inferred intent, not an established functional classification. The supplied evidence does not establish that this function implements a hash, checksum, index calculation, or any particular algorithm.

This contract distinguishes the recovered interface from a conditional 32-bit interpretation. The supplied declaration uses unsigned and signed integer types without providing their definitions or the complete calling convention. The constants and register-oriented result notation are consistent with a 32-bit implementation, but they do not independently settle every width and overflow question.

## Interface

**Input**

The function accepts one pointer to an unsigned integer object.

The pointer must designate an object that can validly be read using the recovered unsigned-word type. The function contains no explicit null-pointer, address-range, alignment, lifetime, or accessibility checks. A pointer that does not meet the execution environment’s requirements has no established error-return behavior.

The input is read by value. Its contents are not interpreted as an address, length, string, aggregate header, or sequence of words.

**Output**

The declared result is an unsigned integer. It combines:

- A folded form of the captured input.
- Thirteen times an auxiliary value available after F-d7be9cfb4000 completes.

The auxiliary value’s acquisition is not represented as an ordinary return-value assignment. Consequently, the ordinary return type of F-d7be9cfb4000, if any, is not sufficient to describe this interface.

**Width profile**

For the conditional 32-bit profile used below:

- The input, folded value, and final result are unsigned 32-bit words.
- Right shifts of those words are logical, introducing zero bits.
- The auxiliary value is represented as a signed 32-bit quantity before multiplication.
- A separate assumption is required to treat overflowing multiplication as low-32-bit machine arithmetic.

The supplied evidence does not contain the type definitions or dependency ABI needed to make all of those properties unconditional. Examples that require the 32-bit profile are labeled accordingly.

**Layout and ownership**

The visible operation accesses one scalar object through the input pointer. It does not visibly access adjacent elements or fields. Byte order is not specified: the function reads a typed integer value rather than explicitly assembling bytes.

No allocation, deallocation, ownership transfer, or retention of the input pointer is visible in F-fc2bfe131d26. Ownership and retention behavior inside the dependency remain unknown.

## Behavior

### Capturing the input

The function first reads the unsigned word designated by the input pointer.

This is the only explicit access through that pointer in the supplied body. Subsequent arithmetic uses the captured value, and the final combination uses the folded value derived from it. There is no explicit second read of the original object after the dependency call.

This distinction matters if the dependency changes the input object through some independently available alias. Such a change is not visibly re-read by this function. Whether the dependency can reach that object is unknown.

### Folding the captured value

The folded value is the bitwise exclusive-or of three terms:

1. The captured input itself.
2. The captured input shifted right by 24 bits.
3. The captured input shifted right by 12 bits, with the shifted result masked by hexadecimal FFF00FFF.

The mask applies only to the third term. It does not mask the whole exclusive-or result.

Under the 32-bit profile, the shifted-by-12 term has no set bits above bit 19. The mask retains bits 0–11 and 20–31, so only bits 0–11 of that shifted term can contribute. The resulting fold has these observable properties:

- Bits 12–31 are unchanged from the captured input.
- Bits 8–11 combine the corresponding original bits with original bits 20–23.
- Bits 0–7 combine the corresponding original bits with original bits 12–19 and 24–31.
- There is no carry propagation between bits during folding.

These properties are specific to the 32-bit profile. A wider unsigned type could leave additional significant bits in the shifted and masked terms.

### Invoking the dependency

After computing the folded value, the function invokes F-d7be9cfb4000 once, with two value arguments in this order:

- The folded value.
- The integer value 19.

The second argument is fixed; it is not derived from the input.

The visible invocation does not pass the input pointer. That limits the explicit argument flow but does not establish isolation. The dependency may have access to shared state, retained pointers, or other resources not shown in the supplied body.

No retry, alternate dependency invocation, or explicit recovery branch is present.

### Forming the result

After the dependency invocation, the function uses an auxiliary value represented by the recovered interface as an additional register output. It multiplies that value by 13 and exclusive-ors the product with the folded value.

Where the signed multiplication is representable in its evaluated type, the final exclusive-or combines the folded unsigned value with the unsigned-converted product.

For a signed 32-bit auxiliary value, multiplication by 13 is representable only when that value lies between −165,191,049 and 165,191,049, inclusive. Outside that range, the supplied expression alone does not establish a portable signed-arithmetic result.

Under an explicitly assumed 32-bit machine-wrap interpretation, the combination is:

- Retain the low 32 bits of thirteen times the auxiliary value.
- Exclusive-or those bits with the folded word.
- Return the resulting unsigned 32-bit word.

Machine wrapping is not established merely because the function’s final result is unsigned. The multiplication precedes the exclusive-or and is represented with a signed operand.

### Completion and exceptional cases

There is no explicit success indicator, failure indicator, sentinel result, or error-reporting branch.

If the dependency does not return normally, the visible path to the final result is not reached. The supplied evidence does not establish whether the dependency can fail, trap, terminate execution, or transfer control elsewhere.

There is likewise no established recovery behavior for an invalid input pointer or for execution conditions outside the applicable arithmetic model.

## Side effects

The directly visible memory effect is one read of the caller-supplied unsigned object.

F-fc2bfe131d26 contains no explicit write through the input pointer and no explicit write to global state. Its full side effects nevertheless include whatever F-d7be9cfb4000 does during its invocation.

The following conclusions are therefore not justified:

- That the input object remains unchanged after the function returns.
- That global state is preserved.
- That the operation is pure, deterministic, reentrant, or thread-safe.
- That no allocation, release, I/O, or synchronization occurs.
- That the dependency cannot retain information or references obtained through other means.

The observable order supported by the body is: input read, folding computation, dependency invocation, auxiliary-value combination, and return.

No atomicity or synchronization guarantee is supplied for the input read. Concurrent mutation must be assessed against the actual object type, language rules, and execution environment.

## Dependencies

### F-d7be9cfb4000

**Established use:** One invocation receives the folded word followed by the fixed value 19.

**Established result consumption:** The enclosing function subsequently consumes an auxiliary register value associated with the post-call state. The evidence does not show an ordinary return-value assignment supplying it.

**Unknown contract:** The dependency’s argument widths, signedness, formal return convention, valid-input domain, memory effects, global-state effects, and failure behavior are not supplied.

In particular, it is not established whether the auxiliary value is:

- A deliberate secondary result.
- A remainder or another arithmetic quantity.
- An ABI-specific output.
- Merely a value left in a register without a documented semantic guarantee.

Those alternatives have materially different implications for a stable behavioral contract. None should be selected solely from the dependency’s arguments or the multiplication that follows it.

A usable integration must establish how F-d7be9cfb4000 supplies the consumed auxiliary value. Its ordinary return value must not be substituted without independent ABI evidence.

## Acceptance examples

These examples are conditional arithmetic checks, not recorded executions. They assume the 32-bit profile, a valid input pointer, normal dependency completion, and the stated auxiliary value.

### Zero input

For input hexadecimal 00000000, the folded value is 00000000. The dependency receives that word and 19.

If the auxiliary value is zero, the final result is 00000000. If it is one, the final result is 0000000D.

### A contribution from the middle input bits

For input hexadecimal 00001000, the folded value is 00001001.

With an auxiliary value of zero, the returned result is 00001001. This distinguishes the required folding from behavior that ignores the input’s bit-12 contribution to the low bits.

### A contribution from the highest byte

For input hexadecimal 01000000, the folded value is 01000001.

With an auxiliary value of zero, the result is 01000001. This checks that the right shift by 24 contributes to the low bits.

### Mask placement

For input hexadecimal 01000000, the shift by 12 produces hexadecimal 00001000 before masking. That contribution is removed by the mask; the shift-by-24 contribution remains.

Accordingly, the folded value is 01000001, not 01001001. This distinguishes masking the third term from retaining all shifted bits.

### Combined folding and scaling

For input hexadecimal 12345678, the folded value is 1234552F.

If the auxiliary value is two, its product with 13 is hexadecimal 0000001A. The final result is 12345535.

The dependency’s first argument remains 1234552F; it is not the final combined result.

### Negative auxiliary value without multiplication overflow

For input zero and auxiliary value −1, the signed product is −13, which is representable in signed 32-bit arithmetic.

Under the 32-bit unsigned-result profile, conversion for the exclusive-or yields hexadecimal FFFFFFF3, so that is the returned result.

### Mutation during the dependency call

Suppose the captured input is 00001000, the dependency subsequently changes the pointed-to object through an independent alias, and its auxiliary value is zero.

Provided normal execution continues and the captured folded value remains the value used by the caller, the final result is 00001001. The changed object is not explicitly re-read.

### Out-of-range auxiliary multiplication

An auxiliary value of 165,191,050 produces a mathematical product greater than the signed 32-bit maximum.

The supplied evidence does not establish an unconditional returned word for that case. A low-32-bit result may be specified only after confirming the machine-arithmetic interpretation.

## Uncertainties

The following items are **documented_only**, not resolved by assumption:

- Exact definitions and widths of the recovered signed and unsigned types.
- The complete ABI of F-d7be9cfb4000.
- The meaning, validity, and signed interpretation of the auxiliary register value.
- Overflow behavior of the multiplication.
- Dependency side effects, including memory ownership and global-state changes.
- Required alignment and access properties of the input object.
- Concurrency guarantees and exceptional control-flow behavior.

The apparent mixing purpose is inferred intent. The visible argument flow, read order, bitwise operations, and final combination are established static evidence.

## Evidence

This contract is based solely on the supplied function body and opaque dependency mapping.

The static evidence establishes one explicit input dereference, the grouping of the shifts and mask, one dependency call with a fixed second argument of 19, and a final exclusive-or involving thirteen times an auxiliary post-call value.

The 32-bit bit relationships and acceptance values are arithmetic deductions under the stated profile. They do not establish the missing type definitions, dependency semantics, or overflow rules.

No runtime tests have been performed. No dependency implementation, ABI documentation, or execution trace was supplied.
