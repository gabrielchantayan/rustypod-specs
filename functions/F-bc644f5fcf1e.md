<!-- Trusted provenance: F-bc644f5fcf1e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bc644f5fcf1e

## Purpose

F-bc644f5fcf1e is a seven-argument routine with a signed 32-bit return value. Its directly visible behavior consists of state-dependent argument capture, argument predicates, arithmetic, global-word reads, calls to two dependencies, a conditional four-byte caller-memory read, and a conditional four-byte caller-memory write.

The supplied representation does not establish an application-level operation such as allocation, copying, encoding, or release. Neither the return value nor the output-word write is established to indicate success.

Control flow depends on unavailable global values and secondary results from F-870c4fcf7295. Consequently, this contract distinguishes:

- **Established static evidence:** operations, data dependencies, and ordering visible in the supplied representation.
- **Inferred intent:** interpretations suggested by the structure but not established.
- **documented_only:** unresolved behavior that must not be replaced with a guessed conventional API.

Individual control regions have definite local ordering. Their reachability, repetition, and overall execution order are not fully established.

## Interface

The routine accepts seven positional arguments:

| Position | Contract name | Representation | Directly visible role |
|---|---|---|---|
| 1 | Input word | Unsigned 32-bit scalar | Captured into arithmetic state used in deriving a downstream argument. |
| 2 | Output address | Pointer to a four-byte object | Tested against null; destination of a conditional four-byte store. |
| 3 | Mode word | Signed 32-bit scalar | Captured and tested for nonzero. |
| 4 | Alternate-mode word | Signed 32-bit scalar | Captured and tested for zero. |
| 5 | Extent word | Signed 32-bit scalar | Captured, tested for nonzero, and supplied as the fourth argument to F-8e2a731476b8. |
| 6 | Count address | Pointer to a signed 32-bit object | Tested against null, conditionally dereferenced, and supplied as the third argument to F-8e2a731476b8. |
| 7 | Context word | Signed 32-bit scalar | Captured, tested for nonzero, and supplied as the first argument to F-8e2a731476b8. |

These names identify observable roles, not recovered application semantics. In particular, “extent” does not establish a byte count, and “context” does not establish a handle or pointer.

The pointee accesses are four bytes wide. That fact does not independently establish pointer width, endianness, alignment requirements, or a larger object layout.

The return value comes from a signed 32-bit result slot initialized to zero. A completion-calculation region can replace that slot using a working arithmetic value, a global-sourced multiplier, and a global-sourced additive term. No result units, status convention, or relationship to the output word is established.

### Preconditions and ownership

Requirements are conditional on the executed path:

- Reaching the direct count read requires the retained count address to designate a readable four-byte object.
- Reaching the direct output store requires the retained output address to designate a writable four-byte object.
- Reaching a dependency requires its arguments to satisfy that dependency’s independently unresolved requirements.
- Accessed objects must remain valid for the actual accesses.

No direct alignment, bounds, lifetime, or access-failure check is visible. Null predicates do not establish a complete null-rejection policy because the transition graph is unresolved.

No ownership transfer, allocation, release, or retention policy is established. In particular, passing the count address to F-8e2a731476b8 does not prove that the dependency merely borrows it or leaves its object unchanged.

## Behavior

### Entry snapshots and initialization

The routine initializes its return slot, arithmetic state, retained argument slots, and predicates to zero, null, or false. Original arguments are not all captured into those retained slots at entry.

Six global words are read before dispatch begins. Their read order is established:

1. A seed later used to form the argument-capture region’s next state.
2. An initial arithmetic-helper seed.
3. A completion-transition seed.
4. A first transition seed.
5. A second transition seed.
6. The initial dispatch-state seed.

The first read precedes the bulk of working-state initialization. The remaining reads are interspersed with initialization in the order above.

These are entry observations, not requests to reread the corresponding globals whenever their values are used. Some retained seeds are subsequently modified or replaced. A dependency’s later change to a source global cannot retroactively change an already captured entry value.

### Every dispatch observes globals again

Each dispatch begins with this ordered sequence:

1. Read a global word into a retained dispatch observation.
2. Narrow the current predicate word to a character-sized value.
3. Read the selector-offset global and combine it with the current unsigned state to choose a control region.

Thus, the dispatch-observation global is read **before** the selector-offset global. The selector offset is not an entry snapshot: it is read again on every subsequent dispatch. The earlier dispatch observation is also refreshed on each dispatch.

There is an important delayed use of the earlier observation. One region makes four successive calls to F-870c4fcf7295 and then copies that dispatch observation into working arithmetic state. The copied value is the one read **before selection of that region**, not a fresh read after those calls. Changes made by those calls to its source global affect a later dispatch observation, not this retained value.

Conversely, a dependency’s changes to the selector-offset global can affect the next dispatch, because the next dispatch reads that global again.

These ordering facts do not require any assumption about dependency preservation.

### Argument-capture region

Argument capture has a definite split around one call to F-870c4fcf7295.

Before the call, the region:

1. Retains the original count address.
2. Establishes whether that address is non-null.
3. Establishes whether the original extent word is nonzero.
4. Retains the extent word.
5. Retains the context word.
6. Computes a next-state value from the first entry snapshot and a newly read global.
7. Reads a separate helper-input global and calls F-870c4fcf7295 using the current helper seed minus that observation, with second argument 2,147,483,647.

After normal return, it performs further global-dependent updates in this order:

1. Updates the first retained transition seed using a newly read global.
2. Reads a global into one arithmetic working value.
3. Reads another global into a second arithmetic working value.
4. Updates the second retained transition seed using another global.
5. Updates the completion-transition seed using another global.
6. Retains the helper’s secondary result as the new helper seed.
7. Captures the output address.
8. Captures the mode word.
9. Captures the alternate-mode word.
10. Captures the input word.

The next-state value formed before the call remains based on its pre-call observations. The five global reads listed after the call remain sensitive to changes the dependency may have made.

This region does not itself read either caller-supplied object. Its position in the complete execution is unresolved; it cannot be declared an unconditional first region.

### Argument predicates and state selection

Other regions establish or use the following distinctions:

- Output address: null versus non-null.
- Mode word: zero versus nonzero.
- Context word: zero versus nonzero.
- Alternate-mode word: zero versus nonzero, with the zero case represented by a true selection value.
- Count-derived arithmetic: negative versus nonnegative intermediate result.

The output-address, mode, and context predicates are established together before two helper calls. Subsequent global-dependent state updates occur after the second call.

The alternate-mode predicate is established in another region. That region also updates arithmetic state and a retained transition value before one helper call; additional state-selection globals are read after that call.

No positive-only restriction follows from these predicates. For example, −1 satisfies the extent, mode, and context nonzero tests. The visible predicates do not establish error precedence or prove that an invalid pointer cannot reach an access.

Some transitions select between two candidate states through multiplication by a predicate and its complement. That structure does not make every candidate a valid success or error destination.

### Count read, arithmetic, and three-call sequence

One region directly reads the signed four-byte object at the retained count address.

The loaded word contributes to arithmetic involving global multipliers and offsets, retained state, bitwise operations, and a later subtraction. The region also updates a working value used to construct the second argument of F-8e2a731476b8.

That downstream argument is not established to equal the original input word. The working state initially populated from the input can itself be transformed or replaced on later visits.

A sign-derived selection value is formed from the final subtraction. Under signed 32-bit arithmetic-right-shift behavior, shifting that intermediate right by 31 and negating produces zero for a nonnegative intermediate and one for a negative intermediate. This is not a direct count-versus-extent comparison.

After the direct count read and these calculations, the region makes three successive calls to F-870c4fcf7295:

- The first call’s input uses a global read before the first call.
- The second call’s input uses a distinct global read **after the first call returns**.
- The third call’s input uses another global read **after the second call returns**.

Each call uses 2,147,483,647 as its second argument. The later helper inputs must not be modeled as if all their global operands were captured before the first call.

After the third call returns, the region reads further globals in this statement-level order:

1. A value installed into an output-path transition slot.
2. A value installed into arithmetic state.
3. An operand updating a retained transition word.
4. An operand constructing one candidate next state.
5. An operand constructing the other selection contribution.

The three secondary helper results are then retained in working state before redispatch.

A single visit exposes one direct count read. Repeated visits, and therefore repeated reads, are not ruled out.

### Other helper-call schedules

Additional regions contain helper sequences with distinct observation boundaries:

| Region role | Established ordering |
|---|---|
| Mode-based selection | Two helper calls, each with its own fresh global input operand; transition updates and candidate-state reads follow the second call. |
| Output-address-based selection | Two helper calls; then two global-backed state assignments; then two more helper calls. Each call’s global input operand is read at its call stage. |
| Count-address-based selection | Transition updates precede two helper calls; three global-dependent updates occur between the second and third calls; a fourth call follows the third. |
| Context-based selection | Transition updates precede two helper calls; two global-backed assignments occur between the second and third calls; a fourth call follows the third. |
| Dispatch-observation transfer | One helper call; two global-dependent transition updates; three further helper calls; then transfer of the preselection dispatch observation into arithmetic state. |
| Shared arithmetic transition | A selected region prepares arithmetic operands; one helper call supplies the secondary result used as the next state. |

For each successive call, any global operand read while preparing that call is observed at that point, not at entry. Retained scalar operands remain captured values unless the routine explicitly changes them.

The ordering above describes separate operations. It does not impose an otherwise unspecified ordering between multiple global operands within a single compound arithmetic expression.

### Downstream invocation and output write

A separate region calls F-8e2a731476b8 with:

1. The retained context word.
2. A 32-bit expression combining previously computed working state with a global multiplier and additive term.
3. The retained count address.
4. The retained extent word.

The expression’s application-level meaning is unresolved.

If the dependency returns normally, the next statement reads an output-value global and writes that four-byte value through the retained output address. A later global read then participates in selecting the next state.

The observable sequence is therefore:

1. Prepare the downstream arguments.
2. Invoke F-8e2a731476b8.
3. After return, read the output-value global.
4. Store that value through the output address.
5. Read the subsequent transition operand.
6. Redispatch, including fresh dispatch-observation and selector-offset reads.

There is no visible conventional return-value check between the call and store, and no immediate null recheck. The stored value is not visibly a returned dependency result.

### Completion and widths

A return region returns the current result slot. The completion-calculation region updates that slot and continues dispatching rather than directly returning. Repetition can therefore replace an earlier computed result.

Arithmetic must not be interpreted as unbounded integer mathematics:

- Scalar words and direct pointee accesses are 32 bits.
- The input and dispatch state have unsigned representations.
- Several arithmetic operands and the count load have signed representations.
- Unsigned 32-bit addition and multiplication, where established by the represented types, use modulo-\(2^{32}\) word results.
- Signed operations, mixed conversions, and narrowing require separate attention.
- Character-sized conversion occurs before dispatch selection and is also used in some state-selection paths.
- The fixed helper argument is \(2^{31}-1\), not an all-ones word.

Exact machine treatment of signed overflow, implementation-dependent conversions, character signedness, and signed right shift remains **documented_only** where the supplied representation does not settle it.

## Side effects

Directly visible effects are global-word reads, conditional count-object reads, dependency calls, and conditional output-object writes.

The global observations are not one immutable snapshot. They comprise entry captures, per-dispatch reads, call-input reads, inter-call reads, and post-call reads. Already captured values and future observations must be distinguished.

No direct store through the count address is visible. F-8e2a731476b8 nevertheless receives that address and may modify or retain it; neither behavior is established or excluded.

If both pointer arguments designate the same four-byte object, the direct output store overwrites that object when reached. No alias rejection is visible. Dependency compatibility and subsequent count-read results remain unresolved.

No direct allocation, release, logging, synchronization, or persistent pointer retention is visible in this body. That does not constrain dependency effects.

There is no established fault-to-status conversion, exception recovery, or recovery from nonlocal dependency exits. Termination is unresolved.

## Dependencies

### F-870c4fcf7295

**Established static evidence:**

- Receives two visible arguments.
- Its first argument is an internally derived word.
- Its second argument is always 2,147,483,647 at the visible call sites.
- Supplies a secondary machine-level result used in arithmetic and state selection.
- Its conventional primary return value is not visibly consumed.
- Occurs in single-call and ordered multi-call regions described above.

**Inferred intent:** A numerical helper is plausible. Division, remainder, normalization, clamping, and randomness are not established.

**documented_only:** Result semantics, errors, determinism, memory effects, ownership effects, global preservation, and termination.

### F-8e2a731476b8

**Established static evidence:**

- Receives the four arguments specified above.
- Receives the retained caller-supplied count address, not an address of a copied count object.
- Is followed on normal return by the direct output-value global read and output store.
- Has no visible conventional result check at this call site.

**documented_only:** Argument semantics, accessible extents, modifications, retention, ownership, errors, global effects, and termination.

## Acceptance examples

These are conditional obligations, not executed tests or reachability claims.

### Entry snapshot versus later mutation

**Condition:** The first entry seed is observed as one value, its source global later changes, and argument capture is reached.

**Required:** The capture-region next-state calculation uses the retained entry observation together with its newly read operand. It does not reread the entry seed’s source merely because that source changed.

### Delayed dispatch-observation transfer

**Condition:** A dispatch observes value A before reading its selector offset and enters the four-call transfer region. A helper changes the observation source global to B.

**Required:** The later transfer into arithmetic state uses A. A subsequent dispatch reads the source global again.

### Split argument capture

**Condition:** The capture region reaches its helper call.

**Required:** Count address, extent, context, and the two corresponding predicates have already been captured or established. Output address, mode, alternate mode, and input capture follow the helper and the specified post-call updates.

### Inter-call global changes

**Condition:** The count-read region executes, and its first helper changes the global used to prepare the second helper’s input.

**Required:** The second input uses the post-first-call observation. Post-third-call state-update reads occur only after the third call returns.

### Downstream output observation

**Condition:** F-8e2a731476b8 is reached, changes the output-value global, and returns normally.

**Required:** The subsequent direct output store uses the value read after return, not a pre-call snapshot.

### Negative extent

**Condition:** Argument capture receives extent −1.

**Required:** Its extent predicate is true. If the downstream region is reached without a later recapture, the fourth downstream argument is −1.

**Not established:** Dependency acceptance of that value.

### Null and aliased addresses

**Condition:** The retained output address is null.

**Required:** Its explicit non-null predicate is false.

**Not established:** Safe rejection or a particular result.

**Condition:** Both addresses designate the same valid four-byte object and the downstream region completes its call.

**Required:** The dependency receives that address before the direct output store overwrites the object.

**Not established:** Dependency permission for the alias or the value observed on a later count read.

### Initialized result

**Condition:** The return region is reached without replacement of the initialized result slot.

**Required:** The result is zero.

**Not established:** Reachability of that condition or a success interpretation.

## Uncertainties

The following remain **documented_only**:

- Global values, initial selected region, and complete reachable transition graph.
- Loop bounds, repetition counts, and termination.
- Whether state selection protects every pointer access.
- Concrete error outcomes and return-value meaning.
- Concrete downstream arithmetic arguments and output values.
- Dependency semantics, preservation guarantees, and ownership behavior.
- Machine arithmetic details not fixed by the supplied representation.
- Object layout beyond the direct four-byte accesses.
- Thread safety, reentrancy, and concurrent modification behavior.
- Recovery after invalid accesses or abnormal dependency exits.

The structure is consistent with transformed control flow. That inference does not justify replacing the visible conditional behavior with a guessed validation wrapper.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping. The prior draft and feedback are not independent behavioral evidence.

Established evidence includes argument representations, direct access widths, explicit predicates, dependency argument positions, initialization, and ordering across separate operations. It also establishes the split capture sequence, entry snapshots, repeated dispatch reads, delayed use of a dispatch observation, inter-call helper-input reads, and post-call state-update reads.

Global contents, dependency implementations, machine-level arithmetic confirmation, and execution traces were not supplied.

No runtime tests have been performed. Acceptance examples describe obligations for later verification, not observed outcomes.
