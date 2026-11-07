<!-- Trusted provenance: F-e31cbdd7772d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e31cbdd7772d

## Purpose

F-e31cbdd7772d translates two unsigned integer inputs into compact selectors and, if both translations succeed, writes the selectors into adjacent character-sized locations in a state object obtained from F-7fba0909c452.

The two inputs follow different classification rules. Configuration matching, fixed values, signed ordering, and a derived configuration value have different precedence for each input.

If either input is rejected, neither selector is written. Instead, the function reads the state's leading integer and replaces zero with `0x500`, preserving any nonzero value. If state acquisition returns null, the function returns before its own configuration or state accesses.

The supplied representation establishes this control flow. It does **not** declare the shared configuration's type. Consequently, configuration arithmetic, comparison conversions, and the derived value cannot all be assigned unconditional 32-bit unsigned semantics.

## Interface

### Inputs and result

The function receives two unsigned integer inputs, referred to here as the **first input** and **second input**. It has no explicit return value.

The representation distinguishes:

- Unsigned integer inputs and selector candidates.
- Explicit conversions to signed integer for ordering comparisons.
- An integer at the beginning of the state object.
- Character-sized selector stores.

The exact widths of these types are not independently established by the excerpt.

For concrete numerical examples, this contract uses an explicitly conditional **32-bit unsigned-configuration model**:

- Unsigned and signed integers are 32 bits.
- Signed integers use two's-complement interpretation.
- Conversion from an unsigned input to signed integer preserves its bit pattern.
- The configuration has unsigned 32-bit integer type.
- Right shift of a negative signed integer is arithmetic.
- A character-sized storage unit is eight bits.
- The state address calculations identify the stated offsets correctly.

These assumptions support the examples; they are not all established declarations or target guarantees.

### State acquisition and layout

F-7fba0909c452 is called once, without explicit arguments. Its result is treated as a pointer to an object with these accessed locations:

| Location | Access | Established use |
|---|---|---|
| Base address | Integer read; conditional integer write | Preserve an existing nonzero value, or record `0x500` |
| Base plus `0x9D1` | Character-sized write | First selector |
| Base plus `0x9D2` | Character-sized write | Second selector |

Under the eight-bit-character model, the selector offsets are byte offsets 2513 and 2514.

The representation calculates the selector addresses through a signed integer conversion of the base pointer. Correct correspondence to the intended object offsets therefore depends on the target's pointer/integer conversion and address-width behavior. It is not a portable guarantee for arbitrary pointer widths.

A non-null result must support the accesses reached:

- Rejection requires a readable leading integer, writable when its value is zero.
- Successful translation requires writable selector locations.
- The leading integer is not directly accessed on success.

The function does not validate the allocation's size, lifetime, alignment, or permissions.

### Configuration terminology and arithmetic

Let **G** denote the shared configuration value observed at a particular reference. Different references need not observe the same value.

A comparison described as “matches configuration” means equality after the operand conversions applicable to the configuration's actual declared type and the unsigned input. It does not unconditionally mean equality between two unsigned 32-bit words.

Similarly, “matches configuration plus three” means:

1. Evaluate the configuration-plus-three expression using the configuration's declared type, applicable integer promotions, and arithmetic conversions.
2. Compare the resulting value with the unsigned input using the conversions applicable to that equality comparison.

The unsigned input in the comparison does **not** establish that the preceding addition is unsigned.

Relevant possibilities include:

- Under the 32-bit unsigned-configuration model, addition wraps modulo \(2^{32}\).
- If addition occurs in a signed integer type, an unrepresentable sum is signed overflow, not established modulo arithmetic.
- A narrower configuration type may be promoted before addition.
- A wider configuration type may change both the addition width and the comparison conversions.

The missing declaration prevents selecting one of these interpretations unconditionally.

The function also prepares an unsigned integer **derived selector H**. Its expression combines one configuration reference with another configuration reference converted to signed integer and shifted right by eight positions, using bitwise OR. The result is then assigned to an unsigned integer.

Under stable configuration and the 32-bit unsigned-configuration model:

\[
H = G \mathbin{\mathrm{OR}} \operatorname{ASR}_{32}(G,8)
\]

Outside that model, the signed conversion, shift behavior, OR operand conversions, and final unsigned conversion must follow the actual types and target rules.

## Behavior

### Acquisition and preparation

The dependency call precedes the function's configuration reads and input classification.

If the returned pointer is null, the function returns immediately:

- Neither input is classified.
- H is not computed.
- The function performs no direct state access.

The dependency's own effects are not constrained by this early return.

For a non-null pointer, the first candidate initially corresponds to the first input, except that first input `0x304` sets the candidate to `6`. The function then computes H.

H is computed even when the first input is `0x304`, `0`, or `1`, and even when classification will subsequently reject an input. Preparation alone does not write a selector.

### First-input classification

First input `0x304` selects `6` and bypasses all remaining first-input classification.

For other inputs, the function converts the first input to signed integer and compares it with `0x305`.

#### Signed-converted input below `0x305`

The following precedence applies:

| Condition | First selector |
|---|---:|
| Input is `0` | `0` |
| Input is `1` | `1` |
| Input matches configuration | `4` |
| Input equals H | `5` |
| No preceding condition holds | Reject |

Thus `0` and `1` take precedence over configuration equality for the first input.

Under the stated 32-bit signed-conversion model, inputs from `0x80000000` through `0xFFFFFFFF` enter this branch. They may succeed by matching configuration or H; their large unsigned magnitude alone does not cause rejection.

Without that conversion model, the branch taken for an input not representable as signed integer remains target-dependent.

#### Signed-converted input at least `0x305`

The following precedence applies:

| Condition | First selector |
|---|---:|
| Input matches configuration plus three | `7` |
| Input is `0x306` | `2` |
| Input is `0x307` | `3` |
| Input is `0x308` | `8` |
| No preceding condition holds | Reject |

The configuration-plus-three comparison precedes all three fixed translations. Its expression is evaluated on this branch before those fixed cases can be selected. If that expression incurs signed overflow under the actual configuration type, the excerpt does not establish a defined fixed-value fallback.

A first-input rejection immediately enters the rejection path. The second input is not classified.

### Second-input classification

Second-input classification begins only after the first input has been accepted.

First, the function compares the second input with a fresh configuration reference. A match selects `4`, without further second-input classification.

If that equality fails, the function makes a signed ordering comparison between:

- A further configuration reference converted to signed integer.
- The second input converted to signed integer.

The configuration used for ordering need not be the value used for the preceding equality comparison.

#### Signed-converted configuration below signed-converted input

The following precedence applies:

| Condition | Second selector |
|---|---:|
| Input equals H | `5` |
| Input is `0x304` | `6` |
| Input matches configuration plus three | `7` |
| No preceding condition holds | Reject |

H takes precedence over `0x304`. Both precede configuration-plus-three matching.

The addition is reached only if the H and `0x304` checks fail. Its arithmetic semantics remain dependent on the missing configuration declaration.

#### Signed-converted configuration not below signed-converted input

After the initial configuration-equality check has failed, the following rules apply:

| Condition | Second selector |
|---|---:|
| Input is `0` | `0` |
| Input is `1` | `1` |
| Input is `0x300` | `2` |
| Input is `0x301` | `3` |
| No preceding condition holds | Reject |

Second inputs `0` and `1` are therefore not unconditional pass-through values. Configuration equality may select `4` first, or signed ordering may send them to the other branch.

The fixed second-input mappings for `0x300` and `0x301` are likewise available only in this branch.

### Successful commit

After both translations succeed, the function performs two ordered stores:

1. First selector to the location at offset `0x9D1`.
2. Second selector to the location at offset `0x9D2`.

Every successfully selected value is between `0` and `8`, inclusive, and is representable in the character-sized destination.

The function does not directly inspect or clear the leading status integer on this path. An existing nonzero status does not prevent selector writes.

The two stores are not established as an atomic pair. No rollback is shown if the first store completes but the second cannot.

### Rejection

Either classification failure reaches the same path:

1. Read the leading state integer.
2. If it is zero, write `0x500` there.
3. Otherwise leave that integer unchanged.
4. Return without writing either selector location.

A successfully translated first candidate remains uncommitted if second-input classification fails.

This is a conditional error-recording operation, not an unconditional assignment and not a success-time status reset.

## Side effects

The function's explicit effects consist of:

- One call to F-7fba0909c452.
- Configuration references during H preparation and reached classification conditions.
- A leading-integer read on rejection.
- A conditional leading-integer write of `0x500`.
- Two ordered character-sized writes on success.

The control-flow ordering is established. The representation does not establish an exact physical load count for the configuration, nor an order between the two configuration references within H's expression.

Important boundaries are:

- Null acquisition prevents all direct configuration and state accesses.
- H preparation precedes either possible classification rejection.
- First-input rejection skips second-input classification.
- Second-input rejection precedes both selector stores.
- Success performs no explicit leading-status access.
- Rejection performs no explicit selector store.

These are explicit-access claims, not guarantees that every other memory location remains unchanged. The dependency may modify state or configuration. The excerpt also provides no non-aliasing or concurrency guarantee.

There is no explicit allocation, release, synchronization, retry, diagnostic output, or recovery mechanism in this function. Absence of a release does not establish ownership or prove that releasing the returned object is unnecessary elsewhere.

## Dependencies

### F-7fba0909c452

**Observed interface:** One call without explicit arguments; the result is interpreted as the state pointer.

**Caller behavior:**

- Null produces immediate return.
- A non-null result is retained for later accesses.
- No further dependency call or explicit release appears.

**Unknown behavior:**

- How the state object is selected or obtained.
- Whether it is borrowed, shared, thread-local, or newly allocated.
- Its lifetime and ownership requirements.
- Whether acquisition changes configuration or existing state.
- Whether it records an error when returning null.
- Whether it can terminate abnormally rather than return.

All configuration behavior described here concerns references made after this call. Configuration preservation across the dependency must not be assumed.

## Acceptance examples

These are conditional deductions from static control flow, not executed tests.

Except for the null case, assume valid state storage, ordinary nonoverlapping locations, stable configuration, and the complete 32-bit unsigned-configuration model defined above. In particular, examples involving wrapped addition depend on that configuration-type assumption.

### Conventional configuration

For configuration `0x302`, H is `0x303`.

| First input | First selector | Second input | Second selector |
|---|---:|---|---:|
| `0` | `0` | `0` | `0` |
| `1` | `1` | `1` | `1` |
| `0x302` | `4` | `0x300` | `2` |
| `0x303` | `5` | `0x301` | `3` |
| `0x304` | `6` | `0x302` | `4` |
| `0x305` | `7` | `0x303` | `5` |
| `0x306` | `2` | `0x304` | `6` |
| `0x307` | `3` | `0x305` | `7` |
| `0x308` | `8` | — | — |

The columns list independent mappings, not required input pairings.

- Inputs `(0x306, 0x300)` commit selectors `(2, 2)`, in first-then-second order.
- Inputs `(0x308, 0x306)` reject the second input and write neither selector.
- Inputs `(0x300, 0)` reject the first input without classifying the second.

### Error preservation and successful writes

With configuration `0x302` and inputs `(0x300, 0)`:

- An initial leading integer of zero becomes `0x500`.
- An initial leading integer of `0x1234` remains `0x1234`.
- Neither case writes a selector.

With the same configuration and inputs `(0x304, 0x301)`, the selectors become `(6, 3)`. An existing leading error is neither directly read nor cleared.

### Different configuration-match precedence

With configuration zero, H is zero. Inputs `(0, 0)` commit `(0, 4)`:

- The first input's zero check wins over configuration matching.
- The second input's configuration match wins over its zero handling.

### Configuration-plus-three precedence

With configuration `0x303`, inputs `(0x306, 0)` commit `(7, 0)`. Configuration plus three matches the first input before the fixed mapping to `2`.

With configuration `0x304`, H is `0x307`. Inputs `(0x304, 0x307)` commit `(6, 5)`: the second input matches H before the configuration-plus-three check could select `7`.

### Signed ordering and conditional wraparound

Under the unsigned-configuration model, configuration `0xFFFFFFFF` yields H equal to `0xFFFFFFFF` and configuration plus three equal to `2`.

- Inputs `(0xFFFFFFFF, 0)` reject the second input. Its signed-converted value exceeds configuration's signed-converted value of `-1`, but it matches none of H, `0x304`, or `2`.
- Inputs `(0, 2)` commit `(0, 7)`.

The wrapped-addition conclusion is not an unconditional requirement for an undeclared configuration type. If addition instead occurs in a signed type, its result or definedness must be established from that type and value.

### Null acquisition

If F-7fba0909c452 returns null, any inputs produce immediate return without this function's configuration reads, status access, or selector stores.

## Uncertainties

### Established static evidence

The excerpt establishes the null gate, classification precedence, explicit signed conversions, H preparation, conditional status write, and ordered selector stores.

It shows configuration-plus-three expressions but does **not** establish their arithmetic type. Explicit signed ordering also does not establish that all other operations use signed or unsigned 32-bit arithmetic.

### Inferred intent

The selector values plausibly encode two related modes, and the leading integer plausibly preserves an outstanding error. These interpretations are not required behavior. The concrete mappings and access sequence are the contract.

### documented_only

The following remain unresolved:

- The shared configuration's declaration, signedness, width, and qualifiers.
- Integer promotions and arithmetic conversions involving that declaration.
- Whether configuration-plus-three uses unsigned wrapping arithmetic, signed arithmetic, or a wider type.
- Defined behavior if a reached addition overflows a signed type.
- Exact integer, pointer, and character widths.
- Out-of-range unsigned-to-signed conversion and negative signed-right-shift behavior.
- Physical configuration load count and ordering within unordered expression operands.
- Configuration stability, synchronization, and possible aliasing with state.
- Pointer/integer conversion validity for the selector address calculations.
- State ownership, lifetime, full layout, and dependency side effects.
- Semantic meanings of the selectors and `0x500`.
- Outcomes after invalid memory access or abnormal dependency exit.

These unknowns do not justify assuming a stable global, unsigned addition, or an ownership policy.

## Evidence

This contract uses only the supplied static representation of F-e31cbdd7772d and the opaque dependency mapping.

The representation supports the stated control flow and explicit accesses. The configuration declaration is absent, so unsigned 32-bit configuration arithmetic is presented only as a conditional model, including in the wraparound examples.

No runtime tests, implementation execution, binary-level verification, or independent ABI validation have been performed.
