<!-- Trusted provenance: F-49909d5fffbf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-49909d5fffbf

## Purpose

F-49909d5fffbf describes a partially recovered operation that performs two byte stores, interleaves them with twelve indirect calls, and reaches a boundary at which subsequent execution is not established.

The visible behavior is not sufficient to identify a higher-level purpose such as device initialization, message dispatch, or object mutation. Such interpretations are not part of this contract. In particular, the small numerical addresses used by the byte stores must not be silently reinterpreted as offsets within an ordinary allocated object.

This is a source-facing, static behavioral contract. It distinguishes behavior represented in the supplied material from machine-level details that the material does not establish. The recovery contains an explicit bad-instruction warning and an overlapping-global warning. Consequently, the contract does not assert that the entire represented sequence is executable or that the final recovery boundary is an intentional runtime termination.

## Interface

### Explicit inputs

The represented interface has three arguments:

| Input | Represented width | Observable use |
|---|---:|---|
| First argument | 32-bit unsigned | Supplies the first stored byte, participates in the later absolute-address calculation, and supplies the second stored byte through a logical right shift. |
| Second argument | 32 bits | No explicit use is visible. |
| Third argument | 32 bits | Forwarded as the third argument of the first indirect call. No other explicit use is visible. |

The third argument is treated as a 32-bit payload. Its interpretation as an integer, pointer, flags, or another representation is unknown. The second argument's lack of visible use does not prove that its incoming physical register or stack slot is unobservable to callees under the actual calling convention.

### Additional entry state

The operation also depends on state not represented as explicit arguments:

- An incoming register-held base used for a byte read at a displacement of one.
- Incoming link-register state used to calculate the first call target.
- A global value read before the first call and shifted left by ten.
- Memory at a later address derived from the first argument.
- The behavior of twelve indirect call sites.

These are genuine prerequisites of the represented behavior, not optional configuration.

### Outputs

There is no explicit return value in the recovered interface. No normal return is visible.

Two intermediate call results are used:

- A 32-bit result from F-DEP-02 is passed to F-DEP-03.
- A result represented as a 32-bit integer from F-DEP-07 is shifted left by two and passed to F-DEP-08.

No status code, success indicator, or error result is established for F-49909d5fffbf itself.

## Behavior

### Interpretation and ordering

The following sequence describes explicit accesses and calls in the supplied representation. Every later event is conditional on earlier accesses succeeding and earlier calls returning.

Calls may perform additional reads, writes, transfers of control, or termination. Those effects are unknown. The stated order therefore describes this function's visible operations, not a claim that nothing else happens between them.

Byte means eight bits. The represented address and arithmetic model is 32-bit. Where a source expression would require machine-level confirmation to determine overflow or shift behavior, that uncertainty is stated rather than resolved by assumption.

### Initial global read and byte selection

Before the first indirect call, the function reads the global value and forms a value shifted left by ten. This becomes the fourth explicit argument to F-DEP-01.

The representation suggests a 32-bit result, but does not reliably establish the global object's declared width, signedness, or relationship to overlapping objects. A consumer must not assume an isolated, ordinary 32-bit global merely because the result is held in a 32-bit temporary.

Next, the function reads one byte from the incoming register-held base plus one. The byte is zero-extended, producing a selector in the inclusive range 0 through 255.

That selector is used directly as an address component. The function writes the low eight bits of the first argument to address:

\[
17 + \text{selector}
\]

The destination is therefore numerically between 17 and 272 inclusive. There is no visible addition of another object base, range validation, or null check.

This byte store occurs before F-DEP-01.

### First call and initial call chain

F-DEP-01 is invoked with four explicit arguments, in this order:

1. The zero-extended selector.
2. The same zero-extended selector.
3. The third entry argument.
4. The previously formed global-derived value shifted left by ten.

The first target is calculated from incoming link-register state plus a fixed displacement of 4,080 bytes. This target calculation depends on entry state; it is not represented as a lookup through a global function-pointer table.

The return value of F-DEP-01 is not explicitly consumed.

The next calls occur in this order:

1. F-DEP-02, with no explicit arguments shown.
2. F-DEP-03, receiving the 32-bit result of F-DEP-02.
3. F-DEP-04, with no explicit arguments shown.
4. F-DEP-05, with no explicit arguments shown.
5. F-DEP-06, with no explicit arguments shown.
6. F-DEP-07, with no explicit arguments shown.

The result of F-DEP-07 is retained for a later call.

“No explicit arguments shown” does not establish an empty machine-level interface. These calls may depend on registers, flags, memory, or other implicit state that the supplied representation does not expose.

### Second byte read and store

After F-DEP-07 returns, and before F-DEP-08 is invoked, the function performs another byte-based selection and store.

It calculates an absolute read address from the first argument:

\[
2 \times \text{first argument} + 1
\]

Under the represented 32-bit unsigned arithmetic, multiplication and addition are reduced modulo \(2^{32}\). The address is not relative to the incoming register-held base used for the first byte read.

The function reads one byte at this calculated address and zero-extends it. It then writes one byte to address:

\[
17 + \text{new selector}
\]

Again, this destination lies numerically between 17 and 272 inclusive.

The value stored is the low eight bits of the first argument after a logical right shift by 22. Equivalently, the stored value consists of original argument bits 22 through 29. Original bits 30 and 31 do not survive the final truncation to one byte.

This second selector read is performed after all seven preceding calls. It must not be treated as an entry-time snapshot: a preceding callee could have changed the memory from which it is read.

The second store may target the same byte as the first store. No special handling for that case is visible.

### Remaining calls

After the second store, F-DEP-08 is invoked with the retained result of F-DEP-07 shifted left by two.

The recovered result is represented as a signed 32-bit integer. The supplied material does not establish enough machine semantics to prescribe the result for every signed input. In particular, negative values and values whose left shift exceeds the signed range require confirmation. This contract does not silently substitute unsigned wraparound semantics for that expression.

The function then invokes, in order:

1. F-DEP-09.
2. F-DEP-10.
3. F-DEP-11.
4. F-DEP-12.

No explicit arguments are shown for these four calls, and their results are not explicitly consumed.

### End of the established sequence

After F-DEP-12, the recovered control flow reaches a bad-instruction boundary.

The evidence does not establish whether actual execution would encounter an invalid instruction, continue into unrecovered instructions, or follow another control-flow interpretation. There is no supported normal-return behavior beyond this point.

## Side effects

### Explicit memory accesses

The visible access order is:

1. Read the global value.
2. Read one byte from the incoming register-held base plus one.
3. Write one byte at the first selector-derived destination.
4. Execute F-DEP-01 through F-DEP-07 in order.
5. Read one byte at the absolute address derived from the first argument.
6. Write one byte at the second selector-derived destination.
7. Execute F-DEP-08 through F-DEP-12 in order.
8. Reach the unresolved control-flow boundary.

The two stores affect one byte each. No wider store, allocation, deallocation, locking, transaction, or rollback is explicitly represented.

If both stores have the same destination, the second store replaces the byte left there by the first store or by intervening callees. If an intervening call does not return, the second store is not reached.

### Ownership and lifetime

No ownership transfer is established.

The first argument participates in address arithmetic, but is not visibly allocated or released. The third argument is forwarded without an established ownership convention. Passing it to F-DEP-01 does not prove that the callee borrows, retains, consumes, or frees it.

Likewise, no preservation guarantee exists for globals or memory across the dependency calls. Their behavior must remain opaque until independently specified.

### Preconditions and failures

For the entire visible sequence to complete:

- Both explicit byte-read addresses must be readable when accessed.
- Both selector-derived destinations must be writable when accessed.
- The initial global read must be valid.
- Every reached indirect target must be callable under the actual execution environment.
- Each preceding call must return to permit later operations.

There is no visible validation or recovery for failed memory access, invalid call targets, or dependency failure. The exact fault, exception, or termination behavior belongs to the execution environment and is unknown here.

## Dependencies

The following are opaque, document-local function IDs for call sites. They do not assert independently verified callee identities. Distinct IDs distinguish invocation positions; they do not prove that the runtime targets cannot alias.

| Function ID | Explicit call contract visible here |
|---|---|
| F-DEP-01 | Entry-state-derived target; receives selector, duplicate selector, third argument, and shifted global-derived value. Result unused. |
| F-DEP-02 | Fixed-expression target; no explicit arguments shown. Produces a 32-bit value used by F-DEP-03. |
| F-DEP-03 | Fixed-expression target; receives F-DEP-02's result. Result unused. |
| F-DEP-04 | Target derived from a code-location reference and fixed displacement; no explicit arguments shown. |
| F-DEP-05 | Fixed-expression target; no explicit arguments shown. |
| F-DEP-06 | Fixed-expression target; no explicit arguments shown. |
| F-DEP-07 | Fixed-expression target; no explicit arguments shown. Produces the integer used by F-DEP-08. |
| F-DEP-08 | Fixed-expression target; receives F-DEP-07's result shifted left by two. |
| F-DEP-09 | Fixed-expression target; no explicit arguments shown. |
| F-DEP-10 | Fixed-expression target; no explicit arguments shown. |
| F-DEP-11 | Fixed-expression target; no explicit arguments shown. |
| F-DEP-12 | Fixed-expression target; no explicit arguments shown. Last visible call. |

Apart from the first target's entry-state dependency, the targets are represented using fixed code-location expressions. Their concrete identities, valid entry modes, calling conventions, side effects, and return behavior remain unknown.

## Acceptance examples

These are conditional static acceptance examples, not results of executed tests. Each assumes valid required accesses and returning callees unless otherwise stated.

### First-store truncation

If the first argument is 291 and the initial selector byte is 5:

- The first store targets address 22.
- The stored byte is 35, the low eight bits of 291.
- F-DEP-01 receives 5 as each of its first two explicit arguments.

The store must precede that call.

### Independent second lookup

If the first argument is 3:

- The later selector read uses address 7.
- It does not use the incoming register-held base.
- The second stored value is zero because shifting 3 right by 22 yields zero.

If the byte read at address 7 is 200, the second store targets address 217.

### Upper-bit truncation

If the first argument is hexadecimal C0400000:

- Logical right shift by 22 produces hexadecimal 301.
- The second stored byte is hexadecimal 01.
- The second selector-read address is hexadecimal 80800001 under 32-bit unsigned arithmetic.

This example distinguishes retaining eight result bits from retaining all ten bits produced by the shift.

### Address wraparound

If the first argument is hexadecimal 80000000, the later selector-read address is 1 under the represented 32-bit unsigned arithmetic.

The calculation does not create a wider address merely because the mathematical product exceeds 32 bits.

### Same destination

If both selector reads produce 9, both stores target address 26. The second store occurs after F-DEP-07 and before F-DEP-08. No deduplication or skipped write is represented.

### Intermediate result forwarding

If F-DEP-02 returns hexadecimal 12345678, F-DEP-03 receives that same 32-bit value.

If F-DEP-07 returns 7, F-DEP-08 receives 28, after the second byte store.

### Dependency does not return

If F-DEP-04 does not return, the established prefix includes the first store and calls through F-DEP-04. The second selector read, second store, and remaining calls are not reached through the visible sequence.

## Uncertainties

The following are **documented_only**:

- The actual higher-level purpose.
- The complete execution architecture and calling convention.
- The source, validity, and preservation rules of hidden incoming register state.
- The global object's width, signedness, overlapping storage, and shift semantics.
- Signed left-shift behavior for F-DEP-07 results outside the unambiguous nonnegative representable range.
- The identities and full contracts of all dependency targets.
- Whether low numerical destinations are mapped memory, device registers, aliases, or invalid addresses.
- Whether explicit memory accesses have volatile, atomic, or device-specific properties.
- Any ownership or lifetime obligations imposed by callees.
- Behavior beyond the bad-instruction recovery boundary.
- Whether the supplied recovery faithfully represents executable instructions throughout.

An apparent sequence of small-address byte writes may suggest specialized memory or hardware behavior. That is inferred intent only, not an established requirement.

## Evidence

Established static evidence consists of the supplied recovered function representation: three explicit 32-bit inputs, two byte reads, two byte stores, twelve ordered indirect calls, two consumed intermediate call results, hidden entry-state dependencies, and an unresolved final control-flow boundary.

The supplied warnings reduce confidence in complete control-flow recovery and in the interpretation of the global object. They do not independently establish a particular runtime fault or memory layout.

No runtime tests, emulation, dependency inspection, or machine-instruction validation have been performed. The acceptance examples are arithmetic and ordering consequences of the represented operations, conditional on the assumptions stated above.
