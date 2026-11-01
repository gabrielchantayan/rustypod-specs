<!-- Trusted provenance: F-a00bdbbd286c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a00bdbbd286c

## Purpose

F-a00bdbbd286c conditionally reads a four-byte table entry selected through a pointer-shaped argument. A zero entry permits ordinary return. A nonzero entry initiates an ordered dependency-call sequence whose second call is characterized as non-returning.

Eligibility is determined by **distinct signed-conversion and pointer-comparison gates**, not unconditionally by a signed, half-open interval:

- The selector pointer must convert to a nonnegative signed integer.
- The selector pointer must differ from the pointer obtained by converting the descriptor’s second integer word.
- The selector’s signed integer conversion must be less than or equal to the signed integer conversion of that derived pointer.

Only under additional target conversion guarantees do these gates simplify to “selector is nonnegative and strictly less than the stored bound.”

Ordinary return does not distinguish a skipped table access from an inspected zero entry. An inferred purpose is enforcement of a zero-entry invariant, but the meanings of the descriptor, table entries, and exceptional call sequence are not established.

## Interface

### Inputs

**Descriptor address**

An address used to read two consecutive integer words:

| Location in the four-byte integer model | Operational role |
|---|---|
| Offset 0 | Signed integer used in the selected-entry address calculation |
| Offset 4 | Signed integer converted to a pointer for eligibility checks |

The second word is not directly compared with the selector as an integer. Its value first undergoes an integer-to-pointer conversion; the resulting pointer participates in pointer equality and is subsequently converted back to a signed integer.

“Base-address word” and “bound word” are convenient operational descriptions, not established declared field names or semantic types.

**Selector pointer**

A pointer-shaped argument used both:

- As a pointer in equality comparisons.
- Through conversion to a signed integer for sign checks, an inclusive comparison, and table-offset arithmetic.

The function does not dereference the selector pointer itself.

### Output

There is no explicit return value or status result. Observable outcomes include:

- Ordinary return without inspecting a table entry.
- Ordinary return after reading a zero entry.
- Invocation of the dependency sequence after reading a nonzero entry.
- Failure or other behavior caused by an invalid access, invalid arithmetic or conversion, or a dependency.

No local recovery is defined for those failures.

### Widths and layout

The source-facing model uses four-byte signed integer words, corresponding to 32-bit signed values. On that model, the descriptor words occupy offsets 0 and 4, and the selected table read consumes four bytes. The table-offset multiplier is explicitly four.

Pointer width is not established by those integer widths. In particular, a descriptor word is an integer converted to a pointer, not an independently established pointer-width field.

The original ABI’s integer and pointer representations, conversion rules, alignment requirements, and treatment of invalid addresses are not independently supplied. The four-byte descriptor layout is therefore a representation-level contract; mapping it to a different ABI requires confirmation of the integer width.

### Path-dependent access requirements

- A selector whose signed integer conversion is negative requires no descriptor or table access.
- A selector whose signed integer conversion is nonnegative requires the second descriptor word to be readable, even when a later gate rejects it.
- Passing all eligibility gates additionally requires the first descriptor word and the selected table word to be readable.
- The relevant conversions and arithmetic must have defined behavior under the target execution model.

A null descriptor is not rejected explicitly. Its acceptability depends on whether the chosen path accesses it.

## Behavior

### Initial selector handling

Before any descriptor read, the function compares the selector pointer with a null pointer and checks whether its signed integer conversion is greater than −1.

If that integer conversion is negative, the descriptor is not read. The later signed gate also excludes a table access, and the function returns ordinarily without dependency calls.

If the conversion is nonnegative, the function reads the descriptor’s second integer word and converts that integer to a pointer. Equality between this derived pointer and the original selector pointer replaces the earlier null-pointer equality result.

Consequently, **a null selector is not an unconditional early-return case**. When its integer conversion is nonnegative, the function still reads the second descriptor word and applies the subsequent gates.

These statements assume consistent evaluation of the target’s pointer-to-integer conversion for the unchanged selector. No dependency call occurs between the selector checks.

### Distinct eligibility gates

After reading and converting the second descriptor word, the function tests the following conditions in order:

1. The derived pointer and selector pointer must be unequal.
2. The selector pointer’s signed integer conversion must again be nonnegative.
3. That conversion must be less than or equal to the signed integer conversion of the derived pointer.

Failure of any gate causes ordinary return without reading the first descriptor word or a table entry.

The final comparison is **inclusive**. It is not itself a strict-less-than comparison.

The preceding pointer inequality does not, without additional target guarantees, establish inequality between the two converted integers. Distinct pointers could have equal integer conversions under an otherwise unspecified conversion model. In that case, the inclusive integer comparison can succeed at equality.

Likewise, converting the stored bound integer to a pointer and then back to an integer is not independently established to reproduce the stored value. Eligibility therefore cannot generally be expressed by comparing the selector’s converted integer directly with the original second descriptor word.

### Conditional interval interpretation

The familiar interval interpretation is valid if the target guarantees, for the relevant values, that:

- The stored bound’s integer-to-pointer-to-integer round trip preserves its signed integer value.
- Equality between the selector pointer and the derived pointer corresponds exactly to equality between their signed integer conversions.
- The conversions are defined and consistent.

Under those conditions, pointer inequality excludes integer equality, and the inclusive integer comparison becomes a strict comparison with the stored bound. Eligibility then means a nonnegative selector integer strictly below that bound.

**Only under these conditions:**

- A negative or zero stored bound permits no table access.
- A positive bound permits converted selector values from zero through one less than the bound.
- A selector integer equal to the stored bound is excluded.
- A bound of 3 admits selector integers 0, 1, and 2, but not 3.

Without these guarantees, neither the stored bound’s sign nor equality between the stored bound and the selector’s integer conversion is sufficient to determine eligibility.

### Selected-entry read

Once all gates pass, the function:

- Reads the descriptor’s first signed integer word.
- Multiplies the selector’s signed integer conversion by four.
- Adds that product to the first integer word.
- Converts the resulting integer to a pointer.
- Reads one integer word through that pointer.

Thus, the entry address is produced by **integer arithmetic followed by an integer-to-pointer conversion**. It is not established as ordinary pointer arithmetic on a validated allocation.

The observable logical read order is:

1. Second descriptor word.
2. First descriptor word, only after eligibility succeeds.
3. Selected table word.

The selected word is compared with zero. Its signedness does not affect this equality test.

If the word is zero, the function returns ordinarily. It does not write the entry or call either dependency.

If the word is nonzero, the function proceeds to the dependency sequence. It does not reread the descriptor or entry before the first call.

### Dependency sequence

The first call is to F-9beda7955a36, with no explicit arguments shown.

If that call returns normally, the next call is to F-feee9156a257, also with no explicit arguments shown. No intervening local memory update, eligibility check, or table reread occurs.

F-feee9156a257 is characterized as non-returning in the supplied static material. Under that characterization, the nonzero-entry path does not return ordinarily to this function’s caller.

The mechanism of non-return is unknown. The first dependency may itself terminate, transfer control elsewhere, or otherwise fail to return, preventing the second call from being reached.

### Arithmetic boundaries

The eligibility gates do not establish that the table-offset multiplication or subsequent addition is representable.

Under ordinary 32-bit signed arithmetic, multiplying a nonnegative selector integer by four remains within the signed range only through selector value 536,870,911. Even within that range, adding the descriptor’s first integer word may exceed the signed range.

The function supplies no explicit overflow check, allocation-size check, or address-validity check. A large positive bound does not prove that a correspondingly large table exists.

The contract does not assume modular wraparound. Under ordinary signed-language semantics, signed overflow would invalidate a defined-execution guarantee; the exact original execution model remains unresolved.

## Side effects

### Direct reads and writes

The function contains no explicit memory writes.

| Path | Direct data reads |
|---|---|
| Negative selector integer conversion | None |
| Nonnegative selector conversion, later gate rejects | Second descriptor word |
| All eligibility gates pass | Second descriptor word, first descriptor word, selected table word |

This describes logical program accesses, not speculative hardware activity or bus transactions.

The descriptor reads and table read are separate observations. No synchronization or coherent-snapshot guarantee is established.

### Dependency effects

A nonzero selected entry triggers the dependency path. Either dependency may modify globals or reachable memory, acquire or release resources, or alter control flow. Their bodies and contracts are not supplied.

The absence of direct writes does not make the complete invocation side-effect-free.

### Ownership and lifetime

There is no explicit local allocation, deallocation, reference-count operation, or ownership transfer.

Neither dependency is assumed to preserve or release any particular memory. Memory readable before the first call is not guaranteed to remain allocated or unchanged afterward.

### Errors

There is no error return, local exception handler, fallback address, or recovery path for:

- An unreadable required descriptor word.
- An invalid converted pointer.
- Unrepresentable address arithmetic.
- An unreadable selected entry.
- Dependency failure or non-return.

An entry-read failure does not establish that either dependency is called.

## Dependencies

| Opaque ID | Invocation condition | Established relationship | Unresolved contract |
|---|---|---|---|
| F-9beda7955a36 | A selected entry was successfully read and was nonzero | First dependency call; no explicit arguments shown | Purpose, implicit inputs, state changes, ownership effects, errors, and whether it returns |
| F-feee9156a257 | F-9beda7955a36 returns normally | Second dependency call; characterized as non-returning; no explicit arguments shown | Non-return mechanism, state changes, ownership effects, and reliability of the characterization |

No dependency return value participates in a local decision. Absence of explicit arguments does not imply absence of implicit state dependencies.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Required accesses, conversions, and arithmetic are assumed valid unless an example concerns their failure.

### Examples that do not require the simplified interval interpretation

1. **Negative converted selector with an unusable descriptor.**  
   If the selector pointer consistently converts to −1, the function returns without reading descriptor or table memory and without calling either dependency.

2. **Nonnegative selector with an unreadable second descriptor word.**  
   The function attempts that read before it can determine pointer equality or complete eligibility. Ordinary return is not guaranteed.

3. **Derived pointer equals selector pointer.**  
   If the selector conversion is nonnegative and converting the second descriptor word produces the selector pointer itself, the pointer-equality gate rejects the access. The first descriptor word and table entry are not read.

4. **Unequal pointers with equal integer conversions.**  
   Suppose the selector converts to 4, the derived pointer differs from it, and that derived pointer also converts to 4. If the target permits those relationships, all eligibility gates pass. Integer equality alone does not exclude the table read.

5. **Stored bound differs from its round-trip result.**  
   Suppose the stored second word is 2, but its pointer conversion converts back to signed integer 5. A selector converting to 3 passes the numerical gate if the pointers are unequal. The stored value 2 is not a direct numerical limit in this scenario. This is a conditional example, not a claim that the actual target has such conversions.

6. **Unequal pointers but failed inclusive comparison.**  
   If the selector converts to 5 and the derived pointer converts to 4, the function returns after the second-word read. It does not read the table base or entry.

7. **Eligible zero entry.**  
   If all gates pass and the selected entry is zero, the function returns without dependency calls or direct memory writes.

8. **Eligible nonzero entry.**  
   If all gates pass and the entry is nonzero, F-9beda7955a36 is called. If it returns normally, F-feee9156a257 is called.

9. **First dependency does not return.**  
   A non-returning first call prevents the second call from being reached through this sequence.

### Examples requiring interval-preserving conversions

The following additionally assume all conversion guarantees stated under “Conditional interval interpretation.”

10. **Zero selector, zero bound.**  
    A selector converting to 0 and stored bound 0 is rejected by pointer equality after the second-word read.

11. **Zero selector, positive bound.**  
    With selector integer 0 and stored bound 1, the first table entry is inspected. If it is zero, the function returns without dependency calls.

12. **Equality with the stored bound.**  
    Selector integer 4 and stored bound 4 produce no table read. Pointer equality provides the exclusion.

13. **Negative stored bound.**  
    Selector integer 0 and stored bound −1 produce no table read because the round-tripped bound remains −1 and the inclusive comparison fails.

14. **Last eligible entry.**  
    Selector integer 2 and stored bound 3 select the word at an integer-computed displacement of eight bytes from the first descriptor word’s value. A zero word permits ordinary return.

Passing eligibility in any example does not excuse invalid arithmetic or an unreadable entry address.

## Uncertainties

The following remain **documented_only**:

- Original ABI details, including confirmation of integer width, pointer width, alignment, and pointer representations.
- Exact pointer-to-integer and integer-to-pointer conversion behavior.
- Whether the stored bound survives its integer-to-pointer-to-integer round trip unchanged.
- Whether pointer equality corresponds to equality of signed integer conversions.
- Behavior on signed arithmetic overflow or invalid converted addresses.
- Whether descriptor and table memory can change concurrently.
- The semantic meaning and actual allocation extent of the descriptor and table.
- Both dependencies’ implicit inputs, state changes, ownership effects, and failures.
- The mechanism and reliability of the second dependency’s non-returning characterization.

The bounded-table invariant is inferred intent. The unconditional contract is the sequence of conversions, separate gates, conditional reads, zero test, and ordered calls—not the simplified strict interval.

## Evidence

The supplied static representation establishes:

- A selector null-pointer comparison before the first sign check.
- A signed selector conversion check before any descriptor read.
- A second-descriptor-word read on the nonnegative path.
- Conversion of that integer word to a pointer.
- A pointer-equality exclusion distinct from the later integer comparison.
- A repeated selector sign check followed by an inclusive comparison of two pointer-to-signed-integer conversions.
- A conditional first-descriptor-word read, integer scaling by four, integer addition, conversion to a pointer, and selected-word read.
- A zero comparison followed, for nonzero entries, by ordered calls to F-9beda7955a36 and F-feee9156a257.
- No explicit local memory write or return value.

Strict-less-than eligibility against the stored bound is a conditional consequence of additional conversion guarantees, not an unconditional fact established by the comparisons alone.

The non-returning characterization comes from the supplied static material; the dependency implementation was not available for independent confirmation.

No runtime tests have been performed.
