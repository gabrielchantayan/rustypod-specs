<!-- Trusted provenance: F-fbfa6017b0f5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fbfa6017b0f5

## Purpose

F-fbfa6017b0f5 returns the low byte of a cached 32-bit word, advances a 32-bit control value, and replaces the consumed cache value with its logical right shift by eight bits. When the initially read control value has both low bits clear, the function first performs a word refill.

**Established static behavior:** the function captures a context reference for its final cache consumption and state updates, but obtains the initial control value through a separate evaluation of the global context reference. Refill accesses also evaluate that global reference again. These accesses are not unconditionally accesses to one context.

**Inferred intent:** with a stable context reference and suitably initialized storage, the function implements sequential byte consumption from words obtained through two selectable backing-region addresses.

The evidence does not establish a stream format, storage capacity, initialization protocol, ownership arrangement, or supported concurrent-use model.

## Interface

### Invocation and result

- Opaque function ID: **F-fbfa6017b0f5**.
- Explicit arguments: none.
- Return width: 32 bits, unsigned.
- Return range on normal completion: 0 through 255.
- Implicit inputs and outputs: a global context reference, context fields, and conditionally addressed backing memory.

There is no explicit error result. Returning zero means that the consumed word’s low byte was zero; it is not an established failure indicator.

### Context layout

The following offsets are byte offsets from whichever context base is used for the particular access:

| Offset | Width | Established role |
|---|---:|---|
| `0x64` | 32 bits | Backing-region address used when initial control bit 10 is clear |
| `0x68` | 32 bits | Backing-region address used when initial control bit 10 is set |
| `0x84` | 32 bits | Control field |
| `0x88` | 32 bits | Cached word |

The supplied representation uses 32-bit address-bearing values and 32-bit word accesses. The backing-region fields are used as addresses; their representation in signed temporaries does not establish signed payload semantics or ownership.

The function does not verify that repeated global-reference evaluations identify the same context.

### Preconditions for normal completion

Every address actually accessed must support the corresponding read or write:

- The entry-captured context must permit a 32-bit cache read and 32-bit writes to its control and cache fields.
- The context selected by the separate initial control-reference evaluation must permit a 32-bit control read.
- On refill, each context selected by a subsequent global-reference evaluation must permit the particular field access performed through it.
- The selected source address must permit a four-byte read.
- The refill destination must permit a four-byte cache write.

These requirements do not imply that all accesses concern one context. A stable reference is an additional condition for ordinary single-context operation, not a check performed by the function.

No visible bounds, alignment, validity, initialization, or synchronization checks are present. Starting with nonzero low control bits consumes the existing cache without first initializing it.

## Behavior

### Entry capture and independent initial control read

The function first evaluates the global context reference and retains that context base for later cache consumption and final writes.

It then evaluates the global context reference again and reads the 32-bit control field at offset `0x84` through that fresh reference. The initial control value is therefore not necessarily read from the entry-captured context.

This distinction applies on **both** refill and non-refill paths. Even an invocation that performs no refill can obtain its initial control from one context while reading the cache and writing the final state in another.

The retained initial control value determines:

- Whether a refill occurs.
- Which backing-region address field is selected if a refill occurs.
- The value written in the final control update.

The final control update does not reread the destination’s current control field.

### Refill condition and region selection

A refill occurs exactly when the initial control value’s low two bits are zero.

On refill, bit 10 of that same initial value selects the backing-region address:

- Bit 10 clear: use offset `0x64`.
- Bit 10 set: use offset `0x68`.

The selection is established by a 32-bit left shift by 21 followed by a signed sign test. Original bit 10 becomes the sign bit. Higher bits do not affect the selection after that 32-bit shift.

The refill byte offset comes from a **later** control-field read through another evaluation of the global reference. The function masks that later value with `0x3FF`, producing an offset from 0 through 1023.

Consequently, eligibility and region selection depend on the retained initial control value, while the source offset depends on a separate observation. The later control read need not use the same context or return the same value.

### Refill access order

For the branch with initial bit 10 clear, the visible sequence is:

1. Read the backing-region address at offset `0x64` through a fresh global-reference evaluation.
2. Read the control field through another global-reference evaluation and retain its low ten bits as the byte offset.
3. Read the 32-bit word at the selected address plus that byte offset.
4. Store the source word into a cache field whose destination base is obtained through another global-reference evaluation.

For the branch with initial bit 10 set, the visible sequence differs:

1. Read the control field through a fresh global-reference evaluation and retain its low ten bits.
2. Read the backing-region address at offset `0x68` through another global-reference evaluation.
3. Read the 32-bit source word at that address plus the retained byte offset.
4. Store that word into a cache field whose destination base is obtained through another global-reference evaluation.

Only the selected backing-region address field is read.

The refill store requires the source word to have been read. The supplied representation does not independently establish the relative evaluation order of the refill destination’s global reference and the source-side evaluations within the assignment. That uncertainty does not change the established fact that the destination is not taken from the entry-captured base.

The offset is measured in **bytes**, not in 32-bit elements. No visible operation constrains the resulting address to a declared allocation.

If the initial and later control observations agree, refill offsets are multiples of four from 0 through 1020. This does not guarantee address alignment unless the selected base is appropriately aligned.

If those observations differ, an offset such as 1023 is possible. It requests four bytes beginning at offset 1023; there is no check against crossing a presumed 1024-byte boundary.

### Non-refill path

If either low bit of the initial control value is set, the function skips:

- Both backing-region address fields.
- The later control read used for a refill offset.
- The source-word read.
- The refill cache store.

It proceeds to the entry-captured context’s cache. The initial control read still uses the separate global-reference evaluation described above.

No check establishes that the existing cache contains valid or current data.

### Cache consumption, final writes, and result

After the optional refill, the function reads the cached 32-bit word at offset `0x88` from the **entry-captured context**.

Using that consumed word and the retained initial control value, it then:

1. Writes the initial control value plus one to offset `0x84` of the entry-captured context.
2. Writes the consumed word shifted logically right by eight bits to offset `0x88` of that same context.
3. Returns the consumed word’s low eight bits as an unsigned 32-bit result.

The control increment is modulo \(2^{32}\). The logical shift inserts eight zero bits at the top.

The result and final shifted-cache value both derive from the cache read performed after the optional refill, not directly from the source-word read.

### Distinct refill and consumption destinations

Unconditionally, a refill invocation performs:

- A complete source-word store to the cache selected by the refill destination’s global-reference evaluation.
- A later cache read from the entry-captured context.
- Final control and shifted-cache writes to the entry-captured context.

Without global-reference stability, the consumed word need not be the refilled word. The refill store and final cache store need not target the same context.

Under a stable global reference, ordinary nonoverlapping storage, and no intervening mutation, the refill destination and entry-captured context coincide. Only under those conditions does the usual description apply: the same cache receives the full source word and then that word shifted right by eight bits.

### Repeated calls under stable conditions

With a stable context reference, ordinary nonoverlapping memory, and no external mutation, a loaded numeric word `0xA1B2C3D4` supplies:

- `0xD4` on the refill call.
- `0xC3` on the next call.
- `0xB2` on the next call.
- `0xA1` on the next call.

After those four consumptions, the cache is zero. The next call refills because the control value is again divisible by four.

This is least-significant-byte-first consumption of the **numeric loaded word**. Its relationship to increasing source-memory addresses depends on target byte order, which is not established here.

## Side effects

### Reads

The function visibly performs:

- An entry evaluation of the global context reference.
- A separate global-reference evaluation for the initial control read.
- Conditional global-reference evaluations for refill field accesses and the refill destination.
- A conditional 32-bit source read.
- A cache read through the entry-captured context.

The distinction between the first two reference evaluations is independent of the additional distinctions introduced by refill.

### Writes

On every normally completing invocation, the entry-captured context receives:

- The retained initial control value incremented modulo \(2^{32}\).
- The consumed cache word shifted logically right by eight bits.

On refill, an independently selected cache destination also receives the full source word before cache consumption.

These are stores, not atomic read-modify-write operations. No locking or atomicity guarantee is visible.

There is no explicit store through the source address. However, no check excludes overlap among backing memory and context storage. If addresses alias, a context write can also change memory reachable through a backing-region address. The simple repeated-call examples assume such overlap is absent.

### Ownership and lifetime

No allocation, deallocation, ownership transfer, reference-count operation, or resource-release call is visible.

Storage must remain accessible for the accesses actually performed. The evidence does not identify which component owns it, initializes it, preserves the global reference, or eventually releases it.

### Failure and partial effects

There is no visible error handling, rollback, retry, or exception translation.

An inaccessible or otherwise unsupported access can prevent normal completion. Platform-specific fault behavior is **documented_only**.

Earlier writes are not undone. For example:

- A refill store may have completed before a later cache access fails.
- The final control store precedes the final shifted-cache store, so failure at the latter can leave the control already changed.
- If refill and final destinations differ, partial effects may be distributed across contexts.

## Dependencies

No calls to other functions are visible. There are no additional function dependencies to identify by opaque function ID.

Operational dependencies are:

- The global context reference.
- The documented context fields.
- Conditionally selected backing memory.
- Platform semantics for 32-bit accesses, address arithmetic, byte order, and faults.

No outside component’s preservation of state or ownership of memory is assumed.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Unless stated otherwise, assume a stable global reference, ordinary accessible nonoverlapping memory, no external mutation, and successful accesses.

### First-region refill and consumption

Given initial control `0` and a numeric source word `0xA1B2C3D4` at the first backing-region base:

- The call selects offset `0x64` and source byte offset zero.
- The cache first receives `0xA1B2C3D4`.
- The result is `0xD4`.
- Final control is `1`.
- Final cache is `0x00A1B2C3`.

Three subsequent calls return `0xC3`, `0xB2`, and `0xA1`. They do not read a backing-region address or source word. Final control is `4`, and final cache is zero.

### Second-region selection

Given initial control `0x400` and numeric source word `0x11223344` at the second backing-region base, the call selects offset `0x68` and byte offset zero. It returns `0x44`, leaves control `0x401`, and leaves cache `0x00112233`.

### Existing-cache consumption

Given control `5` and cache `0x89ABCDEF`, the call returns `0xEF`, writes control `6`, and leaves cache `0x0089ABCD`.

Invalid backing-region addresses alone do not affect this path because neither address field nor source memory is accessed.

### Region boundary and wraparound

Starting at control `0x3FC`, a refill uses the first region at byte offset 1020. After four uninterrupted calls, control is `0x400`; the following call refills from the second region at offset zero.

Separately, control `0xFFFFFFFF` with cache `0x12345678` causes no refill. The call returns `0x78`, writes control zero, and leaves cache `0x00123456`. The next uninterrupted call refills from the first region at offset zero.

### Initial control from a different context

Conditionally suppose the entry capture identifies context A, while the separate initial control evaluation identifies context B:

- Context B supplies initial control `5`.
- Context A supplies cache `0x89ABCDEF`.

The function performs no refill, returns `0xEF`, writes control `6` into A, and writes cache `0x0089ABCD` into A. It does not increment B’s control merely because B supplied the initial value.

This describes the represented data dependencies. Whether an execution can produce such reference changes with defined platform behavior is **documented_only**.

### Refill into a different context

Conditionally suppose:

- The entry capture identifies context A.
- The initial control read returns zero.
- The refill source read returns `0x11223344`.
- The refill destination’s reference evaluation identifies a distinct context B.
- A’s cache remains `0x89ABCDEF` until consumption.

The refill writes `0x11223344` into B’s cache. The function then consumes A’s cache, returns `0xEF`, writes control `1` into A, and writes cache `0x0089ABCD` into A.

The returned byte is not required to be `0x44`, because the consumed cache is not the refill destination.

### Different control observations

If the initial control read returns zero but the later offset read returns `7`, the function selects the first backing-region address and attempts a four-byte source read at byte offset `7`. Its final control write is still `1` in the entry-captured context.

Successful completion additionally depends on the resulting source access and all context accesses being supported.

## Uncertainties

- **Documented_only — global-reference changes:** supported causes, synchronization, atomicity, and platform-level validity of differing evaluations are not established.
- **Documented_only — expression evaluation:** the refill destination reference’s ordering relative to source-side evaluations within the assignment is not independently established.
- **Documented_only — memory rules:** alignment requirements, address-overflow behavior, memory-mapped I/O semantics, and fault handling are unknown.
- **Documented_only — byte order:** numeric low-byte extraction is established; correspondence to source byte addresses is not.
- **Documented_only — storage extent and lifetime:** actual backing-region sizes, initialization, ownership, and release responsibilities are unknown.
- **Documented_only — higher control bits:** bits above bit 10 remain part of the incremented control value but do not directly select the refill region or offset. Their broader meaning is unknown.
- **Inferred intent:** sequential byte reading across word refills and two selectable regions is plausible, not an established higher-level protocol.

## Evidence

This contract is based solely on the supplied static representation of F-fbfa6017b0f5.

Established evidence includes the independent entry capture and initial control-reference evaluation; conditional refill; bit-10 selection; later ten-bit byte offset; branch-dependent address-field read order; independently evaluated refill destination; entry-captured cache consumption and final stores; 32-bit wrapping increment; logical shift; and eight-bit return mask.

The contract does not equate distinct reference evaluations, control observations, refill destinations, or consumed words without stated stability conditions.

No runtime tests have been performed. No execution, target-specific memory behavior, or external initialization protocol has been verified.
