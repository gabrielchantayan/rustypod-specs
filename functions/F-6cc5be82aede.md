<!-- Trusted provenance: F-6cc5be82aede; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6cc5be82aede

## Purpose

F-6cc5be82aede performs an unconditional repeating sequence of indirect calls and explicit memory accesses. No normal return is represented. Its established contract is an ordered memory-and-call trace, not a recognized application-level service.

The available static evidence does not establish object construction, serialization, allocation, protocol processing, or any other higher-level purpose. Such interpretations are not acceptance requirements.

Each completed iteration establishes state for the next. In particular, the third and fourth explicit inputs become loop-carried values, and hidden address, payload, and code-position state are replaced.

This contract distinguishes established static behavior from conditional machine-arithmetic interpretations and unresolved behavior. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Visible use |
|---|---:|---|
| First | 32 bits, uninterpreted | No explicit use in the represented body. |
| Second | 32 bits, uninterpreted | No explicit use in the represented body. |
| Third | Signed 32-bit integer | Supplies shifted values; replaced near the iteration end by an opaque-call result. |
| Fourth | Signed 32-bit integer | Passed to an opaque dependency; replaced near the iteration end by four times a returned index. |

Lack of explicit use does not establish ABI-level irrelevance of the first two inputs. Hidden register inputs and incomplete dependency signatures remain unresolved.

### Hidden inputs and storage

Initial execution additionally depends on:

- An address base for early selector-byte reads.
- A 32-bit payload for the first word store.
- A code-position value used to derive the first indirect-call target.
- A shared 32-bit memory location sampled during each iteration.
- Dependency results, including represented 32-bit and 64-bit results.
- An additional register-derived argument whose provenance is not established.

No initialization or validation of the initial hidden state is visible.

There is no established caller-owned buffer layout. Addresses arise from integer arithmetic, byte-valued selectors, and hidden state rather than from a validated buffer interface.

### Output, termination, and errors

The represented interface returns no value. If accesses succeed and dependencies return normally, execution continues into another iteration without a termination predicate.

No success status, failure status, explicit error branch, cancellation check, cleanup path, rollback, or exception handler is visible. Faults, nonreturning dependencies, and nonlocal transfers remain outside the established contract.

### Arithmetic and layout notation

The following notation describes values without implying portable language semantics:

- **Lₙ(x):** low 32 bits of a left shift by *n*.
- **Low(x):** low eight bits.
- **T(x):** mask a 32-bit value with `0x3FFFFFFF`, then shift right unsigned by 20. The result has up to ten significant bits.
- **E(x):** Low(T(x)), used for byte-store payloads.
- **U(x):** low eight bits after shifting a 32-bit value right unsigned by 22.
- **Word:** a 32-bit memory access.
- **Selector:** an unsigned eight-bit memory read.

Products and address additions below express the represented arithmetic. Conclusions involving overflow or discarded high bits are conditional on 32-bit machine-word arithmetic. The representation mixes signed shifts, signed multiplication, and unsigned operations; portable signed-overflow behavior is not established.

A selector supplies a numerical address component directly. If a selector equals 7, a byte access at selector plus 17 targets address 24—not the selector-read address plus 24.

Word endianness, address-space width, alignment requirements, and memory permissions are unknown.

## Behavior

### Ordering and dependency notation

The phases below describe one completed iteration. Their order, explicit access widths, retained values, and intervening calls are observable requirements.

Dependency labels F-opaque-01 through F-opaque-06 distinguish unresolved target-derivation forms. Unless another label is specified, a call below uses F-opaque-01. These labels identify classes of call sites, not proven function identities.

A **four-call group** means four consecutive indirect calls, with the fourth supplying the stated result when one is used. Calls without displayed arguments still have unknown complete interfaces and effects. Calls whose results are unused remain part of the trace.

### 1. Initial indexed word store

An initial four-call group supplies an index. The hidden payload is written as a word at:

**4 × index + 20**

A subsequent call receives four times that index.

The first call target depends on the incoming hidden code-position value. Later target derivations use represented code-position anchors. No target validation is visible.

### 2. Early selector updates and retained-selector word store

A four-call group supplies a 32-bit token. A selector is read at hidden base plus 1, and Low(L₂(token)) is written at selector plus 17.

F-opaque-02 receives L₂(token). A following call supplies a 32-bit result, which is passed to another call as its first argument; the second argument is L₃₀(token).

Another four-call group supplies an index. The selector at hidden base plus 1 is read again and retained. E(token) is written at retained selector plus 17, followed by a call receiving L₂(index).

After another four-call group, L₂(the current third input) is written as a word at retained selector plus 20.

That word store uses the saved selector, not a fresh read. Dependencies between the selector read and word store may alter memory without changing this retained address component.

### 3. Shared-value sample and first zero update

Immediately after the preceding word store, the shared 32-bit location is read once and its value retained.

A call receives the current fourth input. A subsequent four-call group supplies a represented 64-bit result. Define **first scaled result** as L₂ of its low 32 bits.

A fresh selector read at hidden base plus 1 directs a byte store of U(the retained shared sample) to selector plus 17.

F-opaque-03 receives the 64-bit result’s high 32 bits. Another call supplies a 32-bit result, which is forwarded as the first argument of a call whose second argument is L₂₈(first scaled result).

A four-call group then supplies an index. A fresh selector read at hidden base plus 1 directs a zero-byte store to selector plus 17, followed by a call receiving L₂(index).

Another four-call group supplies another index. A call receives L₂ of that index twice, in that argument order.

### 4. First retained payload pair

A four-call group supplies a value. Define **first payload** as L₂(value).

The first payload is written as a word at:

**4096 × first scaled result + 20**

A call receives L₁₄(first scaled result). A four-call group then supplies a destination index. The same retained payload is written as a word at:

**4 × destination index + 20**

A call receives four times that destination index. The two destinations may coincide or overlap other accesses.

### 5. Derived-selector updates

A four-call group supplies another represented 64-bit result. Define **second scaled result** as L₂ of its low 32 bits.

A selector is read at:

**16384 × first scaled result + 1**

E(first payload) is written at selector plus 17. F-opaque-03 receives the new 64-bit result’s high word. A following 32-bit result is obtained and forwarded to another call.

Another call supplies a represented 64-bit result; its low and high words are forwarded, in that order, to a subsequent call.

Two further calls occur, the second supplying an index. A selector is read at:

**268435456 × second scaled result + 1**

Zero is written at selector plus 17, followed by a call receiving L₂(index).

A four-call group supplies the **middle index**. A selector is then read at:

**4 × second scaled result + 1**

Zero is written at selector plus 17.

### 6. Third-input-dependent call and second payload

F-opaque-04 receives L₁₀(second scaled result). A following call supplies a 32-bit result. Another call receives that result first and L₂₈(the current third input) second.

A four-call group supplies an index. The selector at 4 × second scaled result + 1 is read again. Zero is written at selector plus 17, followed by a call receiving L₂(index).

After another four-call group, define **second payload** as L₁₄(middle index). It is written as a word at:

**4096 × L₂(middle index) + 20**

F-opaque-05 receives L₁₄(L₂(middle index)). Another opaque call occurs. A subsequent call receives L₂₈(second payload) first and an additional register-derived value second.

Under the conditional 32-bit interpretation, L₂₈(second payload) is zero because the two left shifts discard all contributing bits. This does not eliminate the call or resolve its other argument.

### 7. Second-payload copy and further selector updates

A four-call group supplies a destination index. The second payload is written as a word at 4 × destination index + 20, followed by a call receiving four times that index.

A four-call group supplies the **late base index**. A fresh selector read at 4 × second scaled result + 1 directs a byte store of E(second payload) to selector plus 17.

F-opaque-03 receives L₂₆(middle index). A subsequent 32-bit result is passed as the first argument of a call whose second argument is L₂₈(L₂(late base index)).

Another represented 64-bit result is obtained and forwarded to a call as low word followed by high word.

Two further calls supply an index through the second call. The selector at 4 × second scaled result + 1 is read again. Zero is written at selector plus 17, followed by a call receiving L₂(index).

### 8. Third payload and self-derived payload pairs

A four-call group supplies a value. Define **third payload** as L₂(value). It is written as a word at:

**4096 × L₂(late base index) + 20**

A call receives four arguments, in order:

1. 1024 × L₂(late base index).
2. The same value again.
3. L₁₄(late base index).
4. The third payload.

A four-call group supplies a destination index. The third payload is written as a word at 4 × destination index + 20, followed by a call receiving four times that index.

A four-call group then supplies a new index. Define **first self-derived payload** as four times this index. It is written as a word at 16 × that index + 20. A call receives the payload three times. A subsequent four-call group supplies a destination index; the same payload is written at 4 × destination index + 20, followed by a call receiving four times that destination index.

The same pattern occurs again with a newly returned index:

- **Second self-derived payload** is four times the new index.
- It is written as a word at 16 × that index + 20.
- **Next hidden base** is that index multiplied by 1073741824.
- A call receives the payload three times.
- A four-call group supplies the **retained final destination index**.
- The payload is written at 4 × retained final destination index + 20.
- A call receives four times the retained final destination index.

The retained final destination index is distinct in provenance from the earlier index used to compute the next hidden base. They must not be equated.

### 9. Tail zero pairs and iteration handoff

A four-call group supplies the **tail token**. Two zero-byte stores occur in this order:

1. Next hidden base plus 17.
2. Next hidden base plus 1.

F-opaque-06 receives L₂(tail token). A following call supplies a 32-bit result, which is forwarded to another call.

A four-call group supplies an index. The same two zero-byte stores repeat in the same order, followed by a call receiving L₂(index).

A final four-call group supplies the **final index**. The next fourth input becomes four times the final index.

The last explicit word store writes:

**L₃₁(retained final destination index)**

at:

**16 × final index + 20**

Its payload therefore uses the destination index retained from the end of phase 8—not the index that produced the next hidden base, and not the final index used for this store’s address.

A word is then read at T(tail token) + 20. This lookup uses the full ten-bit T result, without byte truncation.

F-opaque-06 receives the next fourth input. A following call supplies the next third input. The next hidden payload becomes zero, and the hidden code-position state is set to the represented loop-reentry position.

A final call receives the retained lookup word shifted right unsigned by 22 and then left by 28. Execution repeats with the next hidden base established in phase 8.

## Side effects

Each completed iteration contains **27 explicit stores: 13 word stores and 14 byte stores**. It also contains **10 explicit byte reads and two explicit word reads**.

| Phase | Word stores | Byte stores | Byte reads | Word reads |
|---|---:|---:|---:|---:|
| 1 | 1 | 0 | 0 | 0 |
| 2 | 1 | 2 | 2 | 0 |
| 3 | 0 | 2 | 2 | 1 |
| 4 | 2 | 0 | 0 | 0 |
| 5 | 0 | 3 | 3 | 0 |
| 6 | 1 | 1 | 1 | 0 |
| 7 | 1 | 2 | 2 | 0 |
| 8 | 6 | 0 | 0 | 0 |
| 9 | 1 | 4 | 0 | 1 |
| **Total** | **13** | **14** | **10** | **2** |

These counts exclude dependency accesses and indirect-target fetching.

Byte writes affect only the represented byte; they do not establish clearing of adjacent storage. Selector-directed byte-store addresses range numerically from 17 through 272. The saved-selector word-store starting address ranges from 20 through 275, subject to the unresolved memory model.

Aliasing does not relax ordering. Repeated selector reads cannot be replaced with a cached value; retained selectors cannot be replaced with fresh reads. Repeated zero stores remain distinct accesses. Calls may change memory between accesses.

No allocation, deallocation, ownership transfer, retention policy, or lifetime guarantee is established. Dependency calls must not be assumed to preserve globals, own memory, or release memory. No visible rollback restores earlier writes after partial progress.

## Dependencies

All dependencies are unresolved indirect calls.

| Opaque reference | Established target-derivation distinction |
|---|---|
| F-opaque-01 | Prevalent code-position displacement of 4094. |
| F-opaque-02 | Early displacement of 3976. |
| F-opaque-03 | Displacement of 4036; receives high result words or a shifted middle index. |
| F-opaque-04 | Displacement of 4056; receives a shifted second scaled result. |
| F-opaque-05 | Displacement of 3986 following the second-payload store. |
| F-opaque-06 | Tail displacement of 4086. |

The displacement is not an absolute target address. Different anchors can produce different targets within one class. These labels establish neither shared implementations nor complete signatures.

Displayed argument order and result uses are established. Purity, ABI, hidden arguments, register preservation, failure behavior, and memory effects are not.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, dependencies return and accesses succeed.

1. **Selector addressing.** A selector value of 7 directs its associated byte store to address 24, regardless of the address from which the selector was read.

2. **Byte truncation.** An early token of `0x12345678` produces `0xE0` for the first selector-directed byte store.

3. **Extraction widths.** For `0x3FF00000`, T produces 1023 and E produces 255. Used as a tail token, this value selects a word-read starting address of 1043, not 275.

4. **Retained selector.** If phase 2 saves selector 9 and a dependency later changes the underlying selector byte to 12, the third-input-dependent word store still starts at address 29.

5. **Fresh selector.** If the selector location used in phase 6 changes before its repeated read, that read’s new value determines the following zero-byte destination.

6. **Sample timing.** A dependency changing the shared location after phase 3 samples it does not replace the retained sample used by the subsequent byte store.

7. **Final payload provenance.** Under 32-bit arithmetic, suppose the index establishing the next hidden base is 2, the retained final destination index is 3, and the final index is 5. The next hidden base is `0x80000000`. The final word store starts at address 100 and writes `0x80000000`, derived from 3 shifted left by 31. Substituting the earlier index 2 would incorrectly write zero.

8. **Tail ordering and continuation.** Each tail pair writes base plus 17 before base plus 1. Both pairs occur. After successful handoff, the next iteration’s initial word-store payload is zero.

9. **Access failure.** An inaccessible computed address has no established recovery result. Skipping the access or inventing a successful return is not supported.

## Uncertainties

The following remain **documented_only**:

- Actual entry ABI, hidden state initialization, and register-only dependency inputs.
- Dependency identities, signatures, global effects, and memory ownership behavior.
- Whether represented 64-bit results are true wide returns or paired register results.
- Signed-overflow, signed-left-shift, and integer-to-address semantics.
- Address width, alignment, endianness, permissions, and memory mapping.
- Volatility, concurrency, external observation, and access atomicity.
- Fault handling, exceptional exits, and nonlocal control transfer.
- Completeness and correctness of the represented control flow.

The low selector-derived addresses and computed targets do not establish that execution is invalid or reveal a higher-level purpose.

## Evidence

This contract uses only the supplied static representation of F-6cc5be82aede. Established evidence comprises explicit widths, access order, call boundaries, arithmetic expressions, retained values, state assignments, and the unconditional loop.

The final-store provenance follows the retained result used for the last second-self-derived-payload destination in phase 8. Separate later results supply the tail token and final index; neither replaces that retained payload source.

The access totals are obtained by classifying the explicit accesses phase by phase, as shown above. They exclude unknown dependency behavior.

Warnings about an omitted unreachable region and overlapping shared symbols do not establish additional runtime behavior.

No runtime execution, dependency inspection, memory-map validation, or independent binary verification has been performed.
