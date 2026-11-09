<!-- Trusted provenance: F-cfd9a93ecce5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cfd9a93ecce5

## Purpose

F-cfd9a93ecce5 computes a signed integer result through configuration-directed execution. It accepts one integer and may return a staged result or continue indefinitely. Configuration controls both arithmetic and control flow; the supplied evidence does not establish a fixed sequence of operations or a termination guarantee.

The numerical constants suggest threshold classification. That is **inferred intent**, not an application-level guarantee: the actual predicates combine unsigned subtraction, bitwise operations, and bit extraction. Ordinary signed comparisons are not interchangeable with those predicates.

Under a conditional 32-bit integer model, bit-preserving unsigned-to-signed conversion, arithmetic signed right shift, and absence of signed overflow, every normal return is either **0 or 147**. Without those conditions, this restriction is not established.

## Interface

### Values and ownership

- **Input:** one native signed integer, passed by value.
- **Output:** one native signed integer on normal return.
- **Errors:** no designated error value, status channel, or explicit recovery path.
- **Ownership:** no pointer argument, returned allocation, buffer, or ownership transfer is visible.
- **Lifetime:** arithmetic state and the staged result belong to the invocation. They are not retained explicitly between calls.

Negative input values are accepted without an explicit check. An execution that returns without reaching multiplication setup need not use the argument at all.

### External configuration

Thirteen scalar configuration roles are referenced. The labels below are descriptive, not storage identifiers.

| Role | Meaning |
|---|---|
| Seed | Initial routing token; also contributes two later routing destinations |
| Bias | Added to the token at each dispatch |
| Route A | Routing associated with the 250-related predicate |
| Return route | Destination after staging a result |
| Route B | Routing associated with the 150-related predicate |
| Route C | Routing associated with the 251-related predicate |
| Route D | Routing associated with the 160-related predicate |
| Comparison factor | Multiplier used to prepare the comparison quantity |
| Input factor | Multiplier applied directly to the argument |
| Upper factor | Multiplier used to prepare the upper quantity |
| Setup route | Destination after multiplication setup |
| Lower-250 route | Alternative destination associated with the 250 cache |
| Lower-380 route | Alternative destination used by default handling |

Concrete values, declarations, storage widths, qualifiers, mutability, and synchronization arrangements are not supplied. No explicit assignment to configuration storage appears.

### Integer interpretation

The evidence distinguishes signed integers, unsigned integers, plain characters, and signed shorts. It does not independently establish their ABI widths.

The detailed formulas and concrete examples below use this **conditional model**:

- Signed and unsigned integers are 32 bits.
- Unsigned arithmetic wraps modulo \(2^{32}\).
- Plain characters and signed shorts promote to signed integer.
- Conversion to an out-of-range signed destination remains target-dependent unless additional conditions are stated.
- Signed arithmetic must be representable for claims of ordinary defined execution.

Let \(U(x)\) mean conversion to a 32-bit unsigned word. Subsequent unsigned operations, including complement, use that width.

Assignment to unsigned storage does not make a preceding signed expression wrap. In particular, input multiplication, signed comparison offsets, dispatch addition, and several routing expressions can overflow before their results are converted.

## Behavior

### Execution state and dispatch

Each invocation begins with zero-valued arithmetic workspace, upper and comparison quantities, two offset caches, two flags, and a staged result. The routing token alone is initialized from Seed.

For descriptive purposes, the flags are called **Decision** and **Retained**. These labels describe their uses, not Boolean types: both are unsigned integer words.

At every dispatch boundary:

- Decision is converted to plain character.
- Retained is then converted to plain character.
- Bias is read, and the signed sum of the current token and Bias determines the selector.

Selectors 0 through 14 have defined handling. Selector 5 and every value outside that range share default handling. Except for selector 4, handling returns to dispatch.

No selector automatically resets unrelated state. Re-entering setup, for example, does not clear the flags or an already staged result. Caches do not track subsequent changes to the comparison quantity.

### Arithmetic preparation and workspace reuse

The workspace is deliberately shared between arithmetic preparation and predicate evaluation. It is not a preserved copy of the input product.

| Selector | Arithmetic effect |
|---|---|
| 1 | Replaces workspace with the signed input-times-Input-factor product converted to unsigned; replaces the upper quantity with workspace-times-Upper-factor evaluated unsigned and converted to signed |
| 3 | Replaces the comparison quantity with current-workspace-times-Comparison-factor evaluated unsigned and converted to signed |
| 0 | Replaces the 250 cache with comparison plus 250 |
| 3 | Also replaces workspace with comparison plus 160 and the 260 cache with comparison plus 260 |
| 11 | Replaces workspace with comparison plus 150 |

The additions of 150, 160, 250, and 260 in this table are signed additions before conversion to unsigned storage.

Selector 1 proceeds through Setup route. Its effect on the upper quantity uses the newly calculated input product. By contrast, selector 3 uses whatever workspace value exists when it is reached, including a value left by an earlier predicate. A specification that always derives the comparison quantity directly from the argument would therefore be incorrect.

### Predicate meaning

For an unsigned threshold \(t\) and unsigned operand \(a\), define

\[
P(t,a)=\operatorname{NOT}\bigl((U(\text{upper})-t)\operatorname{OR}a\bigr).
\]

Subtraction, OR, and complement in this definition are unsigned. Define

\[
H(w)=\left\lfloor w/2^{31}\right\rfloor
\]

for a 32-bit unsigned word. Thus \(H(w)\) is either zero or one.

The ordinary predicate-producing operations are:

| Selector | Threshold | Other operand | Flag replaced | Following token |
|---|---:|---|---|---|
| 6 | 260 | 260 cache | Retained | Seed + 10 |
| 7 | 160 | Incoming workspace | Decision | Route D |
| 8 | 150 | Incoming workspace | Decision | Route B |
| 10 | 251 | \(U(\text{comparison})+251\) | Decision | Route C |
| 13 | 250 | 250 cache | Retained | Route A |
| 14 | 380 | \(U(\text{comparison})+380\) | Retained | Seed + 11 |

Each operation in this table leaves its predicate word in workspace and assigns \(H(P(t,a))\) to the indicated flag. Other flags and caches remain unchanged.

The additions of 251 and 380 are unsigned, unlike the signed cache-preparation additions. Selectors 7 and 8 consume incoming workspace directly; they do not regenerate comparison offsets.

For these predicates, a flag becomes one exactly when both the unsigned subtraction result and the other operand have bit 31 clear. This bit condition is the established meaning. A signed interval interpretation requires additional restrictions on operands.

Selector 3 has a distinct extraction rule. It uses \(P(9,U(\text{comparison})+9)\), converts that word to signed integer, right-shifts the signed value by 31, negates the result, and converts it to unsigned Decision.

With bit-preserving conversion and arithmetic right shift, this produces zero or one. Other conversion or shift semantics can produce a non-Boolean value or otherwise prevent that conclusion. Selector 3 does not retain this predicate word in workspace; its final workspace value is comparison plus 160.

### Routing and flag transfer

Routing expressions are arithmetic combinations, not short-circuit choices. A factor of zero does not remove a configuration read or make an overflowing signed subexpression harmless.

For character-based routing, let \(c\) be the relevant plain-character value after promotion, \(b\) a signed base, and \(r\) a signed weighted contribution. The unsigned combined value is

\[
U(b)\bigl(U(c)\mathbin{\operatorname{XOR}}1\bigr)+U(r),
\]

which is converted to the signed routing token.

| Selector | Character source | Base | Signed contribution | Flag transfer |
|---|---|---|---|---|
| 0 | Entry Decision | Lower-250 route | \((\text{Seed}+10)c\) | Full Decision copied to Retained |
| 5 or unlisted | Entry Decision | Lower-380 route | \((\text{Seed}+11)c\) | Full Decision copied to Retained |
| 9 | Decision | First Route A read + 11 | Second Route A read times \(c\) | Full Decision copied to Retained |
| 2 | Entry Retained | First Route C read + 5 | Second Route C read times \(c\) | Full Retained copied to Decision, then to Retained |
| 11 | Entry Retained | One Route B read − 1 | Another Route B read times \(c\) | Full Retained copied to Decision |

Selectors 2 and 9 perform an additional character conversion of Decision for the unsigned combining factor. For selector 2, that conversion follows the full-word copy from Retained. Under fixed conversion semantics, it agrees with the corresponding entry character value used in the weighted contribution. None of these conversions normalizes the stored flag words.

Selector 3 routes using the newly calculated **full unsigned Decision**, without character narrowing. For Decision \(f\), and its two Route D read values \(d_1,d_2\), its unsigned routing value is

\[
U(d_1+7)(f\mathbin{\operatorname{XOR}}1)+U(d_2)f.
\]

The addition of 7 is signed; the products and their sum are unsigned.

For stable configuration and Boolean flags, the routing rules simplify as follows:

| Selector | Relevant flag | Destination for zero | Destination for one |
|---|---|---|---|
| 0 | Decision | Lower-250 route | Seed + 10 |
| 5 or unlisted | Decision | Lower-380 route | Seed + 11 |
| 9 | Decision | Route A + 11 | Route A |
| 2 | Retained | Route C + 5 | Route C |
| 11 | Retained | Route B − 1 | Route B |
| 3 | Newly computed Decision | Route D + 7 | Route D |

These are conditional simplifications, not replacements for the arithmetic contract. XOR with one is not general Boolean negation, and a narrowed character need not preserve a non-Boolean word.

### Result and termination

Selector 12 replaces the staged result with Retained converted to signed short, promoted to integer, and multiplied by 147. Return route becomes the next token. Staging itself does not return.

Selector 4 immediately returns the current staged result.

Consequently:

- A return before any staging yields zero.
- Later flag changes do not change an already staged result.
- Repeated staging replaces the previous result.
- A path that never reaches selector 4 does not normally return.

For Retained zero or one, staging yields zero or 147. For other values, signed-short conversion matters. The multiplication must also be representable; a sufficiently wide short can admit a converted value whose product with 147 overflows signed integer.

With 32-bit unsigned words, the ordinary predicate extractions produce Boolean flags. If selector 3 also has the stated bit-preserving conversion and arithmetic-shift behavior, initialization and full-word copies preserve that invariant. This establishes the conditional 0-or-147 return restriction.

## Side effects

### Configuration observations

Seed is read before the first dispatch. Each dispatch subsequently reads Bias after the two entry character conversions.

The following table records additional expression-level configuration reads. “Then” denotes established sequencing between expressions; paired reads described as unordered have no established relative evaluation order.

| Selector | Additional reads |
|---|---|
| 0 | Seed, then Lower-250 route |
| 1 | Input factor, then Upper factor, then Setup route |
| 2 | Route C for the offset base, then Route C for the contribution |
| 3 | Comparison factor, then two Route D reads with unspecified relative order |
| 4 | None |
| 5 or unlisted | Seed, then Lower-380 route |
| 6 | Seed |
| 7 | Route D |
| 8 | Route B |
| 9 | Route A for the offset base, then Route A for the contribution |
| 10 | Route C |
| 11 | Two Route B reads with unspecified relative order |
| 12 | Return route |
| 13 | Route A |
| 14 | Seed |

Relevant local preparation precedes later routing reads. Examples include the 250-cache update before selector 0 reads Seed, selector 3’s arithmetic and flag updates before its Route D reads, and result staging before reading Return route.

These are source-level read occurrences, not a guarantee of distinct physical loads. The evidence does not establish volatile access, atomicity, synchronization, or legal concurrent mutation. Repeated occurrences must neither be promised as separate hardware observations nor silently assumed to yield equal values.

### Writes and exceptional behavior

All explicit writes affect invocation-local state. No external write, I/O, allocation, release, synchronization operation, or called function is visible.

There is no explicit handling for inaccessible configuration, signed overflow, target-dependent conversions, or failure to terminate. These conditions do not map to a designated error result. In particular, signed overflow must not be described as ordinary modular arithmetic or a recoverable failure.

## Dependencies

No called-function dependency appears in the supplied body. There is no additional opaque function ID to specify.

Behavior depends on the thirteen configuration roles and the execution environment’s integer widths, promotions, conversions, signed-shift rules, and memory-access semantics. No assumption about a callee preserving globals or owning memory is required or made.

## Acceptance examples

These are **conditional static deductions**, not executed tests. Concrete examples assume the 32-bit model, stable configuration, representable signed arithmetic, and the stated conversion behavior where needed. Configuration roles not reached by an example do not affect its result.

1. **Direct return without using the argument.**  
   If Seed plus Bias selects 4 initially, the result is zero. Beyond the initial Seed read, only the dispatch Bias read is required; no multiplier or Return route is read.

2. **Staging the initial flag.**  
   If the first selector is 12 and Return route plus Bias selects 4, the result is zero. Staging uses the initially zero Retained flag, not the input.

3. **A concrete return of 147.**  
   Let Bias be zero, Seed be 1, Input factor and Upper factor both be 1, Setup route be 14, and Return route be 4. For input 380, setup establishes upper 380 while comparison remains zero. The 380 predicate combines zero subtraction with unsigned 380, producing a retained high bit of one. Its next token is 12, so staging and return yield 147.

4. **Adjacent input with a different result.**  
   Under the preceding configuration, input 379 yields zero. Subtracting unsigned 380 from upper sets bit 31, so the retained predicate flag is zero.

5. **Execution without multiplication setup.**  
   Let Bias be zero, Seed be 6, Lower-380 route be 12, and Return route be 4. Initially zero upper and the zero 260 cache yield Retained zero. The next token is 16, which uses default handling. Decision is still zero, so routing reaches 12 and ultimately returns zero.

6. **Staged-result independence.**  
   If a reachable execution stages 147, subsequently changes Retained to zero, and reaches selector 4 without staging again, it returns 147. Reachability of that sequence is a separate configuration requirement.

7. **Nontermination.**  
   With Bias zero, Seed 7, and Route D 7, every dispatch selects 7. Workspace and Decision can change, but no return selector is reached.

8. **Conditional non-Boolean result.**  
   Suppose selector 3’s signed right shift produces positive one for a negative converted predicate. Negation and unsigned conversion then produce \(2^{32}-1\), not Boolean one. If a reachable continuation transfers that word into Retained, reaches staging unchanged, and converts it to signed-short −1, the staged result is −147. Neither the target semantics nor such a continuation is established by the supplied evidence.

## Uncertainties

- **documented_only — Configuration:** Values, declarations, qualifiers, mutability, and synchronization are unavailable.
- **documented_only — Widths:** Exact integer, character, and short widths are not independently established. A shift by 31 does not alone prove a 32-bit unsigned type; wider types can invalidate Boolean extraction.
- **documented_only — Signed arithmetic:** Input multiplication, comparison offsets, token-plus-Bias, routing offsets, character-weighted products, and potentially result staging can overflow.
- **documented_only — Signed conversion:** Upper, comparison, generated tokens, and selector 3’s predicate conversion can require out-of-range conversion.
- **documented_only — Signed shift and narrowing:** Selector 3’s extraction, plain-character conversion of non-Boolean flags, and signed-short conversion require target semantics.
- **documented_only — Memory observations:** Physical load count and concurrent-mutation behavior are not established.
- **Unknown termination:** No configuration-independent progress condition is supported.
- **Inferred intent:** Threshold classification is plausible; its application meaning remains unknown.

## Evidence

The supplied function body is the sole static evidence. It establishes invocation initialization, selector handling, arithmetic distinctions, configuration-read occurrences, local state changes, staging, and the explicit return.

The formulas and acceptance examples are deductions from those operations under their stated conditions. Configuration values, ABI documentation, and runtime observations were not supplied. The Boolean invariant and 0-or-147 result restriction are conditional, not unconditional interface guarantees.

**No runtime tests have been performed.**
