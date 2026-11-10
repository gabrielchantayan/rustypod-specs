<!-- Trusted provenance: F-1cd4c8db3fc1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1cd4c8db3fc1

## Purpose

F-1cd4c8db3fc1 accepts two 32-bit input words and produces a represented 32-bit result. Its visible behavior consists of:

- Two unconditional dependency invocations.
- A rejection gate based on carry and zero conditions.
- Word-level magnitude construction and classification.
- Direct returns for several classes of input.
- Configuration-dependent indirect transfers for the remaining class.

**Established static evidence:** The supplied representation supports the invocation order, explicit arguments, branch predicates, word transformations, direct results, and conditional shared-data reads described below.

**Inferred intent:** The two-word arrangement, exponent-like extraction, magnitude construction, and exceptional-value classification resemble a conversion from a floating-point representation to an unsigned integer. This resemblance does not establish a numerical format, rounding standard, or complete conversion contract.

**Unresolved behavior:** Condition provenance, oversized-shift semantics, dependency effects, and the actual meaning of the indirect transfers remain documented_only. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Width | Established use |
|---|---:|---|
| Lower word, designated **L** | 32 bits | Unsigned bit extraction and a whole-word nonzero test. |
| Upper word, designated **H** | 32 bits | Signed comparisons and right shifts, plus word-level bit extraction and arithmetic. |

These designations are specification notation. Both inputs are passed by value; neither is visibly dereferenced as a pointer.

For signed operations, H is interpreted as a 32-bit two’s-complement integer. Define **Q** as that signed value arithmetically shifted right by 20 positions. Under this interpretation, Q ranges from −2048 through 2047.

The function also depends on:

- A represented carry condition.
- A represented zero condition.
- A represented link-register value used to form an indirect destination.
- A shared 32-bit control word, designated **C**.
- Three other shared 32-bit values, designated **D**, **K1**, and **K2**.

The explicit inputs therefore do not determine every outcome.

### Result and error interface

Every represented returning path supplies one 32-bit word. The result may be:

- `0xFFFFFFFF`.
- Zero.
- A logically right-shifted magnitude word.
- The result supplied by an unresolved indirect transfer.

There is no visible separate status result, error object, or output pointer. The all-ones result is not established as exclusively an error indication: several paths can produce it.

No recovery behavior is shown if a dependency or indirect transfer fails to return normally.

### Arithmetic and representation

The contract describes the recovered operations as 32-bit word operations:

- Unsigned word arithmetic retains the low 32 bits.
- Left shifts discard bits beyond bit 31.
- Complements invert exactly 32 bits.
- The whole-word nonzero test contributes either zero or one.
- Right shifts of signed H are interpreted as arithmetic shifts.
- Right shifts of the constructed magnitude are logical shifts.

These are descriptions of the supplied static representation, not a claim that every recovered signed expression is independently validated as portable language-level arithmetic.

A magnitude-shift count can exceed 31. Its result is not established by the representation alone.

## Behavior

### Unconditional dependency sequence

Before any visible input classification, the function invokes F-9e5e664c8ae7 with the 32-bit argument `0xFFFFFFFF`.

Its represented 64-bit result is divided into a lower 32-bit component and an upper 32-bit component. The function then invokes F-11ed8242c40c with four arguments, in this order:

1. The lower component.
2. The upper component.
3. L.
4. H.

No ordinary return value from F-11ed8242c40c is consumed by the subsequent visible calculations.

This does not make either dependency dispensable. Their condition-state effects and other side effects are unknown. If either invocation does not return normally, the subsequent behavior is not reached.

### Condition-based rejection

After both dependencies, the function examines the represented carry and zero conditions.

It returns `0xFFFFFFFF` if carry is clear or zero is set. Processing continues only if carry is set and zero is clear.

The representation does not establish whether these conditions originated at function entry, were produced by either dependency, or reflect another recovered machine-state relationship.

No visible read of C, D, K1, or K2 precedes this gate. Reads performed inside dependencies are outside the available evidence.

### Initial magnitude construction

For an invocation that passes the gate, the function constructs a 32-bit magnitude word, designated **M**, by combining:

- H shifted left by 11 positions.
- L logically shifted right by 21 positions.

The combination is bitwise OR.

If Q is nonzero, bit 31 of M is forced to one.

If Q is zero, bit 31 of M is necessarily clear. Under the stated arithmetic-right-shift interpretation, Q equal to zero restricts H to `0x00000000` through `0x000FFFFF`. Consequently:

- The shifted H contribution occupies only bits 11 through 30.
- The shifted L contribution occupies only bits 0 through 10.
- Neither contribution can set bit 31.
- M lies between `0x00000000` and `0x7FFFFFFF`, inclusive.

Thus the initial construction guarantees that bit 31 is set exactly when Q is nonzero.

The lowest 21 bits of L do not contribute to M. They remain relevant to the later test of whether the entire lower word is zero.

### Negative-Q path

When Q is negative, the function compares signed H plus `0x40000000` with −`0x00100000`.

It returns zero if the sum is strictly less than −`0x00100000`.

The addition is representable as a signed 32-bit value within this branch. The predicate is equivalent to signed H being less than −`0x40100000`, or H having a bit pattern in the inclusive range:

`0x80000000` through `0xBFEFFFFF`.

H equal to `0xBFF00000` does not satisfy this strict comparison.

If the comparison is false, processing continues to the later classifier. Negative-Q inputs never take the nonnegative magnitude-shift path.

### Nonnegative-Q magnitude path

When Q is nonnegative, the function compares Q minus `0x380` with `0x9F`.

If the difference is less than `0x9F`, Q is in the range 0 through 1054. The applicable shift quantity is:

**S = 1054 − Q.**

S is therefore between zero and 1054 on this path, without subtraction overflow.

The result is determined as follows:

- If S is at least 256, return zero.
- Otherwise, return M logically shifted right by the low eight bits of S.

The resulting ranges are:

| Q | S | Represented outcome |
|---:|---:|---|
| 0–798 | 1054–256 | Direct zero return. |
| 799–1022 | 255–32 | Logical right shift with an unresolved oversized count. |
| 1023–1054 | 31–0 | Logical right shift within the word width. |

Masking S to eight bits does not establish masking it to five bits. In particular, counts from 32 through 255 must not be silently interpreted modulo 32.

When Q is at least 1055, the function bypasses this magnitude return and continues to classification.

### Later classifier

For every invocation reaching this stage, construct a 32-bit classifier, designated **T**, from twice H plus one if L is nonzero, retaining the low 32 bits.

If T is below `0xFFE00001`, return the 32-bit complement of signed H arithmetically shifted right by 31 positions.

Under the stated arithmetic-shift interpretation, the result is:

- `0xFFFFFFFF` for nonnegative H.
- Zero for negative H.

An additional comparison with `0xFFDFFFFF` divides this case internally, but both branches return the same expression. It creates no distinct represented result or shared-data access.

Only T values at or above `0xFFE00001` reach configuration dispatch.

### Configuration-dependent transfer

First read C and use its low four bits to select a mode.

Every represented transfer uses a destination obtained by clearing the low two bits of the represented link-register value. The supplied representation models each transfer as yielding the result returned by this function.

The actual transfer mechanism, calling convention, and destination identity are unresolved. The explicit argument counts below describe only what is shown.

#### Mode 9

Read C again and inspect bit `0x00100000`.

If that bit is clear:

- Read D.
- Transfer with explicit arguments zero and the value read from D, in that order.

If that bit is set:

- Read C again for the argument calculation.
- Shift that 32-bit word left by 15 positions.
- Complement the resulting 32-bit word.
- Shift the complemented word left by one position.
- Transfer with explicit arguments `8` and that transformed word.

The second argument is a word-level transformation, not a Boolean value. The complement occurs after the first shift.

#### Mode 10

Read C again and inspect bit `0x40`.

Transfer with one explicit argument:

- `0x80000000` if the bit is set.
- Zero if the bit is clear.

#### Mode 8

Read C again and inspect bit `0x40`.

If that bit is set, transfer with two explicit zero arguments.

Otherwise, the represented access order is:

1. Read K1 as the provisional first argument.
2. Read D as the provisional second argument.
3. Read C and inspect bit `0x10`.
4. If that bit is set, replace the first argument with zero and read K2 as the replacement second argument.
5. Transfer with the resulting argument pair.

K1 and D are read before the replacement decision, including when both provisional argument values are subsequently discarded.

#### Other modes

Transfer to the same represented destination with no explicit arguments shown.

This does not establish a zero-argument machine-level interface or prove that no additional register or state inputs are consumed.

## Side effects

The visible body establishes:

- Ordered invocation of F-9e5e664c8ae7 and F-11ed8242c40c.
- Conditional reads of C, D, K1, and K2.
- Conditional indirect control transfer.
- A 32-bit result on each represented returning path.

It shows no explicit shared-memory write, allocation, deallocation, or input-buffer mutation.

The absence of visible writes does not establish purity. Either named dependency or the indirect destination may read or modify shared state, allocate or release memory, alter conditions, fail, or not return. None of those contracts is supplied.

No ownership transfer is established. The explicit inputs are words rather than demonstrated owned objects, and reading a shared value does not establish acquisition, retention, or release of a resource.

Repeated appearances of C are separate reads in the supplied representation. They are not specified as one entry snapshot. Whether they correspond to distinct physical accesses, are atomic or volatile, or can observe concurrent changes remains unresolved.

## Dependencies

| Dependency | Established interaction | Unresolved contract |
|---|---|---|
| F-9e5e664c8ae7 | First invocation; receives `0xFFFFFFFF`; represented result supplies two 32-bit components. | Meaning, state changes, condition effects, ownership effects, errors, and nonreturning behavior. |
| F-11ed8242c40c | Second invocation; receives the preceding lower and upper components, followed by L and H. | Operation, condition production, shared-state effects, ownership effects, and failures. |
| Unresolved indirect destination | Receives the mode-selected explicit arguments; its represented result becomes this function’s result. | Identity, ABI, additional inputs, actual transfer semantics, result validity, state changes, ownership, and errors. |

No numerical or memory-management contract is imported from these dependencies.

## Acceptance examples

These examples are conditional consequences of the static representation, not runtime observations. Unless stated otherwise, assume both initial dependencies return normally and the gate observes carry set and zero clear.

### Rejection precedes classification

For any L and H, carry clear or zero set requires a result of `0xFFFFFFFF`.

Both named dependencies have already been invoked. No subsequent visible control-word or exceptional-data read is reached.

### Zero-Q construction keeps bit 31 clear

For H equal to `0x000FFFFF` and L equal to `0xFFFFFFFF`:

- Q is zero.
- The shifted H contribution is `0x7FFFF800`.
- The shifted L contribution is `0x000007FF`.
- M is `0x7FFFFFFF`, with bit 31 clear.
- S is 1054, so the function returns zero without executing the magnitude right shift.

More generally, no input with Q equal to zero can produce an initial M with bit 31 set.

### Zero input

With both words zero, Q and M are zero. S is 1054, so the result is zero. Configuration dispatch is not reached.

### Ordinary magnitude extraction

With L zero and H equal to `0x3FF00000`:

- Q is 1023.
- M is `0x80000000`.
- S is 31.
- The result is one.

With L zero and H equal to `0x40000000`, Q is 1024, M is `0x80000000`, S is 30, and the result is two.

These are bit-level results; they do not require a floating-point interpretation.

### Negative threshold

For H equal to `0xBFEFFFFF`, the negative threshold returns zero regardless of L.

For H equal to `0xBFF00000`, the strict threshold comparison is false. Processing reaches the later classifier, which also returns zero. Equal results do not imply identical paths.

### Oversized shift boundary

With L zero and H equal to `0x3FE00000`, Q is 1022, M is `0x80000000`, and S is 32.

The numerical result remains documented_only until the target’s oversized-shift semantics are established. A modulo-32 result is not justified.

### Classifier boundary

With H equal to `0x7FF00000` and L zero, T is `0xFFE00000`; the direct result is `0xFFFFFFFF`.

With the same H and any nonzero L, T is `0xFFE00001`; configuration dispatch is reached instead.

### Mode 8 replacement preserves earlier reads

Suppose dispatch selects mode 8, the subsequent bit-`0x40` observation is clear, and the later bit-`0x10` observation is set.

The represented read sequence includes K1, D, the bit-`0x10` control read, and then K2. The transfer receives zero and the value read from K2. The earlier reads are not omitted merely because their values are replaced.

### Mode 9 transformation

Suppose mode 9 is selected, its bit-`0x00100000` test is set, and the subsequent argument-forming read yields C equal to `0x00100000`.

The first left shift truncates to zero; complementing yields `0xFFFFFFFF`; the final left shift yields `0xFFFFFFFE`. The explicit transfer arguments are therefore `8` and `0xFFFFFFFE`.

## Uncertainties

The following remain **documented_only**:

- The numerical meaning and encoding of the input pair.
- Any named conversion or rounding standard.
- The producer and lifetime of the carry and zero conditions.
- Logical-right-shift results for counts from 32 through 255.
- Independent confirmation of the signed-shift instruction semantics.
- The indirect destination’s identity, ABI, and additional machine-state inputs.
- Whether the indirect operations are calls, continuation transfers, or another control-flow mechanism.
- Whether the represented link-register value reflects entry state or another recovered value.
- The meanings, mutability, and access properties of C, D, K1, and K2.
- Dependency effects on globals, memory ownership, conditions, failures, and termination.
- Behavior omitted or distorted by the supplied reconstruction.

No indirect-target validation is visible. Consequences of an invalid destination are unspecified.

## Evidence

This contract uses only the supplied static representation. No runtime tests have been performed.

Established evidence includes the dependency sequence, explicit argument order, gate predicate, magnitude construction, branch thresholds, direct return expressions, classifier boundary, conditional shared reads, and mode-specific transfer arguments.

The zero-Q bit-31 requirement follows from the stated arithmetic shift and the resulting upper-word range: H contributes only bits 11–30, while L contributes only bits 0–10.

The supplied material reports removal of an unreachable block and unresolved indirect-control-flow reconstruction. Those limitations constrain conclusions about exceptional transfers; they do not establish missing destinations, errors, or ownership rules.

The floating-point conversion interpretation remains inferred intent, separate from the word-level contract.
