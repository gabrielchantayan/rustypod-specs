<!-- Trusted provenance: F-aee324a7faa5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aee324a7faa5

## Purpose

F-aee324a7faa5 describes a machine-state-dependent sequence of narrow memory writes, state-field updates, and calls to externally defined functions. Five call results determine subsequent write addresses and values. The visible sequence ends at an invalid-instruction boundary rather than an established normal return.

No application-level purpose is established. In particular, the evidence does not establish that the writes initialize hardware, reset a device, decode instructions, or manipulate an allocated object. Those interpretations remain unsupported.

This contract specifies the visible operations and their ordering, conditional on preceding calls returning and memory accesses succeeding. It does not establish that the recovered sequence is executable as shown: the supplied material explicitly warns about invalid instruction data and overlapping global symbols.

## Interface

### Explicit inputs and output

The recovered interface presents two 32-bit arguments:

- **First argument:** no visible use.
- **Second argument:** passed unchanged as the second explicit argument of the first external call.

There is no established return value. The recovered declaration indicates no result, but the visible body does not reach a normal return.

### Additional entry-state inputs

The function depends on machine state beyond its explicit arguments. This contract assigns descriptive labels to those inputs; they are not additional declared parameters:

| Entry-state input | Width and interpretation | Visible use |
|---|---|---|
| **Entry address** | 32-bit address bit pattern | Supplies the first write address, initial stored value, another derived write address, and arguments to the first call |
| **State base** | 32-bit address bit pattern | Identifies a field at byte offset `0x14` |
| **Entry state word** | 32-bit bit pattern | Written to the state field, then shifted left by eight bits for later writes |
| **Entry continuation** | 32-bit address-like bit pattern | Determines the first call target and contributes to its third argument |

These values are not initialized within the visible body. A caller using only the two explicit arguments is therefore insufficient to establish valid execution.

### Referenced storage

The visible accesses involve:

- Halfword destinations: **16-bit writes** at computed addresses.
- A state field: a **32-bit write** at state base plus `0x14`.
- One global location, called **G-01** here, repeatedly assigned zero.

The width, extent, ownership, and hardware meaning of G-01 are not established. A warning reports overlapping global symbols, so this contract does not assign it a larger object layout or a specific reset semantic.

### Arithmetic conventions

Address calculations are described in a 32-bit machine-address domain. Left shifts discard bits above bit 31; additions and doubling retain the low 32 bits. This is a bit-vector description of the visible operations, not a claim that equivalent signed expressions would be portable in a high-level language.

For each 64-bit call result:

- **Lower word** means bits 0–31.
- **Upper word** means bits 32–63.
- Right shifts used to derive addresses are logical.
- A 16-bit store retains only bits 0–15 of its value.

The evidence establishes access widths, not byte order.

## Behavior

### 1. Initial writes and first call

Before any visible external call, the function performs the following ordered effects:

1. Stores the low 16 bits of the entry address at the entry address.
2. Repeats that same 16-bit store at the same address.
3. Stores the same 16-bit value at twice the entry address, with the address calculation reduced to 32 bits.
4. Stores the entry state word into the 32-bit field at state base plus `0x14`.

The repeated stores are distinct visible operations. They must not be collapsed in an interpretation where individual writes are observable.

The function then invokes **F-000000000001**, whose target is derived from the entry continuation plus `0xFFE`. Its three explicit argument bit patterns are:

1. Entry address shifted left by five bits, retaining 32 bits.
2. The second explicit input argument.
3. Entry address plus entry continuation, retaining 32 bits.

The visible body does not consume a return value from this call.

### 2. Five result-driven write stages

Five subsequent calls each supply a 64-bit result. Every result is used to derive a destination address and a 16-bit stored value. Each stage performs two consecutive identical halfword writes, followed by a zero assignment to G-01.

There are two address/value transformations.

**Masked transformation**

- Destination address: bits 3–26 of the lower word, interpreted as an unsigned address.
- Stored value: the low 16 bits of that destination address.

Equivalently, the address is the lower word masked with `0x07FFFFFF`, then logically shifted right by three. Its range is `0x00000000` through `0x00FFFFFF`.

The recovered value expression shifts the lower word left by five and then logically right by eight before narrowing to 16 bits. Under 32-bit arithmetic, this produces the same stored value specified above. Bits 27–31 of the result’s lower word do not affect either output.

**Byte-shift transformation**

- Destination address: the lower word logically shifted right by eight.
- Stored value: bits 8–23 of the result, interpreted as a 16-bit value.

The address range is again `0x00000000` through `0x00FFFFFF`. Although the visible stored-value expression uses a 64-bit shift, narrowing it to 16 bits excludes the upper word entirely.

The five stages are ordered as follows:

| Stage | Result provider | Transformation | State-field write after G-01 is zeroed |
|---|---|---|---|
| 1 | F-000000000002 | Masked | Entry state word |
| 2 | F-000000000004 | Byte-shift | Entry state word shifted left by eight, retaining 32 bits |
| 3 | F-000000000006 | Masked | Same shifted entry state word |
| 4 | F-000000000008 | Masked | Same shifted entry state word |
| 5 | F-000000000010 | Byte-shift | No explicit state-field write in this stage |

Each result-provider call has no explicit arguments in the supplied representation. That does not establish that the underlying callee has no machine-state inputs.

For each stage, the order is:

- Result-provider call returns.
- First halfword write occurs.
- Identical second halfword write occurs.
- G-01 is assigned zero.
- The state-field write occurs, if listed above.
- The next transition call occurs.

### 3. Calls between result stages

After stages 1–4, the function makes a transition call before requesting the next result:

| After stage | Transition dependency | Three explicit arguments |
|---|---|---|
| 1 | F-000000000003 | Zero; stage 1 upper word; `0x0891021D` |
| 2 | F-000000000005 | Zero; stage 2 upper word; `0x08910285` |
| 3 | F-000000000007 | Zero; stage 3 upper word; `0x089102ED` |
| 4 | F-000000000009 | Zero; stage 4 upper word; `0x08910355` |

The numeric third arguments are observable 32-bit tokens. Their meaning is unknown. No return value from these transition calls is visibly used.

### 4. Final call sequence

After stage 5 has performed its duplicate halfword writes and zero assignment to G-01, the function calls **F-000000000011** with four explicit arguments:

1. `0xFFFFFF1C`, a 32-bit bit pattern representing −228 if interpreted as signed.
2. Stage 5 upper word minus `0x264`, retaining 32 bits.
3. `0x4B`.
4. Zero.

It then invokes, in order:

- **F-000000000012**
- **F-000000000013**
- **F-000000000014**
- **F-000000000015**

No explicit arguments or consumed return values are shown for those four calls.

The visible sequence terminates at **F-000000000016**, which represents the reported invalid-instruction boundary. It is not established to be an ordinary callable function or an intentional application-level halt.

### 5. Failure and interruption behavior

No visible validation precedes any computed write or call. There is no explicit error result, rollback, retry, or recovery path.

If a call fails to return, a write faults, or control transfers elsewhere, later operations are not guaranteed to occur. Earlier writes have already happened; the visible body provides no mechanism to undo them.

## Side effects

If all preceding operations complete and all ordinary calls return, the visible body contains:

- **13 explicit 16-bit writes:** three initial writes and two writes in each of five result stages.
- **Five explicit 32-bit state-field writes:** two of the original entry state word and three of its eight-bit-left-shifted form.
- **Five explicit zero assignments to G-01**, with unresolved width.
- **15 external call sites** before the terminal invalid-instruction marker.

These counts exclude all effects inside dependencies.

There are no explicit memory-value loads in the visible body other than obtaining call results and entry state. A destination is not read to determine the value written there. In particular, the first stores write bits from an address itself, not the halfword previously held at that address.

Aliasing is not excluded. A computed halfword destination may overlap another destination, the state field, G-01, or storage used by a dependency. Consequently:

- Later writes may overwrite earlier values.
- A final-state-only observation may hide repeated writes.
- Partial overlaps depend on byte order and the unresolved width of G-01.
- External calls may change any storage they can access.

The body contains no explicit allocation or release operation. That does not establish ownership, lifetime, or allocation behavior inside dependencies. No dependency is assumed to preserve globals or retain, transfer, or release memory.

Successful execution requires writable destinations of sufficient width and callable targets with compatible machine-level entry conditions. Alignment requirements, mapped-memory limits, and executable-address conventions are not established.

## Dependencies

The following are **specification-local opaque function IDs**. Except for the subject function’s supplied ID, they are labels for visible dependency sites, not verified published identities. Distinct labels do not prove distinct implementations or exclude target aliasing.

| Dependency IDs | Established role |
|---|---|
| F-000000000001 | Entry-state-derived call with three explicit arguments |
| F-000000000002, F-000000000004, F-000000000006, F-000000000008, F-000000000010 | Five calls whose results are consumed as 64-bit values |
| F-000000000003, F-000000000005, F-000000000007, F-000000000009 | Inter-stage calls receiving zero, the preceding result’s upper word, and a fixed token |
| F-000000000011 | Four-argument call after the final write stage |
| F-000000000012 | Subsequent call with no explicit arguments shown |
| F-000000000013–F-000000000015 | Three further calls in fixed order |
| F-000000000016 | Terminal invalid-instruction marker; runtime identity unresolved |

Call-target construction differs across these sites. The first depends on entry state; the others are constructed from fixed address expressions or appear as direct calls. Their implementation behavior, calling conventions, failure behavior, and global effects are unknown.

## Acceptance examples

These are conditional specification examples, not executed tests. They assume the visible operations accurately represent executable behavior, relevant memory is writable, and preceding dependencies return.

### Address-derived initial stores

Given entry address `0x00010020`, the initial halfword value is `0x0020`.

The first two writes target `0x00010020`. The third targets `0x00020040`. All three carry `0x0020`; no read of either destination is required.

### Masked result transformation

Given a masked-stage lower word of `0xF9234567`:

- Masking and shifting yields destination `0x002468AC`.
- Both halfword writes carry `0x68AC`.

A lower word of `0x01234567` produces the same address and value because the differing high five bits are discarded.

### Byte-shift result transformation

Given a byte-shift-stage lower word of `0xABCDEF12`:

- Destination is `0x00ABCDEF`.
- Both halfword writes carry `0xCDEF`.

Changing only the result’s upper word does not change those writes, but changes the upper-word argument used by the subsequent call.

### State-word truncation

Given entry state word `0x12345678`, the first two explicit state-field writes carry `0x12345678`. The next three carry `0x34567800`.

This specifies explicit writes, not uninterrupted field contents: intervening dependencies may modify the field.

### Final subtraction wraps

Given stage 5 upper word `0x00000100`, the second argument to F-000000000011 is `0xFFFFFE9C`.

No underflow error is visibly reported.

### Zero and inaccessible destinations

A lower word of zero produces address zero and halfword value zero under either transformation. There is no visible null-address guard.

If the first write faults, this contract does not require the duplicate write, G-01 assignment, or any subsequent operation to occur.

## Uncertainties

The following are **documented_only**:

- **Instruction validity:** the supplied material reports bad instruction data and truncated control flow. Reachability and correct instruction boundaries are unverified.
- **Global layout:** G-01’s width and overlaps are unresolved.
- **Machine conventions:** byte order, alignment behavior, executable-address interpretation, and exception delivery are unknown.
- **Dependency semantics:** implementations, hidden inputs, register preservation, memory effects, and ownership behavior are unavailable.
- **Entry-state validity:** no caller evidence establishes how the extra machine-state inputs are populated.
- **Termination:** the terminal marker does not establish intentional nonreturning behavior in the original program.
- **Concurrency and device semantics:** no atomicity, synchronization, volatility, or memory-order guarantee is established.

The arithmetic and event ordering above are established only within the supplied recovered representation. An interpretation as a hardware-control or initialization routine would be inferred intent, not established behavior.

## Evidence

The sole evidence is the supplied static recovered function representation and its warnings. It exposes two 32-bit formal inputs, additional entry-state dependencies, explicit narrow stores, five 64-bit result uses, fixed call ordering, and a terminal invalid-instruction marker.

No dependency implementations, caller context, memory map, original instruction bytes, or runtime observations were supplied. No tools or runtime tests were used. The acceptance examples are arithmetic consequences and conditional trace requirements, not execution results.
