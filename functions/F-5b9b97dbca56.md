<!-- Trusted provenance: F-5b9b97dbca56; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5b9b97dbca56

## Purpose

F-5b9b97dbca56 coordinates an initial dependency call, obtains a status result, and conditionally invokes one of two follow-up dependencies before obtaining a final status result.

The established behavior is a call-and-status sequence. The supplied static evidence does not establish the application domain, the meaning of the record contents, or whether the dependencies perform validation, conversion, resource management, or another operation.

A null record pointer causes an immediate return of −50 without dependency calls. For a non-null record pointer, the first nonzero status is returned immediately. If the first status is zero, the function selects a follow-up using a character conversion of a local control value, then returns a second status result.

All descriptions of completed calls and returns below assume normal execution, valid required memory, and dependencies that return normally.

## Interface

### Arguments

The recovered interface contains four arguments:

| Argument | Established representation | Observable use |
|---|---|---|
| Record pointer | Pointer to a 16-bit element | Checked for null; otherwise used for a leading-element read and to derive an address one element beyond the start |
| Second argument | 32-bit value | No explicit use in the supplied body |
| Third argument | 32-bit value | No explicit use in the supplied body |
| Initial control value | 32-bit value | Copied into a local 32-bit object whose address is passed to F-ade29969e5e2 |

The second and third arguments are not explicitly forwarded to dependencies. Their presence does not establish a semantic purpose, nor does the body show that they affect branch selection or the returned value.

The local control object is a copy of the fourth argument, not the caller’s original argument storage. The dependency receives the address of that copy.

### Record layout and conversions

The recovered layout establishes only these positions:

| Position | Established use |
|---|---|
| Record start | A 16-bit element read and converted for dependency calls |
| One 16-bit element beyond the start | Address passed to F-ade29969e5e2 and, conditionally, F-5694826ef940 |

In the recovered two-byte-element representation, the second position is at byte offset 2. No length, terminator, nested pointer, or additional field is established. The address passed onward is the address of that region, not a pointer value loaded from it.

The header argument is formed by reading the leading 16-bit element, converting that value to plain character type, and then converting the character result to integer type. This is not shown as a separate byte load.

The conversion must not be replaced unconditionally with a low-eight-bit extraction. Even if plain character type is eight bits wide, conversion of an out-of-range integer to a signed plain character type depends on the target’s conversion rules. Low-eight-bit interpretations in this contract apply only where an example explicitly assumes such conversion behavior.

### Return value

The function returns:

- −50 if the record pointer is null.
- The first result from F-5cecf2e46c23 if that result is nonzero.
- The second result from F-5cecf2e46c23 if the first result is zero.

The recovered interface uses an integer return type consistent with a 32-bit target. The exact ABI and integer representation are not independently established. No return-value normalization, range restriction, or translation is visible.

### Preconditions

For defined non-null execution, the record must support the leading 16-bit access and the pointer arithmetic that forms the following-element address. Alignment, storage lifetime, and accessible extent must meet the target environment’s requirements.

Additional memory requirements belong to the dependencies and are not supplied. In particular, the function does not establish how much storage must be accessible beginning at offset 2.

Any later explicit header read also requires the record to remain valid at that time. The wrapper does not check whether earlier dependencies have invalidated it.

## Behavior

### Null input

The record pointer is checked before the local control copy is initialized.

If the pointer is null:

- The function returns −50.
- It does not read the record.
- It does not call any dependency.
- The other three arguments do not affect this path.

This is the only explicit input-rejection path shown.

### Initial call

For non-null input, the function initializes a local 32-bit control object from the full initial control argument.

It then invokes F-ade29969e5e2 with these positional arguments:

1. The address one 16-bit element beyond the record start.
2. The address of the local control object.
3. The integer obtained by converting the leading 16-bit element through plain character type.

The control initialization precedes the call. The supplied expression structure does not establish a meaningful order between computing the following-element address and reading the header while preparing arguments. The contract therefore imposes no additional ordering between those argument evaluations.

F-ade29969e5e2 receives access to the local control object through its address. Its actual mutation, retention, and memory-access behavior are not supplied.

No return value from this dependency is consumed by F-5b9b97dbca56. This observation does not establish the dependency’s declared return type or its other reporting mechanisms.

### First status result

After the initial dependency returns, F-5cecf2e46c23 is called with no explicit arguments.

If its result is nonzero, the function returns that result immediately. Positive and negative nonzero values receive the same treatment.

On this path:

- No follow-up dependency is called.
- No second status query occurs.
- The local control object is not examined for follow-up selection.
- The wrapper performs no second explicit read of the header.

The result is used as a gate, but the status dependency is not established to be pure or free of other effects.

### Follow-up selection

If the first status result is zero, the function converts the then-current local control value to plain character type and compares the converted value with zero.

The selected value is the local object’s value after both preceding dependencies have returned. It must not be assumed to equal the original fourth argument. F-ade29969e5e2 received its address, and the supplied evidence does not establish what access paths later dependencies might have retained.

The established branch condition is **conversion through plain character type followed by comparison with zero**. It is not an unconditional test of either the complete 32-bit object or its low eight bits.

If the target uses an eight-bit character type and defines the relevant conversions to retain the low eight bits, then upper control bits alone do not determine the branch. Without that conversion rule, this simplification is not established.

### Zero-character follow-up

If the converted control value equals zero, the function invokes F-5694826ef940 with:

1. The address one 16-bit element beyond the record start.
2. An integer obtained by a fresh read of the leading 16-bit element, converted through plain character type.

The earlier header value is not reused. Changes made before this call can therefore affect the header argument it receives.

This second explicit read occurs after the first status query has returned and after the control conversion has selected this path. No additional ordering is established between the argument expressions used to prepare the follow-up call.

No return value from F-5694826ef940 is directly consumed.

### Nonzero-character follow-up

If the converted control value is nonzero, the function invokes F-f3099a09fd7f with no explicit arguments.

This branch performs no second explicit header read in the wrapper. It does not establish that the dependency itself avoids the record or related state.

No return value from F-f3099a09fd7f is directly consumed.

### Final status result

After either follow-up returns, the function calls F-5cecf2e46c23 again and returns that result.

Zero, positive, and negative second results are all returned without further selection. There is no loop, retry, additional follow-up, or third status query.

Thus, normal non-null execution contains one initial operation, one or two status calls, and either zero or one follow-up call.

## Side effects

### Direct accesses and local state

The wrapper explicitly initializes the local 32-bit control copy and stores status results in local result storage.

For non-null input, it reads the leading 16-bit record element before the initial dependency call. It reads that element again only when the first status is zero and the converted control value selects F-5694826ef940.

No direct write through the record pointer is shown. This does not make the record immutable: dependencies receive an interior address and may possess other access paths.

### Dependency-mediated state changes

The static body does not establish that any dependency preserves:

- Global or thread-local state.
- The record header.
- Storage beginning at offset 2.
- The local control object, when reachable.
- Allocation state or ownership relationships.

In particular, F-5cecf2e46c23 is described as a status query because of how its result is used, not because purity is established. Effects during its first invocation precede both branch selection and any second header read.

An ignored direct return value does not imply an ignored operation. Each initial or follow-up dependency may have externally observable effects.

### Ownership and lifetime

The wrapper shows no explicit allocation, release, ownership transfer, reference-count adjustment, or rollback.

The control address passed to F-ade29969e5e2 refers to a local object whose ordinary lifetime ends when F-5b9b97dbca56 returns. No lifetime extension is shown. A dependency retaining that address would not, by retention alone, make access after the local object’s lifetime valid.

Ownership behavior inside dependencies remains unresolved. The body neither proves that they retain caller storage nor proves that they leave it owned and valid.

### Errors and abnormal execution

The only explicit wrapper-generated error value is −50 for null input. The same numerical value could also be returned by a status dependency on a non-null path, with a different call history.

The wrapper does not explicitly validate record length, header meaning, dependency mutation, status-code membership, or follow-up return values.

No recovery for memory faults, conversion-related exceptional behavior, dependency failure to return, or nonlocal exits is shown. The normal-return contract does not promise completion under those conditions.

## Dependencies

| Opaque function ID | Interface used here | Established role |
|---|---|---|
| F-ade29969e5e2 | Following-element address, local 32-bit control address, converted header integer | Initial operation on every normally progressing non-null path |
| F-5cecf2e46c23 | No explicit arguments; integer result consumed | First result gates follow-up; second result, when reached, is returned |
| F-5694826ef940 | Following-element address and freshly converted header integer | Follow-up when the converted local control value is zero |
| F-f3099a09fd7f | No explicit arguments | Follow-up when the converted local control value is nonzero |

These interfaces describe the calls visible here. They do not establish the dependencies’ complete contracts, hidden inputs, effects, or ownership rules.

## Acceptance examples

These are conditional consequences of the static behavior, not executed tests.

### Null record

With a null record pointer and arbitrary remaining arguments, the result is −50. No dependency is called and no record read occurs.

### First status ends processing

With valid non-null input, suppose F-ade29969e5e2 returns normally and the first F-5cecf2e46c23 result is 7.

The function returns 7 without a follow-up or second status query. The same early-return rule applies to a negative nonzero result, including −50.

### Converted control is zero

Suppose the first status result is zero and the local control value converts to character zero at selection time.

F-5694826ef940 is invoked, followed by a second status query. If that query returns −8, the function returns −8.

### Converted control is nonzero

Suppose the first status result is zero and the local control value converts to a nonzero character.

F-f3099a09fd7f is invoked. The wrapper performs no second header read and returns the subsequent status result.

### Conditional truncation examples

Assume both that plain character type is eight bits wide and that the target defines the relevant control conversions by retaining the low eight bits.

Under those assumptions:

- A control value of 256 at selection time converts to zero and selects F-5694826ef940.
- A control value of 257 at selection time converts to a nonzero value and selects F-f3099a09fd7f.

These conclusions are not established by eight-bit character width alone. Without the specified conversion rule, the outcomes for these out-of-range values remain documented_only.

### Header mutation before the second read

Suppose the first header read converts to integer 3. Before the zero-character follow-up, a dependency changes the header so that the later read converts to integer 9.

Provided the record remains valid, F-5694826ef940 receives 9. It does not receive the previously observed 3.

### Conditional header conversion boundary

Assume an eight-bit character target whose conversion of a header value of 255 produces an all-ones character representation.

If plain character type is unsigned, the integer argument is 255. If the target instead defines that conversion to yield the signed character value −1, the integer argument is −1.

Neither the all-ones conversion result nor the signed result follows solely from character width. The required target conversion evidence is absent.

## Uncertainties

### Established static evidence

The supplied body establishes the null check, local control copy, dependency ordering, first-status gate, character-converted branch condition, conditional header reread, and return selection.

It also establishes the absence of explicit use of the second and third arguments and the absence of direct record writes in the wrapper.

### Inferred intent

“Status query” describes the observed use of F-5cecf2e46c23, not a proven dependency contract.

Interpreting the region at offset 2 as application payload is plausible but unnecessary to the contract. No particular format or purpose is established.

### documented_only

The following remain unresolved:

- Exact target ABI, byte order, integer representation, and plain-character width and signedness.
- Out-of-range conversions to plain character type, including whether low-bit truncation applies.
- Domain meanings of the header, control value, and status results.
- Required accessible extent of the following-element region.
- Dependency mutation, retention, allocation, release, and ownership behavior.
- Whether later dependencies can reach the local control object.
- Synchronization and concurrent-access requirements.
- The purpose of the second and third arguments.

The unresolved conversion rules limit numerical predictions for out-of-range values. They do not obscure the established conversion-and-comparison structure.

## Evidence

This contract is based on the supplied static function body and opaque dependency mapping. Recovered type information supports the stated structural widths but does not independently establish the complete target ABI or conversion semantics.

No dependency implementations, authoritative target conversion rules, runtime traces, or execution results were supplied. No runtime tests have been performed. Acceptance examples are conditional specifications, not measured outcomes.
