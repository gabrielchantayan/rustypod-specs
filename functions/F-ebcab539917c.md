<!-- Trusted provenance: F-ebcab539917c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ebcab539917c

## Purpose

F-ebcab539917c conditionally invokes F-c9bded66abe7 and, on rejection or a zero operation result, invokes F-375c8199e7c2 with a selected 32-bit reporting value.

Delegation ordinarily requires:

- Membership of a selector and kind in configured outer sets.
- Two nonnegative signed 32-bit extents.
- Compatibility of the selector and kind under a narrower inner check.

The inner checks reread configuration. If outer eligibility succeeds but neither subsequent inner selector comparison matches, control proceeds to delegation without an inner kind check. This fallthrough is part of the visible behavior; it is not represented by the stable-configuration compatibility table alone.

The operation’s domain and the meanings of the reporting values are not established. “Selector,” “kind,” “extent,” and “reporting value” are descriptive contract labels, not evidence of domain-specific semantics.

## Interface

### Arguments

The function accepts eight arguments:

| Position | Contract name | Representation and use |
|---|---|---|
| 1 | Object base | 32-bit value used as the base of two address calculations and passed unchanged to F-375c8199e7c2. |
| 2 | Payload A | Opaque 32-bit value forwarded unchanged to F-c9bded66abe7. |
| 3 | Payload B | Opaque 32-bit value forwarded unchanged to F-c9bded66abe7. |
| 4 | Extent A | Signed 32-bit value; must be nonnegative for delegation and is forwarded twice. |
| 5 | Extent B | Signed 32-bit value; must be nonnegative for delegation and is forwarded twice. |
| 6 | Selector | Signed 32-bit value used in outer and inner configuration comparisons; not forwarded to the operation dependency. |
| 7 | Kind | Signed 32-bit value used in configuration comparisons and forwarded unchanged to the operation dependency. |
| 8 | Payload C | Opaque 32-bit value forwarded unchanged to F-c9bded66abe7. |

Payloads are not locally dereferenced or validated.

### Configuration storage

The function references **six global 32-bit storage locations**:

| Label | Role |
|---|---|
| S | Selector reference. |
| K₁ | First kind reference. |
| K₂ | Second kind reference. |
| K₃ | Third kind reference. |
| R₀ | Default reporting value, captured before eligibility checks. |
| R₁ | Reporting value selected for an inner compatibility rejection. |

S − 1 and K₂ − 0x330 are derived comparison values, not additional storage locations. The latter subtraction uses 816 decimal.

No configuration contents or distinctness guarantees are supplied. K₁, K₂, K₃, and the derived kind value may overlap. R₀ and R₁ are opaque 32-bit reporting values; no signed interpretation is required for forwarding them.

Repeated references to a configuration label do not establish one shared snapshot. The conditional read sequence is specified under **Behavior**.

### Object layout and validity

Only the delegated path directly reads object-relative memory:

| Byte offset from Object base | Read width | Destination |
|---|---:|---|
| 0xAE4 | 32 bits | Argument 1 of F-c9bded66abe7. |
| 0x90C | 32 bits | Argument 13 of F-c9bded66abe7. |

These offsets are byte displacements. The field values’ semantic types are unknown.

For normal completion of a delegated call, both resulting addresses must support the indicated reads. The function performs no local null, bounds, alignment, or object-validity check. Reporting-only paths do not directly read these fields, but F-375c8199e7c2 may impose its own requirements on Object base.

The supplied representation does not establish address-overflow or invalid-access behavior.

### Output and ownership

There is no established return value.

F-c9bded66abe7 returns a signed 32-bit result that this function distinguishes only as zero or nonzero. That result is not returned to the caller. The result of F-375c8199e7c2, if any, is ignored.

No direct allocation, release, ownership transfer, or retention is established. Forwarding an argument does not establish whether a dependency borrows, retains, consumes, or releases anything represented by it.

## Behavior

### Initial capture and outer eligibility

The first visible configuration read captures R₀ for possible later reporting. This read occurs even when outer eligibility will fail.

Outer eligibility uses short-circuit comparisons in this order:

1. Read S and compare Selector with that value.
2. Only if that comparison fails, read S again and compare Selector with the newly read value minus one.
3. Only if a selector comparison succeeds, begin the outer kind comparisons:
   - Read K₁ and compare Kind with it.
   - If unequal, read K₂ and compare Kind with it.
   - If still unequal, read K₂ again and compare Kind with that newly read value minus 0x330.
   - If still unequal, read K₃ and compare Kind with it.

A successful comparison ends the applicable membership search. In particular, selector rejection prevents all outer kind reads, and an early kind match prevents later outer kind reads.

If outer eligibility fails, invoke F-375c8199e7c2 with Object base and the literal 32-bit value **0x500**, then return. Neither F-c2d002d7d74c nor F-c9bded66abe7 is called.

For stable configuration and representable derived arithmetic, outer eligibility is equivalent to both of the following:

- Selector equals S or S − 1.
- Kind equals K₁, K₂, K₂ − 0x330, or K₃.

This set description does not replace the conditional read sequence when successive reads can differ.

### Signed extent checks

After outer eligibility succeeds, Extent A is checked for nonnegativity. Extent B is checked only if Extent A is nonnegative.

If either extent is negative, invoke F-375c8199e7c2 with Object base and the initially captured R₀, then return. R₁ is not read on this path, and neither operation-related dependency is called.

The accepted range for each extent is 0 through 2,147,483,647. Zero is accepted; −2,147,483,648 is rejected. There is no further local size limit, scaling, multiplication, or combination of the extents.

### Inner compatibility and its read sequence

After both extents pass, read R₁ as the provisional reporting value. This read occurs before the inner selector comparisons, including on paths that ultimately delegate.

The inner comparisons perform fresh configuration reads:

- First, read S and compare Selector with it.
  - If equal, evaluate the first inner kind branch.
  - If unequal, read S again and compare Selector with that newly read value minus one.
    - If equal, evaluate the second inner kind branch.
    - If unequal, proceed directly to delegation.

The first inner kind branch accepts a match against K₁, K₂ − 0x330, or K₃. Its reads are short-circuited in this order:

1. Read K₁.
2. Only if Kind differs, read K₂ and form the derived comparison value.
3. Only if Kind still differs, read K₃.

If all three comparisons fail, report R₁ through F-375c8199e7c2 and return.

The second inner kind branch accepts a match against K₁ or K₂. It reads K₁ first and reads K₂ only if Kind differs from K₁. If both comparisons fail, report R₁ and return.

When an inner selector branch is selected, the other selector branch is not subsequently tried, even if the selected branch rejects the kind.

For stable configuration and representable derived arithmetic, the compatibility rule is:

| Selector | Kinds permitting delegation |
|---|---|
| S | K₁, K₂ − 0x330, or K₃ |
| S − 1 | K₁ or K₂ |

Equality, not an exclusive symbolic classification, controls acceptance. A Kind equal to K₂ is accepted in the S branch if it also equals K₁ or K₃.

**Fallthrough with differing observations:** outer selector acceptance does not force either inner selector comparison to succeed. If the inner comparisons both fail against their respective newly read values, no inner kind reads occur and delegation proceeds. This branch structure is established even though the synchronization or mutation mechanism that could produce differing observations is unspecified.

### Auxiliary acquisition and operation invocation

Every path reaching delegation first calls F-c2d002d7d74c with no arguments. Its 32-bit result is forwarded without validation; zero receives no special treatment.

After that dependency returns, the function reads the object fields used for the operation call. Thus, changes to those fields made by F-c2d002d7d74c can affect the forwarded values. The relative order of the two field reads is not established by the supplied representation.

F-c9bded66abe7 receives thirteen arguments:

| Position | Value |
|---|---|
| 1 | 32-bit field at Object base + 0xAE4 |
| 2 | Payload A |
| 3 | Payload B |
| 4 | Extent A |
| 5 | Extent B |
| 6 | Payload C |
| 7 | Extent A again |
| 8 | Extent B again |
| 9 | 32-bit zero |
| 10 | 32-bit zero |
| 11 | Result from F-c2d002d7d74c |
| 12 | Original Kind input |
| 13 | 32-bit field at Object base + 0x90C |

The duplicated extents and literal zeros are part of the dependency interface. Kind is forwarded unchanged, not replaced with a matching configuration value. Selector is not forwarded.

### Operation result

If F-c9bded66abe7 returns any nonzero 32-bit result, return without reporting. Positive and negative nonzero results have the same effect.

If it returns zero, invoke F-375c8199e7c2 with Object base and the **initially captured R₀**, then return. R₁ is not used for this report.

The retained R₀ is not reread after either dependency. A dependency’s mutation of its backing global storage therefore does not change this selected reporting argument.

There is no visible retry, alternate operation, or local cleanup.

## Side effects

The supplied function contains no explicit writes to global or object memory. Its established effects are conditional reads and dependency calls.

For ordinary returning execution:

| Path | Dependency calls, in order |
|---|---|
| Outer rejection | F-375c8199e7c2 with 0x500 |
| Negative extent after outer acceptance | F-375c8199e7c2 with captured R₀ |
| Inner kind rejection | F-375c8199e7c2 with R₁ |
| Delegation, nonzero result | F-c2d002d7d74c, then F-c9bded66abe7 |
| Delegation, zero result | F-c2d002d7d74c, then F-c9bded66abe7, then F-375c8199e7c2 with captured R₀ |

Each listed dependency is called at most once per invocation. A reporting call occurs exactly once on every ordinary returning path except a nonzero operation result.

Dependencies may modify globals, object memory, or external state. No preservation guarantee is supplied. A zero operation result does not establish absence or rollback of operation side effects.

Invalid memory access, nonreturning dependencies, exceptions, and other dependency-level failure mechanisms have no established local recovery contract.

## Dependencies

| Opaque function ID | Established interaction | Unresolved properties |
|---|---|---|
| F-375c8199e7c2 | Receives Object base and the selected 32-bit reporting value; its result is ignored. | Reporting semantics, object/global mutation, retention, ownership, failure behavior. |
| F-c9bded66abe7 | Receives the thirteen arguments above; zero causes reporting and nonzero suppresses it. | Operation semantics, argument validity requirements, partial effects, retention, ownership, failure behavior. |
| F-c2d002d7d74c | Called without arguments before the object-field reads; supplies one opaque 32-bit operation argument. | Result meaning, lifetime, ownership, memory/global effects, failure behavior. |

None of these dependencies is assumed to preserve configuration or own or release memory.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless stated otherwise, configuration is stable, relevant memory accesses are valid, dependencies return normally, and derived arithmetic is representable.

1. **Outer selector rejection.**  
   If Selector matches neither S nor S − 1, report 0x500. After capturing R₀, perform the two outer S comparisons; do not read any kind configuration or R₁ and do not call either operation-related dependency.

2. **Outer kind rejection takes precedence over negative extents.**  
   With an eligible Selector but Kind matching none of the four outer values, report 0x500 even if an extent is negative. Do not read R₁.

3. **Negative extent after outer acceptance.**  
   If outer eligibility succeeds and either extent is −1, report captured R₀. Do not perform inner configuration checks, acquire the auxiliary value, or invoke the operation.

4. **Zero extents are permitted.**  
   With Selector equal to S, Kind equal to K₁, and both extents zero, acquire the auxiliary value and invoke the operation. All four extent argument positions contain zero.

5. **First-branch compatibility rejection.**  
   With both extents nonnegative, Selector equal to S, and Kind equal to K₂, where K₂ differs from K₁, K₂ − 0x330, and K₃, report R₁ without either operation-related call. If an extent were negative instead, captured R₀ would be selected.

6. **Second-branch compatibility rejection.**  
   With both extents nonnegative, Selector equal to S − 1, and Kind equal to K₃, where K₃ differs from K₁ and K₂, report R₁. A negative extent would instead select captured R₀ before compatibility checking.

7. **Overlapping kind references.**  
   If K₂ equals K₁, a Kind equal to that shared value is compatible with Selector equal to S. An outer K₁ match also prevents the later outer kind reads.

8. **Inner-selector fallthrough.**  
   Suppose Selector is 20, the outer S read is 20, Kind is 1000, K₁ is 100, K₂ is 1000, K₃ is 2000, and both extents are nonnegative. Outer eligibility succeeds through K₂. If both subsequent inner S reads yield 30, neither inner selector comparison matches: 20 equals neither 30 nor 29. Delegation proceeds without inner kind reads. This is conditional on those observations, not a claim about a supported concurrency mechanism.

9. **Nonzero operation result.**  
   After delegation, either −1 or 1 suppresses reporting. Neither result is exposed as this function’s return value.

10. **Captured default survives dependency mutation.**  
    If initial R₀ is P, a dependency changes its storage to Q, and the operation returns zero, report P.

11. **Auxiliary dependency changes object fields.**  
    If F-c2d002d7d74c changes the two relevant object fields before returning, the operation receives the values read afterward, not values captured before that call.

## Uncertainties

- **documented_only — Arithmetic boundaries:** S − 1 and K₂ − 816 involve 32-bit signed values. Overflow semantics are not established by the supplied representation. Neither wraparound nor a specific failure behavior is promised at signed boundaries.
- **documented_only — Configuration synchronization:** The conditional reference sequence is visible, but volatility, atomicity, synchronization, and supported concurrent mutation are unknown. Neither a coherent snapshot nor any particular differing-read outcome is guaranteed.
- **documented_only — Physical reads:** The stated sequence describes source-visible conditional evaluations. Exact machine-load counts and compiler transformations are not established.
- **documented_only — Object access:** Address arithmetic at representation boundaries, alignment requirements, and invalid-access behavior are unspecified. The relative order of the two object-field reads is also unresolved.
- **documented_only — Dependency contracts:** Ownership, retention, global preservation, failure mechanisms, and partial effects are unavailable.
- **documented_only — Domain semantics:** Payloads, fields, configuration values, and reporting values have no established domain-specific meanings.
- **Inferred intent:** The structure resembles eligibility validation followed by an operation and failure reporting. That interpretation supplies no stronger success, rollback, or resource-management guarantee.

## Evidence

Established static evidence consists of the supplied single-function representation and opaque dependency mapping.

It supports the eight argument roles, six configuration storage locations, short-circuit comparison structure, fresh inner configuration references, inner-selector fallthrough, signed extent checks, two object-field offsets, dependency ordering and argument layout, captured-default reporting, and zero-versus-nonzero result handling.

No dependency bodies, configuration contents, object definition, machine instruction listing, runtime traces, or synchronization guarantees were supplied. No runtime tests have been performed.
