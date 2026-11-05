<!-- Trusted provenance: F-13ef3c7b2c3d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-13ef3c7b2c3d

## Purpose

F-13ef3c7b2c3d invokes a dependency using a value obtained through its first input. If that dependency returns exactly −1, the function performs an indirect invocation with a four-byte value obtained from an address derived from the previously read value.

The second input selects between two indirect-call sites: one for a value equal to 1 and another for every other value. It does not change the initial dependency call or the address calculation used for the indirect invocation’s argument.

This is a static behavioral contract derived from the supplied representation. It does not establish an application-level purpose such as destruction, notification, resource release, or error recovery. Those interpretations require evidence about the dependencies.

## Interface

### Inputs

The function has two inputs:

| Input | Represented type | Role |
|---|---|---|
| First input | Pointer to an integer | Identifies the storage from which the working value is read. |
| Second input | Integer | Selects the indirect-call site by comparison with exactly 1. |

The supplied representation declares the integer inputs and working values as `int`, but does not independently establish the target’s integer width, pointer width, or calling convention. A 32-bit integer and address model is plausible from the representation, but remains an inference rather than a verified ABI fact.

One width is explicit: the argument fetched for the indirect invocation occupies four bytes.

### Output

The function has no declared return value. It does not return the status obtained from F-9813035c7ce7 or an explicitly represented result from either indirect invocation.

Normal completion therefore provides no direct indication of whether:

- F-9813035c7ce7 returned −1;
- an indirect invocation occurred;
- a dependency changed memory or external state.

Those distinctions can be observable through dependency behavior and memory access, rather than through a return value from this function.

### Memory layout and preconditions

The first input must identify storage readable as the represented integer type when the function performs its initial load. No null-pointer, alignment, or accessibility check is represented.

Let the value read from that storage be the **captured value**. This term describes a value, not a claim that it is always a valid pointer.

Only if F-9813035c7ce7 returns exactly −1 does the function require a further readable location: the address formed by adding four to the captured value. At that address, it reads a four-byte quantity.

The observable layout is therefore limited to:

| Location | Access |
|---|---|
| Storage designated by the first input | One integer-width read |
| Address represented by captured value plus four | One four-byte read, conditional on a −1 dependency result |

The representation does not establish a containing structure, the meaning of either field, byte order, signedness of the four-byte payload, or the validity of other offsets.

The selected indirect-call target must also be valid for the represented invocation if execution reaches that call. Its precise ABI is not established.

## Behavior

### Selection and initial read

The function first distinguishes whether the second input equals exactly 1.

Both resulting paths then read the integer designated by the first input and call F-9813035c7ce7 with two arguments, in this order:

1. The integer constant 1.
2. The captured value.

Thus, changing the second input does not, by itself, change either argument supplied to F-9813035c7ce7.

The first-input storage is read before this dependency executes. The supplied representation contains no later reread of that storage.

### Dependency result other than −1

If F-9813035c7ce7 returns any represented integer value other than −1, the function returns normally, assuming the dependency itself returned normally.

In this case, the function performs neither the four-byte payload read nor either indirect invocation.

This comparison is exact. A negative value other than −1 does not enter the indirect-call path. Zero and positive values likewise bypass that path. No broader success or failure interpretation is established.

### Dependency result equal to −1

If F-9813035c7ce7 returns exactly −1, the function:

- Uses the captured value from before the dependency call.
- Forms an address by adding four to that value.
- Reads four bytes at that address.
- Supplies the resulting four-byte quantity as the argument to the selected indirect invocation.

The second input determines the call site:

| Second input | Indirect-call site |
|---|---|
| Exactly 1 | Site associated with the equality path |
| Any other representable integer | Site associated with the non-equality path |

The representation refers to distinct target locations for these two sites. This does not prove that they contain different callable addresses at runtime.

If the indirect invocation returns normally, F-13ef3c7b2c3d then returns normally. No further operation is represented.

### Ordering and mutation visibility

The following ordering is established by the supplied representation:

1. Select the path using the second input.
2. Read the captured value through the first input.
3. Invoke F-9813035c7ce7.
4. Examine its returned integer.
5. Only for −1, obtain the four-byte payload and perform the selected indirect invocation.
6. Return if the invoked dependencies return normally.

This ordering has two important consequences.

First, changing the first-input storage during F-9813035c7ce7 does not redirect the later payload read. The later address calculation uses the already captured value.

Second, changing memory at captured value plus four during F-9813035c7ce7 can affect the payload subsequently passed to the indirect invocation. The payload is not read before that dependency runs.

The supplied representation does not establish the relative machine-level order of resolving the indirect target and loading the payload. The contract therefore does not promise which of those two accesses occurs first.

### Arithmetic boundaries

The address calculation is represented as integer addition by four, followed by a four-byte memory access. It is not evidence of indexing by four elements.

The exact behavior at an integer or address-space boundary is unresolved. In particular, the supplied material does not independently establish:

- The integer width governing the addition.
- Whether address arithmetic wraps.
- Whether a signed-overflow interpretation is applicable.
- How an invalid resulting address fails.

An acceptance scenario requiring this read must therefore provide an address calculation and accessible four-byte region valid for the actual execution environment. The contract does not invent wrapping or recovery behavior.

## Side effects

### Direct effects

The supplied representation shows no explicit memory writes by F-13ef3c7b2c3d itself.

Its direct observable actions are:

- Reading the first-input storage.
- Calling F-9813035c7ce7.
- Conditionally reading a four-byte payload.
- Conditionally invoking an indirect target.

There is no represented allocation, explicit release, retry, logging operation, status write, or fallback call.

### Dependency effects

No dependency is established to preserve global state, preserve referenced memory, avoid reentrancy, or leave indirect-target storage unchanged.

Consequently, “no explicit memory writes” is not a guarantee that the overall invocation leaves memory unchanged. F-9813035c7ce7 and the indirect targets may have side effects that are absent from the supplied material.

### Ownership and lifetime

Passing the captured value or the four-byte payload to another function does not establish an ownership transfer.

Nothing supplied establishes whether a dependency allocates, retains, releases, invalidates, or merely inspects any associated resource.

For the conditional payload read to be valid, its storage must remain readable after F-9813035c7ce7 returns. If that dependency invalidates the storage, this function contains no represented guard against the resulting invalid access.

### Errors and abnormal completion

The only explicit result-dependent decision is the comparison with −1. No independent error report is produced.

No local handling is represented for an unreadable first input, an invalid derived address, an invalid indirect target, or abnormal dependency termination. Their actual manifestations remain environment- and dependency-dependent. The representation does not justify promising a particular exception, signal, diagnostic, or recovery action.

## Dependencies

### F-9813035c7ce7

This is the sole dependency with a supplied resolved opaque function ID.

Its observable use is:

- Two arguments: constant 1, then the captured value.
- An integer result examined for equality with −1.
- Exactly one invocation per execution that reaches it and returns through the represented flow.
- No retry.

Its semantics, accepted argument domain, side effects, ownership rules, and abnormal-completion behavior are unknown.

### Unresolved indirect dependencies

There are two indirect-call sites, selected by whether the second input equals 1. No resolved opaque function ID is supplied for either target; inventing identities would misrepresent the evidence.

Each site is represented as receiving one four-byte argument read from captured value plus four. No result is consumed afterward.

The two sites may resolve to different functions or to the same function. Target identity, target-storage mutability, calling convention, and return behavior are **documented_only** unknowns.

## Acceptance examples

These examples are conditional static acceptance criteria, not reports of executed tests. They assume valid memory, compatible calling conventions, and normal dependency returns unless stated otherwise.

1. **Non-triggering result with selector 1.**  
   The first-input storage contains a usable captured value, the second input is 1, and F-9813035c7ce7 returns 0. The dependency receives constant 1 and the captured value. Neither the payload location nor an indirect target is used afterward.

2. **Triggering result with selector 1.**  
   F-9813035c7ce7 returns −1. The equality-path indirect target receives the four-byte value read from captured value plus four. The non-equality-path site is not invoked.

3. **Triggering result with another selector.**  
   The second input is 0, −1, or another representable value different from 1, and F-9813035c7ce7 returns −1. The non-equality-path site is selected. The initial dependency still receives constant 1 as its first argument.

4. **A negative result other than −1.**  
   F-9813035c7ce7 returns −2. No indirect invocation occurs, irrespective of the selector.

5. **Mutation of the outer storage.**  
   F-9813035c7ce7 changes the integer designated by the first input and returns −1. The payload address remains based on the value read before the call, not the replacement value.

6. **Mutation of the payload.**  
   F-9813035c7ce7 updates the four bytes at captured value plus four and returns −1. Subject to the execution environment’s memory semantics, the subsequent read obtains the updated payload.

7. **Payload location invalid but unused.**  
   F-9813035c7ce7 returns a value other than −1. This function does not access the derived payload location. Whether the dependency itself accepts the captured value remains a separate precondition.

8. **Equal target addresses.**  
   Both indirect-target locations resolve to the same callable function. The selector still chooses a distinct represented call site, but distinct downstream behavior is not required merely because the selector differs.

## Uncertainties

The following remain **documented_only**:

- Verified integer width, pointer width, byte order, alignment rules, and ABI.
- Address-addition behavior at numeric boundaries.
- The semantic meaning of the captured value and four-byte payload.
- Dependency effects on memory, globals, resources, and target storage.
- Ownership and lifetime obligations beyond the reads shown.
- Exact indirect-target identities and signatures.
- Behavior under concurrency, asynchronous mutation, faults, or nonlocal control transfer.
- Whether the represented indirect invocations faithfully describe the original control transfers.

An inferred intent is “perform an additional action when the initial dependency returns a distinguished status.” Neither the meaning of that status nor the nature of the additional action is established.

## Evidence

The supplied static representation supports the selector partition, initial integer read, call to F-9813035c7ce7 with constant 1 and the captured value, exact comparison against −1, conditional four-byte read at offset four, and selection between two indirect-call sites.

Its annotations report removed unreachable regions and uncertainty in recovering indirect control flow. Those annotations limit confidence in completeness and call semantics; they do not establish additional reachable behavior.

No independent implementation inspection, runtime execution, or tests have been performed. The acceptance examples specify consequences of the represented behavior under stated preconditions, not observed outcomes.
