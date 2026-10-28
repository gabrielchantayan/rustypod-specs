<!-- Trusted provenance: F-fb55bf51c09c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fb55bf51c09c

## Purpose

F-fb55bf51c09c performs a bounded sequence of observations to determine whether an unsigned target value falls within a half-open interval. A dependency receives an address associated with the interval state and may update that state before each decision.

The function obtains an adjustment value once, then makes at most three observation calls. It returns one of two bitwise-encoded results:

- If an observation places the target inside the interval, the result is the target minus the saved adjustment, with bit 1 forced to one.
- If all three observations place the target outside the interval, the result is the unchanged target, with bit 0 forced to one.

These are descriptions of established arithmetic and control flow, not proof that the return value implements a disjoint success/error tagging scheme. Either result can have both low bits set.

The broader meaning of the target, interval, adjustment, and repeated observations is unknown. Descriptions such as “address lookup,” “retry,” or “relocation” would be inferred intent rather than established behavior.

## Interface

The interface has four inputs and one unsigned result. Under the supplied source representation, the relevant scalar values and arithmetic are 32-bit.

| Input | Representation | Established use |
|---|---|---|
| Context value | Uninterpreted 32-bit value | Passed twice, unchanged, as the first two arguments to F-918901543e8e. |
| Target | Unsigned 32-bit integer | Compared against the observed interval and used to construct the result. |
| Initial lower bound | Unsigned 32-bit integer | Initializes the first word of local interval state before either dependency is invoked. |
| Initial extent | Signed 32-bit integer | Initializes the second word of local interval state. Its bit pattern participates in unsigned endpoint arithmetic. |

The function also creates an internal rejection counter, initially zero, and saves the value returned by F-4ec7d1520840.

### Local state layout

The supplied representation places two four-byte interval-state words consecutively:

| Relative byte offset | Width | Initial content |
|---|---:|---|
| 0 | 4 bytes | Initial lower bound |
| 4 | 4 bytes | Initial extent |

F-918901543e8e receives the address of the first word, not two separately supplied state addresses.

The later interval calculation reads both words. The source does not establish the dependency’s permitted write footprint. In particular, adjacency alone does not establish that F-918901543e8e is authorized to modify the second word, nor that it leaves that word unchanged. The normal interval behavior described below uses whatever values are present when the caller evaluates the condition.

The local state has invocation-local lifetime. No pointer to it is returned by F-fb55bf51c09c. Any dependency retention or later use of the supplied address is outside the established contract.

### Result representation

The return value is unsigned and 32-bit:

- **Accepted observation:** subtract the saved adjustment from the target using modulo-\(2^{32}\) arithmetic, then force bit 1 to one.
- **Three rejected observations:** force bit 0 of the original target to one.

Forcing a bit does not clear any other bit. It does not add a numeric status field separate from the underlying value.

## Behavior

### Initialization and adjustment acquisition

Before calling either dependency, the function initializes:

1. The rejection counter to zero.
2. The local lower-bound word from the third input.
3. The local extent word from the fourth input.

It then calls F-4ec7d1520840 once, with no explicit arguments, and saves its result as a signed 32-bit adjustment.

That call occurs even when the first interval observation will succeed. The adjustment is not obtained again after a rejected observation.

The displayed source establishes no additional relationship between this dependency and the initial interval state. Any global-state effects of the dependency remain unknown.

### Observation order

Each observation begins by calling F-918901543e8e with these arguments, in order:

1. The context value.
2. The same context value again.
3. The address of the local lower-bound word.

The dependency’s return value is ignored. Its ordinary return value does not directly select acceptance, rejection, or an error path.

Only after the call returns does F-fb55bf51c09c evaluate interval membership. Consequently, even if the initial interval already contains the target, the dependency is still called before acceptance can occur.

For each decision, let:

- \(T\) be the unsigned target;
- \(L\) be the lower-bound word after the observation call;
- \(S\) be the extent word’s 32-bit bit pattern after that call;
- \(E = (L + S) \bmod 2^{32}\).

The observation is accepted exactly when both unsigned comparisons hold:

\[
L \le T \quad\text{and}\quad T < E.
\]

The lower bound is inclusive. The computed endpoint is exclusive.

### Condition evaluation and reads

The rejection condition is evaluated in this order:

1. Read the lower-bound word and determine whether the target is below it.
2. If the target is below the lower bound, reject without evaluating the endpoint expression.
3. Otherwise, evaluate the endpoint expression using the lower-bound and extent words.
4. Reject if the computed endpoint is less than or equal to the target.

Thus the extent is not needed on a path rejected by the first comparison. The lower-bound expression appears again in the endpoint calculation; the supplied source does not establish a single atomic snapshot spanning all interval-state reads.

These are source-level evaluation properties. Exact machine-load counts, compiler load reuse, bus ordering, and behavior under concurrent or asynchronous mutation are not established.

### Rejection budget

A rejected observation increments the internal counter by one. The function returns the rejection result when that counter becomes greater than two.

Accordingly:

- There is at least one observation call on every normal-return path.
- There are at most three observation calls.
- Acceptance on the first, second, or third observation returns immediately.
- Rejection on the first or second observation leads to another observation.
- Rejection on the third observation returns immediately, with no fourth call.

The counter cannot overflow along these normal paths: it only reaches one, two, or three.

### Unsigned interval arithmetic

The lower bound is unsigned. Although the extent word is represented as signed, the endpoint addition uses unsigned 32-bit arithmetic. A negative extent is therefore not handled as a separately validated error. Its bit pattern contributes modulo \(2^{32}\).

There is no explicit validation for zero extents, negative extents, endpoint overflow, or a malformed interval.

The comparison does **not** implement a circular interval that crosses the unsigned maximum. If the computed endpoint is less than or equal to the lower bound, no unsigned target can satisfy both acceptance comparisons for that observation.

Consequences include:

- A zero extent produces an empty interval.
- A positive extent that wraps the endpoint below the lower bound produces an empty interval, rather than a two-part wrapped interval.
- A negative signed extent may produce an endpoint greater than the lower bound after unsigned arithmetic. Such an observation can accept targets. The sign of the extent alone does not determine the outcome.

### Result construction

On acceptance, the saved adjustment is converted as required by the unsigned subtraction. The difference wraps modulo \(2^{32}\), including when the adjustment is greater than the target. Bit 1 is then forced to one.

After three rejections, the adjustment has no effect on the returned value. The result is the original target with bit 0 forced to one.

Neither result is guaranteed to preserve the numeric target or adjusted target exactly. Neither result is guaranteed to identify its branch uniquely through the low bits.

## Side effects

The function directly initializes its local interval-state words, updates its local rejection counter, and invokes its two dependencies. No direct write through the context value is visible.

Potential externally visible effects are dependency-mediated:

- F-4ec7d1520840 is called once before any observation.
- F-918901543e8e is called one to three times.
- F-918901543e8e receives access to invocation-local storage.
- Either dependency may have additional effects not described by the supplied evidence.

There is no visible allocation, deallocation, ownership transfer, or resource-release operation in F-fb55bf51c09c itself. That does not establish that either dependency avoids allocation, release, global mutation, or other external effects.

If a dependency does not return normally, subsequent observations and the normal result construction are not guaranteed. The function contains no visible handling for dependency faults or exceptional exits.

## Dependencies

| Opaque ID | Established interaction | Unknowns |
|---|---|---|
| F-4ec7d1520840 | Called once, without explicit arguments; its return is saved as the adjustment. | Meaning of the result; external-state reads and writes; allocation, release, failure, and termination behavior. |
| F-918901543e8e | Called before every membership decision; receives the context twice and an address to the first local state word; its return is discarded. | Meaning of the context; required context validity; write footprint; whether it reads initial state; treatment of the second word; pointer retention; external effects and failure behavior. |

No dependency is assumed to preserve globals or to own or release memory.

Normal-return behavior requires both dependencies to return and their interactions with storage to remain valid. Further preconditions cannot be established without their contracts.

## Acceptance examples

The following examples are conditional arithmetic and control-flow expectations. They are not runtime observations. Each assumes the dependencies return normally and leave the stated interval values available for the corresponding decision.

1. **Lower boundary included.**  
   With target 100, observed lower bound 100, extent 20, and saved adjustment 10, the first observation accepts. The difference is 90; forcing bit 1 leaves the result at 90. There is one observation call.

2. **Upper boundary excluded.**  
   With target 120, lower bound 100, and extent 20 on all three observations, each observation rejects because the endpoint equals the target. The result is 121. There are three observation calls.

3. **Acceptance on the final permitted observation.**  
   For target 50 and saved adjustment 4, suppose the observed intervals are first \([60,70)\), then \([0,10)\), then \([40,60)\). The first two reject and the third accepts. The result is 46. The third observation is not rejected merely because two earlier observations failed.

4. **Zero extent.**  
   With target 100, lower bound 100, and extent zero on every observation, all three reject. The result is 101.

5. **Wrapped endpoint is not a circular range.**  
   With lower bound 4,294,967,280 and extent 32, the endpoint is 16. No target can be both at least the lower bound and below 16. If this state persists for three observations, the rejection result is returned.

6. **Negative extent can still accept.**  
   With target 100, lower bound 0, and extent −1, the computed unsigned endpoint is 4,294,967,295. The observation accepts. With adjustment zero, the result is 102 after bit 1 is forced to one.

7. **Adjustment subtraction wraps.**  
   With target 1, observed interval \([0,2)\), and adjustment 3, acceptance produces a difference of 4,294,967,294. Bit 1 is already set, so that is also the returned value.

8. **Return bits do not uniquely encode the branch.**  
   An accepted target of 1 with adjustment zero returns 3. Three rejections for target 3 also return 3. A consumer cannot classify these outcomes from that result alone.

## Uncertainties

The following remain **documented_only**:

- The semantic role of every input and of the adjustment.
- The concrete ABI beyond the supplied 32-bit scalar representation.
- Whether the context is an integer, handle, address-like value, or another encoding.
- The dependencies’ contracts, including permitted memory accesses and ownership effects.
- Whether the second local state word is intended to be an output of F-918901543e8e.
- Thread safety, reentrancy, atomicity, and asynchronous mutation behavior.
- Whether callers distinguish acceptance from rejection by additional context.
- The intended interpretation of negative extents and wrapped endpoints.
- Dependency failure and nontermination behavior.

The apparent interval-observation purpose is an inference from comparisons and repeated calls. It is not a verified domain-level description.

## Evidence

Established static evidence consists solely of the supplied function representation: its four inputs, local initialization, dependency call order, short-circuit condition, three-rejection limit, and two return expressions.

The interval rule, modulo arithmetic, call counts, and conditional examples above are derived from that representation. Dependency behavior has not been supplied and is not inferred from function identifiers.

No runtime tests have been performed. No implementation files, external dependency definitions, or prior behavioral documents were consulted.
