<!-- Trusted provenance: F-04ee7ef2ef8d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-04ee7ef2ef8d

## Purpose

F-04ee7ef2ef8d modifies a caller-supplied state object in place. Its directly established behavior comprises resetting two words, caching addresses and dependency results, selecting two words through a global root-pointer slot, optionally processing eight records, preparing derived values, conditionally scaling two three-component records, and populating four output records.

The apparent purpose is preparation of working state for later processing. That interpretation is inferred; the application-level meanings of the records, flags, and transformations are not established.

This contract describes the supplied static representation. Dependency behavior is limited to observed arguments and uses of returned values. No runtime tests have been performed.

## Interface

**Input:** one address identifying mutable storage, called **base** below. Object offsets are byte offsets relative to base.

**Output:** no explicit return value. Observable results are memory writes and dependency effects.

**Representations and widths:**

- Addresses and pointer-bearing values are represented as 32-bit quantities.
- A **word** is 32 bits; a **byte** is 8 bits.
- Sign tests, the selected-record index at `0x24C`, mapping indices, and scaling operands use signed 32-bit interpretations.
- Scaling uses a signed 64-bit product and retains product bits 16 through 47.
- Byte order and alignment requirements are not established.

The function performs no explicit null, object-size, selector-range, mapping-bound, or dependency-result validation. Execution requires accessible storage for all reached direct accesses and compatibility with the dependencies’ actual contracts. Embedded addresses and mapping-derived destinations require independent validity; a valid base alone is insufficient.

Principal locations are:

| Location | Established use |
|---|---|
| `0x004`, `0x01C`, `0x028` | Addresses supplied to F-0ff9271d6c39 |
| `0x038`, `0x0C0` | Embedded addresses cached or passed onward |
| `0x14C` | Byte supplying four individual control bits |
| `0x14D`, `0x14E` | Full-byte controls for two entries |
| `0x150`–`0x1F0` | Starting locations supplied to preparation calls |
| `0x210`, `0x21C` | Addresses supplied to later dependencies |
| `0x22C + 0x10 × selector` | Selected-record address; selector is the signed word at `0x24C` |
| `0x254`–`0x25F` | Three words copied from temporary output |
| `0x260`, `0x270` | Two 16-byte records: three data words followed by a gate word |
| `0x2F8 + 0xA8 × entry` | Eight optional record addresses |
| `0x838` and adjacent fields | Shared state used in nonzero mode |
| `0x89C` | Word whose low eight bits select optional records |
| `0x8FD`, `0x902` | Mode byte and secondary selector byte |
| `0x918` | Address supplied to a late dependency |
| `0x9F8` | Working-region address supplied to several dependencies |
| `0xA48`, `0xA49` | Cached bytes read through resolver results |
| `0xA4A`–`0xA4C` | Flag bytes |
| `0xA78`–`0xA98` | Cached addresses, returned words, and selected words |
| `0xAB0`, `0xAB4` | Words explicitly reset |
| `0xB84 + 0x94 × output` | Start of three direct writes for each output |

These are access roles, not complete structure definitions or ownership declarations.

## Behavior

### Initial state changes

The following order is established:

1. Write zero to `0xAB0`, then `0xAB4`.
2. Store the address `base + 0x0C0` at `0xA78`.
3. Call F-0ff9271d6c39 with `base + 0x004`; store its returned word at `0xA7C`.
4. Store `base + 0x038` at `0xA80`.
5. Call D-01 with `base + 0x9F8` and `0x58`.
6. Read the word at `0xA8C` and supply it to F-5d2c52a7254c.
7. Call D-02 with `base + 0x9F8`, the then-current word at `0xA90`, and `0x14`.
8. Read the signed word at `0xA00`; write zero to byte `0xA4C` if negative, otherwise one.
9. Read the signed word at `0x9FC`; write zero to byte `0xA4B` if negative, otherwise one.

D-01 and D-02 are opaque external dependencies. Their argument patterns do not establish clearing, copying, allocation, or release semantics. The sign tests use values present after these calls, not necessarily entry-time values.

### Global-root evaluation and mode selection

After the sign-derived byte writes, the function reads a global root-pointer slot and caches its 32-bit value. This read occurs **before** testing byte `0x8FD`.

The two mode paths do not use that slot identically.

**Zero mode: byte `0x8FD` equals zero**

The initial cached root is not used to obtain the selected pair. Instead, the representation establishes this sequence:

1. Read the global root-pointer slot again.
2. Read the word at offset `0x4C` through that newly obtained root; retain the word.
3. Read the global root-pointer slot again.
4. Read the word at offset `0x48` through this latest root and store it at `0xA94`.
5. Store the retained `0x4C` word at `0xA98`.

Thus, the zero-mode path contains three evaluations of the global root-pointer slot overall: the pre-branch evaluation and two subsequent evaluations. The two selected words need not originate from the same root if the slot changes between evaluations.

This path does not directly write `0xA4A` and does not perform the optional eight-record processing below. It does not guarantee that dependencies leave `0xA4A` unchanged.

**Nonzero mode: byte `0x8FD` is not zero**

The function writes one to `0xA4A`, then one to `0xA4C`. The latter replaces the earlier sign-derived value. There is no corresponding direct replacement of `0xA4B`.

The function then reads byte `0x902`. Both selected words come through the root cached **before** the mode test:

- If `0x902` is zero, read the word at cached-root offset `0x40`, then the word at offset `0x44`.
- Otherwise, read the word at cached-root offset `0x38`, then the word at offset `0x3C`.

Store the second selected word at `0xA98`, then the first selected word at `0xA94`. This path does not reevaluate the global root-pointer slot for these selections.

### Nonzero-mode record processing

Still within nonzero mode, visit entries zero through seven in ascending order. At each entry, reread the word at `0x89C` and test that entry’s corresponding bit. For each set bit, call F-925ad1d16cdd with:

- `base + 0x2F8 + 0xA8 × entry`;
- `base + 0x838`.

The mask is not a loop-wide snapshot. Changes made by an earlier dependency call can alter later selections. Bits above bit seven do not select additional entries.

Next, make three calls to F-e713aa9f4f3b in this order, supplying word pairs from:

1. `0x838` and `0x87C`;
2. `0x83C` and `0x880`;
3. `0x840` and `0x884`.

Each pair is read for its own call; later pairs are not established as having been captured before earlier calls. The relative evaluation order of the two reads within a pair is not specified here.

The three returned words occupy the first three positions of a temporary four-word value. Read `0x844` after these calls for its fourth word. Call F-3f35b3cd17e1 with destination `base + 0x88C` and the temporary value’s address.

After that call:

1. Copy the word at `0x854` to `0x898`.
2. Add the word at `0x868` to the word at `0x88C`, storing at `0x88C`.
3. Add the word at `0x86C` to the word at `0x890`, storing at `0x890`.
4. Add the word at `0x870` to the word at `0x894`, storing at `0x894`.

The stored sums are 32-bit results. No saturation, overflow check, or error branch is shown.

### Three preparation calls

Both modes continue with three calls to F-f5d971abb624:

| Source address | Control | First output address | Second output address |
|---|---|---|---|
| `base + 0x150` | Bit 0 of `0x14C` | `base + 0xA0C` | `base + 0xA50` |
| `base + 0x170` | Bit 1 of `0x14C`, normalized to zero or one | `base + 0xA18` | `base + 0xA58` |
| `base + 0x190` | Bit 2 of `0x14C`, normalized to zero or one | `base + 0xA24` | `base + 0xA60` |

Base is also the first displayed argument of each call. The control byte is reread separately for each call, so earlier dependency effects can change later controls.

A temporary stack word is assigned zero before the first call, zero before the second, and one before the third. Whether these assignments supply an additional ABI-level argument is unresolved. The five displayed arguments are not a verified complete signature.

### Two-entry preparation, resolution, and scaling

Process entries zero and one in sequence. For entry \(j\):

1. Call F-f5d971abb624 with base, source `base + 0x1B0 + 0x20 × j`, the full byte at `0x14D + j`, and output addresses `base + 0xA30 + 0x0C × j` and `base + 0xA68 + 8 × j`. The temporary stack word is assigned zero before this call.
2. Call F-0ff9271d6c39 with `base + 0x01C + 0x0C × j`.
3. Read the signed 32-bit selector at `0x24C`. Call F-108ce6dda517 with the resolver result, selected address `base + 0x22C + 0x10 × selector`, and destination `base + 0x260 + 0x10 × j`.
4. Call F-0ff9271d6c39 again with the same resolver-input address. Read a byte at offset `0x40` through this second result and store it at `0xA48 + j`.
5. Call F-0ff9271d6c39 a third time with that input address. Store this third returned word at `0xA84 + 4 × j`.

The selector is neither a byte nor an unsigned bounded index. Negative values are not rejected. Its read occurs after the first resolver call for that entry.

The three resolver calls are distinct. Their results need not agree, and the second result must permit its byte access independently of the others.

Next, read the destination record’s fourth word at `0x26C + 0x10 × j`.

- If it equals `0x00010000`, skip the scaling-provider call and all three direct scaling writes.
- Otherwise, call F-dc13818ac82a once and use its returned signed 32-bit word as the multiplier for this entry.

When scaling occurs, process the words at `0x260`, `0x264`, and `0x268`, each adjusted by `0x10 × j`, in that order. Read each component after the provider call, multiply its signed 32-bit value by the signed multiplier into a signed 64-bit product, and write back product bits 16 through 47.

Equivalently, the result is the low 32 bits of the product arithmetically shifted right by 16. There is no rounding adjustment or saturation. All signed 32-bit operand products fit in signed 64-bit storage; the final extraction can discard significant upper bits.

The fourth word is only the gate for entering this path. It is not the multiplier, and the scaling path does not directly rewrite it.

### Final derivation

Call F-f5d971abb624 with base, source `base + 0x1F0`, normalized bit 3 of the then-current byte at `0x14C`, a temporary 12-byte output region, and a temporary 8-byte output region. The temporary stack word is assigned zero before this call. No subsequent direct read of these two output regions is shown.

Then call F-9390324a38c0 with a temporary three-word output address, `base + 0x038`, and `base + 0x210`. Copy the three temporary words to `0x254`, `0x258`, and `0x25C`, in order. The values are whatever occupies those temporary words after the call; the callee’s complete initialization behavior is not established here.

Call F-6dc9d8a81ab1 with `base + 0x918`, six, `base + 0x9F8`, and `base + 0x9F8`. The last two arguments are identical addresses.

### Four-output population

Visit outputs zero through three in ascending order, using stride `0x94`. For output \(k\):

1. Read and copy `0x254`, `0x258`, and `0x25C`, in order, to `0xB84 + 0x94 × k`, `0xB88 + 0x94 × k`, and `0xB8C + 0x94 × k`.
2. Call F-3f35b3cd17e1 with destination `base + 0xB90 + 0x94 × k` and input `base + 0x21C`.
3. Process mapping entries zero and one, in that order.

For mapping entry \(j\), reread the pointer-bearing word at `0xA90`, then read a signed word at pointer-relative offset `0x0C + 4 × j`.

A negative mapping value skips that entry. For a nonnegative value \(m\):

- Copy the word at `0x260 + 0x10 × j` to `0xBC0 + 0x94 × k + 4 × m`.
- Then copy the word at `0x264 + 0x10 × j` to the immediately following word.

Mapping values denote **word positions**, not eight-byte slots. There is no upper-bound check. Address multiplication and addition are represented in 32-bit address arithmetic, without a shown overflow guard.

The copies use values present at each read. Dependencies or overlapping mapping destinations can therefore affect later source words, pointers, mappings, or outputs. Replication of identical values across all four outputs is not unconditional.

## Side effects

Direct effects are the ordered writes described above. Dependencies can additionally affect reachable memory or globals; their preservation behavior is unknown.

No allocation, deallocation, ownership transfer, or release is directly established. Passing the word at `0xA8C` to F-5d2c52a7254c does not establish what resource operation, if any, occurs. Caching resolver results does not establish ownership.

Temporary storage is available during the relevant calls. Whether dependencies retain temporary addresses is unknown. Any later access would require a separate valid-lifetime guarantee.

The function is not shown to be atomic or transactional. There is no explicit status return, rollback, exception handling, or recovery path. If an access faults or a dependency does not return, preceding writes are not undone by any shown mechanism.

No concurrency or reentrancy guarantee is established. Distinct root-slot evaluations and repeated object reads remain distinct in this contract; they are not replaced by entry-time snapshots.

## Dependencies

| Opaque ID | Observed interaction |
|---|---|
| F-0ff9271d6c39 | Receives embedded addresses; returns words, some subsequently treated as addresses |
| F-5d2c52a7254c | Receives the word read at `0xA8C` |
| F-925ad1d16cdd | Receives an optional record address and shared-state address |
| F-e713aa9f4f3b | Returns a word from two supplied words |
| F-3f35b3cd17e1 | Receives destination and input addresses |
| F-f5d971abb624 | Receives base, source, control, and two output addresses; complete ABI unresolved |
| F-108ce6dda517 | Receives a resolver result, selected-record address, and destination |
| F-dc13818ac82a | Returns the scaling multiplier |
| F-9390324a38c0 | Receives temporary three-word output storage and two object addresses |
| F-6dc9d8a81ab1 | Receives an object address, six, and two aliased working-region addresses |
| D-01 | Receives the working-region address and `0x58` |
| D-02 | Receives the working-region address, a current word, and `0x14` |

D-01 and D-02 are document-local opaque dependency IDs. Full dependency contracts remain documented_only.

## Acceptance examples

These are conditional static expectations, not observed executions. Referenced accesses must be valid.

1. **Distinct zero-mode roots.** Suppose the initial root-slot evaluation yields root A, the next yields root B, and the final yields root C. In zero mode, `0xA94` receives the word read from C at `0x48`, while `0xA98` receives the earlier word read from B at `0x4C`. Root A supplies neither selected word.

2. **Cached nonzero-mode root.** If the initial root is A and byte `0x902` is zero, nonzero mode reads A’s `0x40` word and then A’s `0x44` word. It stores the latter at `0xA98` before storing the former at `0xA94`; it does not fetch a new root for either read.

3. **Flags.** If the sign-test inputs at `0xA00` and `0x9FC` are respectively negative and nonnegative, the initial flag writes are zero at `0xA4C` and one at `0xA4B`. Nonzero mode subsequently writes one at `0xA4C`.

4. **Optional selection.** If the mask remains `0x00000081`, nonzero mode calls F-925ad1d16cdd for entries zero and seven, in that order.

5. **Signed selector.** If the selector read for an entry is −1, the selected address supplied to F-108ce6dda517 is `base + 0x21C`, provided address formation and access are valid. No rejection is shown.

6. **Scaling.** A gate of `0x00010000` skips scaling. Otherwise, a multiplier of `0x00008000` and component `0x00030000` produce `0x00018000`. A component of −1 with that multiplier produces −1, not zero.

7. **Mapping overlap.** If mappings and sources remain stable and destinations do not alias other relevant state, mappings zero and zero cause entry one to replace entry zero’s pair. Mappings zero and one cause entry one’s first word to replace entry zero’s second word.

8. **Negative mappings.** If every mapping read is negative, no mapping-derived writes occur. The three leading copies and F-3f35b3cd17e1 call still occur for each output.

## Uncertainties

The following remain documented_only:

- Dependency semantics, preservation rules, failure behavior, ownership effects, and complete calling conventions.
- Whether the stack-word assignments surrounding F-f5d971abb624 supply an additional argument.
- The global root-pointer slot’s purpose, mutability rules, and synchronization requirements.
- Legal selector ranges, mapping bounds, full object extents, and permitted aliasing.
- Platform-specific consequences of invalid addresses, overflowing address calculations, or alignment violations.
- Byte order, concurrency guarantees, and temporary-address retention.
- Application-level meanings of transformed words and flags.

No validation shown in this function resolves these unknowns.

## Evidence

Established static evidence comprises the supplied representation’s accesses, widths, branch conditions, call ordering, loop bounds, arithmetic, and separate global root-slot evaluations. It supports a pre-mode cached root for nonzero mode and two additional root-slot evaluations for zero-mode word selection.

Inferred intent is limited to preparation of derived working state and output records. Dependency semantics are not inferred from argument shape.

No implementation access, runtime execution, or tests were used. Acceptance examples express conditional consequences of static evidence only.
