<!-- Trusted provenance: F-af64faf3ea62; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-af64faf3ea62

## Purpose

F-af64faf3ea62 conditionally updates two 32-bit fields in a caller-supplied object, optionally invokes F-240aed7a57b1, obtains a 32-bit result from F-b01436b9ce2b, and dispatches through a global function-pointer slot.

Execution beyond the entry guards requires two nonzero candidate values and a nonzero execution gate. When those guards pass, the selection rule is:

- If the previous selection equals the first candidate, select the second candidate.
- Otherwise, select the first candidate.

The function then copies an associated 32-bit word into the object. These writes occur before either direct dependency is invoked.

Alternation between buffers, resources, or display states is **inferred intent**. The evidence does not establish those application-level meanings, validate the candidate addresses, or identify the indirect dependency.

## Interface

### Arguments and return

The supplied representation exposes three incoming 32-bit integer arguments.

| Argument | Established use |
|---|---|
| Object address | Base for fixed-offset object accesses |
| Second incoming value | Replaced by the second-candidate field before any consequential use |
| Third incoming value | Replaced by the execution-gate field before any consequential use |

The original second and third incoming values do not affect the represented reads, writes, branches, or calls. Short-circuit evaluation excludes their use on paths where they have not yet been replaced.

The function has no explicit return value. Results from the optional dependency and the final indirect invocation are unused.

### Object layout

Offsets are byte displacements from the object address. Field labels describe local behavior, not established application types.

| Offset | Width | Role |
|---|---:|---|
| `0x48` | 32 bits | Associated word used when the selected value equals the retained first candidate |
| `0x4C` | 32 bits | Associated word used otherwise |
| `0x50` | 32 bits | Destination for the associated word |
| `0x54` | 32 bits | First candidate; reread later for address calculations |
| `0x58` | 32 bits | Second candidate |
| `0x5C` | 32 bits | Previous selection; then destination for the selected value |
| `0x78` | 32 bits | Nonzero execution gate |
| `0xF8` | 8 bits | Nonzero gate for the optional dependency |

The associated words at `0x48` and `0x4C` are copied as opaque bit patterns. This function does not dereference their contents.

Additional accesses and derived values are:

| Address basis | Established use |
|---|---|
| Retained previous selection plus `4` | Read a 32-bit word for the second indirect argument |
| Selection reread from `0x5C`, plus `4` | Read a 32-bit word for the first indirect argument |
| First candidate reread from `0x54`, plus `0x98` | Construct the argument to F-b01436b9ce2b |
| First candidate reread after that dependency, plus `8` | Read a 32-bit multiplication operand |
| Global function-pointer slot | Obtain the final indirect target |

The global target slot is separate from the object. Its exact storage width and complete ABI are not established.

### Preconditions for normal completion

Each reached memory access must be valid. The function performs no initial object-address check, size check, or explicit alignment validation.

A fully progressing guarded path additionally requires:

- Valid readable and writable storage for the reached object fields.
- Satisfaction of each invoked dependency’s requirements.
- Normal completion of earlier dependencies before later operations can occur.
- Valid memory for the final pointer-derived reads.
- A readable global target slot containing a compatible callable target.
- Address calculations and multiplication within whatever arithmetic domain the execution environment supports.

The entry guards establish none of these validity conditions. In particular, the previous selection need not be nonzero or equal to either candidate.

Dependencies may alter fields, referenced memory, or the global target before those locations are subsequently read. Object validity must therefore hold at each actual access, not merely at entry.

## Behavior

### Entry guards

The explicit object reads begin in this order:

1. Read the first candidate at `0x54`.
2. If the first candidate is nonzero, read the second candidate at `0x58`.
3. If both candidates are nonzero, read the execution gate at `0x78`.

Execution continues only if all three values are nonzero.

A failed guard ends the function without explicit writes or dependency invocations:

- A zero first candidate prevents the second-candidate and gate reads.
- A zero second candidate prevents the gate read.
- A zero gate prevents the previous-selection read and all updates and calls.

Only zero versus nonzero matters. The execution gate is not dereferenced or passed onward by this function.

### Selection update

After all three guards succeed, the function reads and retains the previous selection from `0x5C`.

The selection decision uses the candidate values already read during entry:

| Previous selection | Selected value |
|---|---|
| Equal to the retained first candidate | Retained second candidate |
| Unequal to the retained first candidate | Retained first candidate |

The selected value is written as a 32-bit word to `0x5C`.

The previous selection may be zero or unrelated to either candidate. Any value unequal to the first candidate causes selection of the first candidate.

Equal nonzero candidates are permitted. They produce the shared candidate value regardless of the previous selection.

The original previous-selection value remains retained for final argument construction. Writing `0x5C` does not replace that retained value.

### Associated-word update

After the selection write, the selected value is compared with the retained first candidate:

- Equality selects a 32-bit read from `0x48`.
- Inequality selects a 32-bit read from `0x4C`.

That word is written to `0x50`.

The established order is:

1. Write the selected value to `0x5C`.
2. Read the chosen associated word.
3. Write that word to `0x50`.

For distinct candidates, selecting the first candidate uses `0x48`; selecting the second uses `0x4C`.

For equal candidates, the source is always `0x48`, including when the selection rule takes the second-candidate alternative. The associated-word choice follows value equality, not the identity of the selection alternative.

Neither write is skipped when the destination already contains the value being written.

### Optional dependency

After both object writes, the function reads the byte at `0xF8`.

- Zero skips F-240aed7a57b1.
- Any nonzero byte invokes F-240aed7a57b1 with no explicit arguments.

The dependency’s result is unused.

Its implementation is unavailable. No preservation of object fields, referenced storage, or global state is assumed. The absence of explicit arguments does not establish an absence of access to shared state.

### Helper invocation

After the optional dependency returns, or immediately if it is skipped, the function rereads the first-candidate field at `0x54`.

It adds the byte displacement `0x98` to this newly read 32-bit value and passes the resulting value to F-b01436b9ce2b. The helper’s 32-bit result is retained for the final multiplication.

This candidate read occurs after the optional dependency. A change made by that dependency can therefore change the helper argument without changing the earlier selection decision.

There is no renewed nonzero check. The function itself does not dereference the derived helper argument before passing it; the helper’s interpretation of that value is unknown.

### Final target and arguments

After F-b01436b9ce2b returns, the represented final dispatch obtains its target from a global function-pointer slot.

The target is not an object field, a fixed function identity, or an entry-time snapshot. Either preceding dependency may affect the slot before the final target read.

The indirect invocation receives three arguments:

| Position | Value |
|---|---|
| First | The 32-bit word at displacement `4` from the selection reread from `0x5C` |
| Second | The 32-bit word at displacement `4` from the retained previous selection |
| Third | The product of the helper result and the 32-bit integer at displacement `8` from the first candidate reread from `0x54` |

The final selection and first-candidate reads occur after the helper returns. They need not match the earlier selected value or the candidate used to construct the helper argument.

The second argument uses the retained previous-selection address, but reads the word at that address only during final argument evaluation. Dependencies may modify that word before this read.

The supplied expression does not establish the relative evaluation order among:

- Reading the global target slot.
- Evaluating the first argument.
- Evaluating the second argument.
- Evaluating the third argument.

These evaluations follow helper completion. No stronger ordering among them is specified. In particular, the contract does not guarantee which final read would occur first if another final read fails.

The indirect result, if any, is unused. The represented function returns if the invocation returns normally.

### Widths and arithmetic

Object words, final pointed-to words, the multiplication operand, and the helper result are 32 bits. The optional-call gate is one byte.

Displacements `4`, `8`, and `0x98` are byte displacements, not element counts.

The multiplication uses 32-bit integer operands. No widening, saturation, range check, or explicit overflow handling is shown. Signed-overflow behavior is **documented_only**: the evidence does not establish either an unbounded mathematical result or a particular wraparound rule.

Address calculations likewise have no explicit overflow checks. Their behavior outside the valid address and arithmetic domain remains **documented_only**.

High-bit-set candidate values are not rejected solely because that bit is set. Entry checks distinguish only zero from nonzero; usable-address requirements remain separate.

## Side effects

On a fully guarded path, the function directly performs two ordered writes:

1. A 32-bit selection write at `0x5C`.
2. A 32-bit associated-word write at `0x50`.

Both precede every dependency invocation. No rollback is shown if a later read, dependency, or dispatch fails.

Guard-failure paths perform no explicit writes and invoke no dependencies.

The global target slot is directly read, not directly written. Dependencies may nevertheless modify it.

No explicit allocation, deallocation, reference-count operation, or ownership transfer appears. Copying a word or passing an address does not establish ownership semantics. Dependency retention, release, mutation, and lifetime requirements remain unknown.

There is no explicit error result, exception handler, or recovery path. Invalid memory, invalid targets, arithmetic outside the supported domain, or abnormal dependency completion can prevent normal return. The precise failure manifestation is not established.

The stated access order does not establish atomicity, synchronization, or cross-thread visibility guarantees.

## Dependencies

| Dependency | Established caller-side contract | Unknowns |
|---|---|---|
| F-240aed7a57b1 | Optional invocation after both writes; no explicit arguments; result unused | Memory and global effects, ownership, retention, release, failure behavior |
| F-b01436b9ce2b | Invoked on every normally progressing fully guarded path; receives the reread first candidate plus `0x98`; provides a 32-bit multiplication factor | Argument interpretation, result meaning and range, memory and global effects, ownership, failure behavior |
| Unresolved indirect dependency | Target obtained from a global slot after helper completion; receives the two 32-bit words and multiplication result | Opaque identity unavailable; complete ABI, argument meanings, effects, ownership, failure behavior |

No identity is invented for the unresolved indirect dependency. None of the dependencies is assumed to preserve global state or to own, retain, or release supplied memory.

## Acceptance examples

These are conditional static specifications, not executed tests.

Unless an example states otherwise, assume valid reached memory accesses, normally returning dependencies, stable relevant storage during the example, a compatible global target, and arithmetic within its representable domain. These general assumptions do **not** imply successful entry guards; the necessary guard premises are stated explicitly below.

### First guard fails

Given a zero first candidate:

- The function explicitly reads only `0x54` from the object.
- It does not read the later object fields or global target slot.
- It performs no explicit writes and invokes no dependencies.

Changing either unused incoming value does not change this result.

### Second or third guard fails

Given a nonzero first candidate and a zero second candidate, execution stops after the read at `0x58`.

Given two nonzero candidates and a zero execution gate, execution stops after the read at `0x78`.

Neither case reads the previous selection, updates either destination, or invokes a dependency. The zero-gate result holds even if the previous selection equals the first candidate.

### Previous selection equals the first candidate

Given **distinct nonzero first and second candidates, a nonzero execution gate**, and a previous selection equal to the first candidate:

- The second candidate is written to `0x5C`.
- The word at `0x4C` is written to `0x50`.
- With the assumed stable storage, the first indirect argument comes from the second candidate plus `4`.
- The second indirect argument comes from the original first candidate plus `4`.

All three successful entry guards are necessary for these updates. A zero gate instead produces no writes.

### Previous selection differs from the first candidate

Given **nonzero first and second candidates, a nonzero execution gate**, and a valid previous-selection address unequal to the first candidate:

- The first candidate is written to `0x5C`.
- The word at `0x48` is written to `0x50`.
- The second indirect argument still comes from the retained previous-selection address plus `4`, even if that address is neither candidate.

Failure of any entry guard prevents these updates and the indirect invocation.

### Equal candidates

Given equal nonzero candidates and a nonzero execution gate:

- The shared candidate value is written to `0x5C`.
- The associated-word source is `0x48`, not `0x4C`.
- The second indirect argument still uses the retained previous-selection address.

### Zero previous selection

Given nonzero candidates, a nonzero execution gate, and a zero previous selection:

- The first candidate is selected.
- The word at `0x48` is copied to `0x50`.
- Final argument evaluation includes an attempted read at the retained zero value plus `4`.

The ordinary valid-memory assumption is not asserted for that final read in this example. The entry guards do not prevent the attempt, and no particular platform response is specified.

### Dependencies change candidate fields

Given successful entry guards and an invoked F-240aed7a57b1:

- If it changes `0x54`, the helper argument uses the changed value plus `0x98`.
- If F-b01436b9ce2b changes `0x54` again, the final multiplication operand is read through that later value.
- If the helper changes `0x5C`, the first indirect argument uses that changed selection.

These changes do not cause the earlier selection decision or associated-word update to be recomputed. All reached accesses must remain valid.

### Dependencies replace the global target

Given successful entry guards, if F-b01436b9ce2b replaces the global target and no subsequent change intervenes, final dispatch uses that replacement.

If only the optional dependency replaces it and the helper leaves it unchanged, that replacement is used.

Neither case establishes an order between the final target read and the final argument reads.

### Bounded multiplication

Given successful entry guards, a helper result of `7`, and a value of `6` at displacement `8` from the post-helper first candidate, the third indirect argument is `42`.

This example establishes no overflow rule.

## Uncertainties

The following remain **documented_only**:

- Concrete object and candidate types and their application meanings.
- Helper-argument interpretation and result range.
- Indirect dependency identity and complete calling convention.
- Exact global target-slot width and initialization requirements.
- Relative evaluation order of the final target and argument expressions.
- Overflow behavior and invalid-address behavior beyond the absence of explicit checks.
- Platform alignment and address-validity requirements.
- Concurrency, atomicity, and visibility guarantees.
- Dependency mutations of object fields, referenced memory, and global state.
- Dependency ownership, retention, release, and lifetime effects.
- Usability of low, unaligned, or high-bit-set addresses.

Alternating-selection intent is inferred, not a guarantee of valid resource switching.

## Evidence

The supplied static representation establishes the short-circuit guards, fixed-offset access widths, retained and reread values, ordered object writes, optional invocation, helper invocation, and represented dispatch through a global target slot after helper completion.

The final expression establishes the argument sources but not the relative evaluation order of the target and arguments. Dependency implementations and runtime observations are unavailable.

Comments and prior descriptions are not treated as independent behavioral authority. No runtime tests have been performed. All acceptance examples are conditional consequences of the supplied static evidence.
