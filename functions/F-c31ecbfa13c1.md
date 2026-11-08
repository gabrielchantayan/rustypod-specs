<!-- Trusted provenance: F-c31ecbfa13c1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c31ecbfa13c1

## Purpose

F-c31ecbfa13c1 obtains two values and submits two output requests. A zero-valued mode argument selects direct submission. Any nonzero mode selects a larger construction that combines the two values with three selected components from each of rows 3, 0, and 1 of an additional input.

The established behavior is the sequence of dependency calls, their argument relationships, and a single explicit context-field read. The supplied body does not establish whether the dependencies compute values immediately, emit deferred operations, or manipulate another representation.

The nonzero path resembles a two-coordinate projective transformation with a shared normalization term. That interpretation is inferred intent, not an unconditional contract: the meanings of operation codes and selected components are not supplied.

## Interface

The function accepts seven positional arguments and has no explicit return value.

| Position | Contract name | Width and established use |
|---|---|---|
| 1 | Request context | Opaque 32-bit value forwarded to value-selection dependencies. |
| 2 | Working context | 32-bit address-like value. A four-byte field is read at byte offset 4. Also forwarded to several dependencies. |
| 3 | Primary input | Opaque 32-bit value supplied when selecting the two primary values. |
| 4 | Output reference | Opaque 32-bit value forwarded to each output request. |
| 5 | Output position | A 32-bit signed integer in the supplied representation. Used unchanged for the first output and increased by 4 for the second. Its unit is not established. |
| 6 | Secondary input / mode | A 32-bit signed integer in the supplied representation. Zero selects the direct path. Every nonzero value selects the constructed path and is also forwarded as the secondary input. |
| 7 | Selection qualifier | Opaque 32-bit value forwarded when selecting each primary value. |

The contract uses the following descriptive labels:

- **First primary value**: the result selected with index 0.
- **Second primary value**: the result selected with index 1.
- **Captured context field**: the four-byte value read from the working context at byte offset 4.
- **Temporary token**: a 32-bit result returned by F-b35e541ebc78.
- **Selected component**: a 32-bit result returned by F-5cd09f83dd02.

These names describe dataflow only. In particular, a temporary token is not established to be a pointer, newly allocated storage, or a unique identifier.

### Preconditions and representation limits

The working context must support the initial four-byte read, including when the mode is zero. No null check, bounds check, alignment check, or context validation is visible.

Each dependency must accept the arguments supplied to it for the caller’s chosen inputs. The body performs no validation of dependency results before reusing them.

The second output position is formed by adding 4 to the supplied output position. For a contract that does not depend on unresolved signed-overflow behavior, the caller must supply a position for which this addition is representable in a signed 32-bit integer. The supplied representation alone does not establish the behavior outside that range.

The loop uses only row indices 0 and 1. Its index multiplication therefore produces offsets 0 and 4, without an arithmetic overflow in those intermediate values.

## Behavior

### Common entry behavior

Before choosing either path, the function reads the captured context field from the working context at byte offset 4.

This read occurs even on the direct path, where its result is not subsequently used. A caller therefore cannot rely on zero mode avoiding that memory access.

On either path, the two primary selections occur in this order:

1. F-3976e291be48 receives the request context, working context, primary input, index 0, and selection qualifier.
2. F-3976e291be48 receives the same arguments except that the index is 1.

Both results are retained before any output request occurs. They are reused without another primary-selection call.

The initial context-field read precedes both selections. On the constructed path, all temporary requests use that previously captured field value. Changes to the working context’s field during dependency calls do not cause this function to reload it.

### Zero mode: direct submission

When the sixth argument is exactly zero, the function makes two calls to F-aa543ac18986:

- First: working context, output reference, original output position, first primary value.
- Second: working context, output reference, output position plus 4, second primary value.

The first output request completes before the second begins.

This path makes no calls to the temporary-token, secondary-component, unary-operation, or ternary-operation dependencies. It does not inspect the primary values or transform them locally.

On normal completion, the path performs four dependency calls: two selections followed by two output requests.

### Nonzero mode: shared construction followed by two row constructions

Every nonzero sixth argument takes the same branch, including negative values. The argument is not converted to a Boolean before being passed to the secondary-component selector.

The constructed path has three ordered stages:

1. Select the two primary values.
2. Construct a shared token using row 3 of the secondary input.
3. Construct and submit one result from row 0, then one result from row 1.

The numerical operation codes below are part of the established interface. Their arithmetic meanings are not established.

#### Shared construction using row 3

After selecting the primary values, the function requests five temporary tokens from F-b35e541ebc78. Each request receives the captured context field and a zero second argument.

The five return values have these subsequent roles, in request order:

1. Shared-result destination.
2. First intermediate destination.
3. Second intermediate destination.
4. Third intermediate destination.
5. Fourth intermediate destination.

These roles do not imply that the returned values differ.

The function then selects row 3, component 0, and row 3, component 1, in that order, using F-5cd09f83dd02. Each selection receives the request context, working context, and original nonzero secondary input.

The remaining shared-construction calls have this exact order and argument relationship:

| Order | Dependency | Operation and operands |
|---|---|---|
| 1 | F-915779408ccb | Code `0x14`; first intermediate destination; row-3 component 0; first primary value. |
| 2 | F-5cd09f83dd02 | Select row 3, component 3. |
| 3 | F-915779408ccb | Code `0x14`; second intermediate destination; row-3 component 1; second primary value. |
| 4 | F-915779408ccb | Code `0x11`; third intermediate destination; first and second intermediate destinations. |
| 5 | F-915779408ccb | Code `0x11`; fourth intermediate destination; third intermediate destination and row-3 component 3. |
| 6 | F-489dcd301bad | Code `0x17`; shared-result destination; fourth intermediate destination. |

The row-3 component-3 selection is specifically interleaved between the first and second `0x14` calls. It does not occur alongside the first two component selections.

The shared-result destination token is retained for both following row constructions. This function does not read a result back from its destination or inspect operation status.

#### Row constructions and output order

Rows 0 and 1 are processed in ascending order. Row 0 is fully constructed and submitted before the function begins requesting row-1 temporary tokens.

For each row, five temporary-token requests occur first, all using the captured context field and zero. Their roles, in request order, are:

1. First intermediate destination.
2. Second intermediate destination.
3. Third intermediate destination.
4. Fourth intermediate destination.
5. Output-value destination.

The function next selects that row’s component 0 and component 1, in that order.

It then performs the following ordered operations:

| Order | Dependency | Operation and operands |
|---|---|---|
| 1 | F-915779408ccb | Code `0x14`; first intermediate destination; selected component 0; first primary value. |
| 2 | F-5cd09f83dd02 | Select the current row’s component 3. |
| 3 | F-915779408ccb | Code `0x14`; second intermediate destination; selected component 1; second primary value. |
| 4 | F-915779408ccb | Code `0x11`; third intermediate destination; first and second intermediate destinations. |
| 5 | F-915779408ccb | Code `0x11`; fourth intermediate destination; third intermediate destination and selected component 3. |
| 6 | F-915779408ccb | Code `0x14`; output-value destination; fourth intermediate destination; shared-result destination. |
| 7 | F-aa543ac18986 | Working context, output reference, row-specific output position, output-value destination. |

The row-specific output position is the original position for row 0 and the original position plus 4 for row 1.

No component-2 selection occurs. No row-2 selection occurs. The function does not generate an output for row 3; that row supplies the shared construction.

### Completion and failure handling

The function has no explicit status result. It does not test for failed selections, failed temporary requests, invalid operation results, or failed output requests.

There is no visible retry, rollback, release, or cleanup path. If a dependency returns normally with an unusual token, that token is passed onward according to the same dataflow. Zero-valued dependency results receive no special treatment.

The supplied body does not establish what happens if a dependency fails to return normally. The normal-completion sequence must not be interpreted as a guarantee of atomicity.

## Side effects

The only explicit memory access in this function’s body is the initial four-byte read at working-context offset 4. All subsequent externally visible work is delegated.

In particular:

- F-aa543ac18986 receives two ordered output requests. Their actual memory effects, write widths, and interpretation of the output position are unknown.
- F-b35e541ebc78 may affect allocation or construction state, but allocation is not established by this body.
- The operation dependencies may mutate the working context or other state.
- Selection dependencies may have side effects; they must not be treated as pure merely because their results are used as values.
- No dependency is assumed to preserve global state.
- No ownership transfer, retention policy, or release obligation is established by these calls alone.

If the output reference aliases the working context or another input, this function provides no alias rejection or protective copy. Its visible ordering still applies: the context field and primary values are captured before the first output request, while row-1 component selections occur after the row-0 output request. Whether aliasing changes later selections depends on the dependencies’ actual behavior.

The function itself makes no explicit release request for any returned token. That observation does not establish either a leak or ownership transfer.

## Dependencies

All identifiers below are opaque.

| Dependency | Established role and call interface |
|---|---|
| F-3976e291be48 | Returns a 32-bit primary value from request context, working context, primary input, selection index, and selection qualifier. Called with indices 0 then 1. |
| F-b35e541ebc78 | Returns a 32-bit temporary token from the captured context field and zero. Called only on the nonzero path. |
| F-5cd09f83dd02 | Returns a 32-bit selected component from request context, working context, secondary input, row index, and component index. |
| F-915779408ccb | Receives working context, an operation code, a destination token, and two operand tokens. Codes used are `0x14` and `0x11`. Any return value is ignored. |
| F-489dcd301bad | Receives working context, code `0x17`, a destination token, and one operand token. Any return value is ignored. |
| F-aa543ac18986 | Receives working context, output reference, output position, and value token. Any return value is ignored. |

For a normally completed nonzero invocation, the call counts are:

- 2 primary selections.
- 15 temporary-token requests.
- 9 secondary-component selections.
- 14 two-operand operation requests: 8 using `0x14` and 6 using `0x11`.
- 1 one-operand operation request using `0x17`.
- 2 output requests.

That is 43 dependency calls in total. These counts describe this function’s direct calls, not work performed internally by dependencies.

## Acceptance examples

These are conditional acceptance scenarios derived from the supplied static body. They have not been executed.

### Direct mode preserves selection and submission order

Given a readable working context, zero mode, and primary selections yielding distinguishable tokens, an acceptable trace has:

- The context-field read before either selection.
- Selection indices 0 and 1 in that order.
- The index-0 token submitted at the original position.
- The index-1 token submitted at the position increased by 4.
- No temporary-token or operation requests.

It is not acceptable to omit the initial read merely because its value is unused on this path.

### Nonzero values are secondary inputs, not just truth values

Given a sixth argument of −1, the constructed path is selected. Every secondary-component selection receives −1 as its secondary-input argument. Replacing that argument with 1 would change established behavior.

### Row and component ordering is observable

Given dependencies whose effects can be observed, the secondary-component selection order is:

- Row 3: components 0, 1, 3.
- Row 0: components 0, 1, 3.
- Row 1: components 0, 1, 3.

Within each row, the component-3 selection follows the first `0x14` operation request and precedes the second. Moving all three selections ahead of operation requests would not preserve the established trace.

### The captured field is reused

Given that a dependency changes the working-context field at offset 4 after the initial read, every temporary-token request still receives the originally captured value. A later reload would violate the established behavior.

### Output is sequential, not transactional

Given that the first output request changes state observed by later dependencies, row-1 selection and construction occur after that change. The function does not prepare both output values before submitting the first.

### Arithmetic interpretation is conditional

If external dependency contracts establish that:

- `0x14` multiplies represented values,
- `0x11` adds represented values,
- `0x17` forms a reciprocal,
- selected components are numeric coefficients,

then the shared construction represents the reciprocal of the row-3 combination of components 0, 1, and 3 with the two primary values. Each output represents the corresponding row-0 or row-1 combination multiplied by that shared reciprocal.

This interpretation is not an unconditional acceptance requirement without those dependency contracts. In particular, no zero-denominator behavior is established here.

## Uncertainties

The following items are **documented_only**:

- The actual types and meanings of all opaque values.
- The byte order and alignment requirements of the context-field read.
- Whether temporary requests allocate, reuse, intern, or otherwise obtain tokens.
- Whether destination and operand tokens may alias safely.
- The meanings of operation codes `0x11`, `0x14`, and `0x17`.
- Numeric widths, signedness, rounding, overflow, and exceptional-value behavior inside operations.
- The interpretation and unit of the output position.
- Actual output write widths and destination layout.
- Dependency failure signaling and behavior after partial progress.
- Global-state mutations, reentrancy, thread safety, and ownership rules.
- Behavior when output-position addition exceeds the signed 32-bit range.

Inferred projective-transform intent does not resolve any of these unknowns.

## Evidence

The supplied static body establishes the seven-argument interface, initial four-byte context read, zero/nonzero branch, dependency arguments, call ordering, two row iterations, numerical operation codes, and absence of local validation or explicit cleanup.

The supplied dependency map establishes only the opaque identities used here. Dependency implementations and independent interface contracts were not provided.

The mathematical interpretation is inferred from the repeated dataflow pattern and operation-code usage; it is not established arithmetic behavior.

No runtime tests have been performed.
