<!-- Trusted provenance: F-e9f8fbd977d4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e9f8fbd977d4

## Purpose

F-e9f8fbd977d4 specifies a sequence of address-derived memory writes, thirteen result-processing stages, and ordered dependency invocations. The represented path ends at an invalid-instruction boundary rather than an established normal return.

The observable contract is the ordering and content of those operations. A higher-level application purpose is not established. Repeated writes, low destination addresses, and repeated clearing of a global location could suggest hardware-facing activity, but that interpretation is **inferred intent**, not required behavior.

This specification describes the supplied static representation conditionally. The supplied material reports invalid instruction data and overlapping global symbols; it does not independently establish that the entire represented path is executable.

## Interface

### Inputs and result

The displayed interface declares no explicit parameters and no result. Nevertheless, four incoming machine-state quantities are used without initialization within the represented body:

| Logical input | Width and interpretation | Uses |
|---|---|---|
| Entry address | 32-bit address bit pattern | Initial halfword destinations and payload |
| Entry base | 32-bit address bit pattern | Initial word-store destination and the word-store destinations through stage 4 |
| Entry word | 32-bit unsigned bit pattern | Word-store payloads, derived destinations, and selected call arguments |
| Entry continuation | 32-bit address bit pattern | First indirect-call target |

These are descriptive input names, not a recovered public calling convention. Their physical placement, caller obligations, and preservation across calls remain **documented_only**.

No returned value is established. If every preceding operation completes, the represented path reaches the invalid-instruction boundary described below.

### Widths and arithmetic

The contract uses these bit-level interpretations:

- Halfwords contain 16 bits.
- Words and addresses contain 32 bits.
- Each collector result contains 64 bits, divided into low and high 32-bit words.
- Unsigned right shifts insert zeroes.
- Narrowing to 16 bits retains the low 16 bits.
- Left shifts, multiplication, addition, and subtraction used to describe 32-bit quantities retain the low 32 bits.

These conventions express the represented bit patterns. They do not establish portable language-level semantics for displayed signed arithmetic, nor do they prove that wrapped addresses are usable on the actual platform.

Byte order is unknown. Memory effects are specified by destination, width, and numeric bit pattern rather than byte sequence.

### Preconditions and validity

To complete the represented sequence:

- Each destination must permit the specified access width.
- Each dependency target must be callable under the actual instruction-set and calling-convention rules.
- Every dependency preceding a later operation must return control sufficiently for that operation to execute.

There are no visible checks for null addresses, alignment, mapping, destination extent, arithmetic overflow, target validity, or dependency success.

No buffer size, object layout beyond the stated accesses, allocation contract, or ownership transfer is established.

## Behavior

### Initial effects

Before the first dependency call, the function performs four memory operations in order:

1. A halfword store at the entry address, containing that address’s low 16 bits.
2. An identical halfword store at the same address.
3. A halfword store at twice the entry address, containing the same payload.
4. A word store at entry base plus `0x14`, containing the entry word.

Doubling the entry address means doubling its numeric address bit pattern, with 32-bit truncation. It is not an array-index operation. The two identical writes are separate represented effects.

F-000000000001 is then invoked at entry continuation plus `0xFFE`. No explicit arguments are displayed, and no returned value is explicitly consumed.

### Collector results and halfword formats

Thirteen stages follow. Each begins with a collector invocation whose returned quantity is treated as 64 bits. Each stage derives a halfword destination and payload from the low result word, then performs two consecutive identical halfword stores.

The high result word does not affect either halfword format. Its later use as a call argument is stage-dependent.

**Format A**

- Destination: low result shifted right by 8.
- Payload: bits 8 through 23 of the low result.
- Effect: two 16-bit stores of that payload at the destination.

Although the displayed payload calculation shifts the full 64-bit result, subsequent narrowing excludes all bits from the high word.

**Format B**

- Destination: low result masked with `0x07FFFFFF`, then shifted right by 3.
- Payload: low result shifted left by 5 with 32-bit truncation, then shifted right by 8, then narrowed to 16 bits.
- Equivalently, the payload is bits 3 through 18 of the low result.
- Effect: two 16-bit stores of that payload at the destination.

For Format B, destination formation retains original bits 3 through 26, whereas the payload retains only original bits 3 through 18. Destination and payload therefore need not have the same numeric value.

Both formats produce destinations between zero and `0x00FFFFFF`, inclusive. Neither guarantees alignment.

### Stage schedule

The following derived quantities are used:

- **Scaled base:** entry word multiplied by 16, truncated to 32 bits.
- **Shifted word:** entry word shifted left by 8, truncated to 32 bits.
- **Small address:** entry word masked with `0x0FFFFFFF`, then shifted right by 18.
- **Token:** the stage-specific numeric value listed below.
- **Clear:** a direct assignment of zero to the distinguished global location.

The global clear’s width is unresolved; it must not be treated as a known 32-bit store.

Except for stage 4, each stage proceeds in this order: collector call, halfword pair, global clear, scheduled word store if present, and following dependency call.

| Stage | Collector | Format | Effects after halfword pair, before following call | Token | Following dependency and explicit arguments |
|---|---|---|---|---|---|
| 1 | F-000000000002 | A | Clear; store entry word at entry base + `0x14` | `0x0890FEDD` | F-000000000003: zero, high result, token, entry word shifted left by 4 |
| 2 | F-000000000004 | A | Clear; store entry word at entry base + `0x14` | `0x0890FF45` | F-000000000005: zero, high result, token, entry word shifted left by 4 |
| 3 | F-000000000006 | A | Clear; store entry word at entry base + `0x14` | `0x0890FFAD` | F-000000000007: zero, high result, token |
| 4 | F-000000000008 | A | Additional halfword group below; store entry word at entry base + `0x14`; no clear | `0x08910015` | F-000000000009: small address, high result, small address + token |
| 5 | F-000000000010 | A | Clear; store entry word at scaled base + `0x14` | `0x0891007D` | F-000000000011: zero, high result, token |
| 6 | F-000000000012 | B | Clear; store entry word at scaled base + `0x14` | `0x089100E5` | F-000000000013: zero, high result, token |
| 7 | F-000000000014 | B | Clear; store entry word at scaled base + `0x14` | `0x0891014D` | F-000000000015: zero, high result, token |
| 8 | F-000000000016 | B | Clear; store entry word at scaled base + `0x14` | `0x089101B5` | F-000000000017: zero, high result, token |
| 9 | F-000000000018 | B | Clear; store entry word at scaled base + `0x14` | `0x0891021D` | F-000000000019: zero, high result, token |
| 10 | F-000000000020 | A | Clear; store shifted word at scaled base + `0x14` | `0x08910285` | F-000000000021: zero, high result, token |
| 11 | F-000000000022 | B | Clear; store shifted word at scaled base + `0x14` | `0x089102ED` | F-000000000023: zero, high result, token |
| 12 | F-000000000024 | B | Clear; store shifted word at scaled base + `0x14` | `0x08910355` | F-000000000025: zero, high result, token |
| 13 | F-000000000026 | A | Clear; no scheduled word store | `0x089103BD` | F-000000000027: terminal arguments below |

Every collector has no explicit arguments displayed. This does not establish that its machine-level invocation lacks inherited register or stack inputs.

For stages 1 through 12, the following call’s second explicit argument preserves the high result’s 32-bit bit pattern. **Stage 13 is different:** its following call receives the high result minus `0x264`, not the unchanged high result. This exception applies despite stage 13 using Format A for its halfword pair.

### Stage 4 additional writes

Stage 4 inserts three halfword stores between its collector-derived pair and its scheduled word store:

1. Store small address as a halfword at small address.
2. Repeat the identical store.
3. Store the same halfword at twice small address.

The represented payload calculation narrows a masked, right-shifted entry word to 16 bits before shifting right by two more bits. Because the mask leaves at most twelve bits after the first shift, this equals small address exactly.

Small address ranges from zero through `0x3FF`. The doubled destination ranges from zero through `0x7FE`.

Consequences include:

- The first two destinations may be odd.
- Small address zero makes all three destinations zero.
- These stores may overlap other destinations.
- No direct global clear occurs in stage 4.

The absence of a direct clear does not establish that a callee or an aliasing store leaves the global unchanged.

### Terminal sequence

After stage 13’s halfword pair and global clear, F-000000000027 is invoked with four explicit 32-bit arguments:

1. `0xFFFFFF1C`, equivalent to −228 when interpreted as signed.
2. Stage 13’s high result minus `0x264`, retaining the low 32 bits.
3. `0x0000004B`.
4. Zero.

The target is stage 13’s token plus `0x7F6`.

Next, F-000000000028 is invoked at `0x089104E3` plus `0x7F6`, with no explicit arguments displayed.

F-000000000029, F-000000000030, and F-000000000031 are then invoked in that order, also without explicit arguments displayed.

The represented path ends at an invalid-instruction boundary. A normal return, particular exception, trap handler, process termination, or subsequent execution is not established.

## Side effects

### Direct memory changes

If all preceding operations complete, the representation contains:

| Effect | Count | Composition |
|---|---:|---|
| Halfword stores | 32 | Three initial stores, twenty-six collector-derived stores, three additional stage 4 stores |
| Word stores | 13 | One initial store and one after each of stages 1 through 12 |
| Global zero assignments | 12 | One in every collector stage except stage 4 |

These counts describe represented operations, not distinct addresses or externally observed bus transactions.

The distinguished global’s width, extent, and overlap layout are unresolved. The supplied material reports overlap with smaller symbols at the same address. Neither a word-sized clear nor disjointness from computed destinations is established.

Computed destinations can coincide or partially overlap. Later stores can replace earlier data. Exact partial-overlap byte results require byte order and global layout information that is not supplied.

### Reads, ordering, and visibility

No explicit ordinary-memory load appears in the represented body. Calculations use incoming machine state, constants, and captured call results.

The specified ordering is the ordering of represented stores and calls. A dependency reached after a write group may observe its effects under the actual execution environment. No memory barrier, atomicity guarantee, interrupt exclusion, cache policy, or cross-thread visibility rule is established.

A dependency may modify the distinguished global or any other memory. A direct clear before a call does not guarantee that the global remains zero afterward.

### Ownership and failure

No direct allocation, release, or explicit ownership transfer is shown. This does not establish that dependencies refrain from allocating, freeing, retaining, or otherwise managing memory.

There is no visible status check, retry, rollback, or recovery path. A faulting store or a dependency that does not return can prevent every later operation. Earlier effects are not explicitly undone.

No dependency is assumed to preserve globals, destination memory, or ownership state.

## Dependencies

F-000000000001 through F-000000000031 are **contract-local opaque dependency labels**, assigned in represented call order. They are not independently established identities. Different labels do not prove different underlying implementations.

Their roles are:

- F-000000000001: entry-continuation-derived invocation.
- Even-numbered labels F-000000000002 through F-000000000026: thirteen collectors with captured 64-bit results.
- Odd-numbered labels F-000000000003 through F-000000000025: following calls for stages 1 through 12.
- F-000000000027: stage 13’s following call, with terminal argument adjustment.
- F-000000000028: subsequent indirect invocation.
- F-000000000029 through F-000000000031: final three invocations.

Each collector target equals its stage token plus `0xFFC`. This expresses the supplied target calculations; it does not establish instruction-set mode handling or executability.

For stages 1 through 12, each following-call target equals its stage token plus `0xFFE`. Stage 13 instead uses token plus `0x7F6`.

Only collector results are explicitly consumed as returned data. Other returned values, if any, are not explicitly examined.

Complete calling conventions, hidden inputs, preservation rules, dependency behavior, errors, ownership effects, and return guarantees remain **documented_only**.

## Acceptance examples

These are conditional static examples, not executed tests. They assume the required accesses and invocations complete and use the bit-width interpretation above.

### Initial address-derived stores

Given entry address `0x00001234`, entry base `0x00002000`, and entry word `0x89ABCDEF`:

- The first two stores write halfword `0x1234` at `0x00001234`.
- The third writes halfword `0x1234` at `0x00002468`.
- The fourth writes word `0x89ABCDEF` at `0x00002014`.
- F-000000000001 is invoked only after these four stores.

### Nonterminal Format A result

For **stage 1**, let the collector result have low word `0x12345678` and high word `0x9ABCDEF0`:

- The halfword destination is `0x00123456`.
- Both halfword stores carry `0x3456`.
- After the global clear and scheduled word store, F-000000000003 receives `0x9ABCDEF0` as its second explicit argument.

The unchanged-high-word argument rule also applies to other nonterminal Format A stages, but not to stage 13.

If **stage 13** receives those same result words, its halfword destination and payload are identical, but F-000000000027 receives **`0x9ABCDC8C`** as its second explicit argument because `0x264` is subtracted from the high word.

Changing only the high word leaves Format A’s halfword pair unchanged while potentially changing the following call’s argument.

### Format B result

For a Format B low result of `0xFFFFFFFF`:

- The destination is `0x00FFFFFF`.
- The payload is `0xFFFF`.
- Two halfword stores are represented at that odd address.

Successful unaligned access is a platform prerequisite, not a guarantee.

### Stage 4 low-address group

With entry word `0x0FFFFFFF`, small address is `0x3FF`.

After its collector-derived pair, stage 4 writes halfword `0x03FF` twice at `0x3FF`, then once at `0x7FE`. It then stores the original entry word at entry base plus `0x14`, without a direct global clear.

### Word-payload transition

With entry word `0x10000001`:

- Scaled base is `0x00000010`.
- The scheduled destination in stages 5 through 12 is `0x00000024`.
- Stages 5 through 9 write `0x10000001`.
- Stages 10 through 12 write `0x00000100`.

The destination scaling and later payload shift are distinct operations.

### Terminal subtraction at zero

If stage 13’s high result is zero, F-000000000027 receives, in order:

- `0xFFFFFF1C`
- `0xFFFFFD9C`
- `0x0000004B`
- `0x00000000`

No result check follows that invocation.

## Uncertainties

The following remain **documented_only**:

- Validity and reachability of the entire represented instruction sequence.
- Actual entry convention and preservation of implicit machine-state inputs.
- Distinguished-global width, address extent, and alias layout.
- Byte order, memory mapping, alignment policy, and device-memory behavior.
- Dependency identities, complete calling conventions, and hidden inputs.
- Dependency effects on memory, globals, resources, and ownership.
- Whether each dependency returns or faults.
- Instruction-set interpretation of computed call targets.
- Exact execution semantics of signed arithmetic and wrapped addresses.
- Outcome at the final invalid-instruction boundary.
- Any higher-level protocol or application purpose.

Unknown behavior is not replaced with assumptions that calls succeed, preserve state, or leave memory owned by the caller.

## Evidence

**Established static evidence:** the supplied representation supports the initial write group, thirteen collector-derived halfword pairs, exceptional stage 4 writes, word-store destination and payload transitions, twelve global zero assignments, and ordered dependency invocations.

The terminal invocation explicitly differs from the nonterminal following calls: its second argument subtracts `0x264` from the high result. Format A alone therefore does not imply unchanged high-result forwarding.

**Evidence limitations:** the supplied material reports bad instruction data and overlapping global symbols. This specification preserves the represented operations without independently validating decoding, reachability, or global layout.

**Inferred intent:** possible hardware-facing activity. No such interpretation is required for acceptance.

**Runtime evidence:** none. No runtime tests, dependency executions, memory observations, or termination checks have been performed.
