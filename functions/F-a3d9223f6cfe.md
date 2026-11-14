<!-- Trusted provenance: F-a3d9223f6cfe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a3d9223f6cfe

## Purpose

F-a3d9223f6cfe exposes an ordered sequence of indirect calls, byte-selector accesses, and four-byte stores. It uses its first explicit input to derive addresses, call arguments, and byte payloads. Its third explicit input supplies two early four-byte stores. Later stores clear locations or populate them with scaled values obtained from dependencies.

An application-level purpose is not established. Initialization or reconfiguration is **inferred intent** only; the visible operations do not establish an object model, resource type, or successful initialized state.

The supplied static representation reports invalid instruction data and ends at a truncated control-flow boundary. This contract therefore describes the visible trace **conditional on that representation accurately describing executable behavior**. It does not establish that the complete trace is reachable or that execution returns normally.

## Interface

### Explicit inputs

| Input | Width | Visible role |
|---|---:|---|
| Selector, designated **S** | Unsigned 32 bits | Address calculations, dependency arguments, and a truncated byte payload. |
| Second input | 32 bits, uninterpreted | No explicit use in the supplied body. |
| Payload, designated **P** | 32 bits, uninterpreted | Stored unchanged at two early destinations. |

The absence of an explicit use of the second input does not establish machine-level irrelevance. The indirect interfaces and recovered calling convention are incomplete.

### Implicit inputs and preconditions

The trace also requires:

- An incoming register-derived address base, designated **B**.
- An incoming link-related value used to calculate the first indirect-call target.
- Existing memory at computed byte and four-byte read locations.
- Dependency results, including one represented 64-bit result and an additional register-derived result.
- An execution environment supporting the encountered addresses, access widths, and instruction stream.

No visible validation establishes address validity, permissions, alignment, dependency availability, or result ranges. For the entire trace to occur, every preceding memory access must complete and every preceding dependency must return. These are conditions for execution, not checks performed by the function.

### Outputs and termination

The displayed interface declares no return value. No normal return is visible. The trace ends at an invalid-instruction termination marker.

That marker establishes a static-analysis boundary. It does not establish a particular runtime exception, exit code, hardware trap, or error-reporting convention.

### Widths and notation

The following names describe contract values, not application fields:

- **Q** is S shifted logically right by three bits.
- **M** is the low 22 bits of Q.
- A **word** means exactly four bytes.
- A **selected byte destination** is the address obtained by reading an unsigned byte and adding 17.

Byte selectors are zero-extended before address arithmetic. Their selected destinations therefore range from address 17 through address 272, inclusive. A saved selector used with an offset of 68 produces a word-start address from 68 through 323.

All represented ordinary scalar results are 32 bits unless stated otherwise. Byte stores retain only the low eight bits of their payload. The represented 64-bit result is separated into low and high 32-bit halves.

Expressions such as 4S or 16 times a dependency result describe the represented scaling. Unsigned 32-bit calculations discard bits above bit 31. In particular, the unsigned products 1024Q and Q multiplied by 2²⁶ are 32-bit products, not unbounded addresses. Calculations represented through signed intermediates have unresolved overflow and conversion semantics; a uniform modulo-2³² interpretation requires confirmation from the actual instructions.

Endianness and alignment requirements are not established.

## Behavior

### Dependency and ordering conventions

Opaque dependency IDs identify call sites, not proven-distinct implementations.

An **ordinary chain** identified by a range such as F-d01a through F-d01e contains five ordered calls. The first receives the arguments stated below; the remaining four have no explicit arguments in the supplied representation. Where a chain result is named, it is the result of its fifth call.

“No explicit arguments” does not mean that a call consumes no register, stack, or global inputs. No dependency is assumed to preserve globals, memory, hidden registers, or ownership state.

All direct accesses described below retain their order relative to the intervening calls. Calls may affect later reads. Equal destination addresses do not eliminate the observability of separate writes.

### Entry payload stores and selector-derived byte update

The first visible memory effect stores P as a word at B + 36.

The ordinary chain F-d01a through F-d01e follows, with initial arguments 4S and S. After that chain, a second word store places P at 4S + 36. The function then reads a word at 4S + 116. That read supplies the first argument of the ordinary chain F-d02a through F-d02e; designate its final result **R**.

After obtaining R, the function reads a byte at 4S + 1 and stores the low eight bits of S shifted right by 22 at the selected byte destination. Thus only selector bits 22 through 29 reach memory; bits 30 and 31 do not survive truncation.

F-d03a receives 4R. F-d03b follows without explicit arguments, and its 32-bit result becomes the first argument of the ordinary chain F-d04a through F-d04e. Designate that chain’s result **T**.

The byte at 4S + 1 is then read again, and zero is stored at the newly selected destination. This is not necessarily the destination of the preceding byte store: intervening calls may change the selector source, and aliasing may also affect it.

The ordinary chain F-d05a through F-d05e follows, initially receiving 4T.

### Q-derived clears and retained selector

The next direct effects clear one byte at Q + 15, then read a byte at 1024Q + 1 and clear its selected byte destination.

F-d06a receives four arguments: three copies of R shifted left by 23 bits, followed by R shifted left by 25 bits. F-d06b follows without explicit arguments. Its result supplies the first argument to the ordinary chain F-d07a through F-d07e; designate that chain’s result **U**.

The function now reads the byte at 1024Q + 1 again. Designate this unsigned byte **C**. It clears the byte at C + 17, then executes the ordinary chain F-d08a through F-d08e with initial argument 4U.

After those calls, it clears a word beginning at C + 68 and then clears a word at absolute address 36. The word destination uses the saved C; it is not based on a selector re-read after the calls.

### Word clearing and the paired result

F-d09a receives four arguments in this order: zero, 152, 1024Q, and 152. F-d09b follows without explicit arguments. Its result is passed to F-d10a.

F-d10b then executes without explicit arguments. Its result supplies the first argument of the ordinary chain F-d11a through F-d11e. Designate the chain’s result **V**.

The function clears a word at 4V + 68, then separately clears a word at absolute address 36.

The ordinary chain F-d12a through F-d12e initially receives 4V. Its final result is represented as 64 bits. Designate its low 32 bits **L** and its high 32 bits **H**.

F-d13a receives 4L. After that call, the function clears a word at 2H + 68 and then clears a word at absolute address 36.

F-d14a receives three arguments: zero, an additional register-derived result exposed after F-d13a, and four times that additional result. F-d14b follows without explicit arguments.

The additional register-derived result is not established to equal H. The word clear uses the high half of the earlier paired result; F-d14a uses the separately represented result associated with F-d13a.

### Masked-selector values and scaled word stores

The ordinary chain F-d15a through F-d15e initially receives 128M. Designate its result **W**. The function clears a word at 4W + 68 and then clears a word at absolute address 36.

The ordinary chain F-d16a through F-d16e initially receives 4W. Its final result, designated **X**, is scaled by four and passed to F-d17a. Designate the result of F-d17a **Y**.

A word at 4Y + 20 is cleared. F-d18a then receives Y, Y, Y, and 128M, in that order. F-d18b through F-d18e follow without explicit arguments. Designate the final result **Z**. A word at 4Z + 20 is cleared.

The following three chains produce distinct scaled-value stores:

| Chain | First argument | Final result | Subsequent word store |
|---|---:|---|---|
| F-d19a through F-d19e | 4Z | **J** | Store 4J at 16J + 20. |
| F-d20a through F-d20e | 4J | **K** | Store the retained value 4J at 4K + 20. |
| F-d21a through F-d21e | 4K | **N** | Store 4N at 16N + 20. |

These operations differ both in destination scaling and in which result supplies the stored value. In particular, the middle store retains the earlier value 4J across the intervening chain. No linked-list, table, or object-field interpretation is established.

F-d22a next receives 4N and 16N. F-d22b follows without explicit arguments. Its result and 16M, in that order, supply the first call of the ordinary chain F-d23a through F-d23e. Designate the final result **A**.

The function stores the word value 128M at 4A + 20. Since M contains at most 22 bits, 128M fits within 29 bits, and 16M fits within 26 bits; these masked-selector values themselves do not require 32-bit truncation.

The ordinary chain F-d24a through F-d24e initially receives 4A. Designate its final unsigned 32-bit result **D**.

### Final byte-selector updates

Designate the 32-bit product of Q and 2²⁶ as **E**.

The function reads a byte at E + 1 and stores the low eight bits of 4D at the selected byte destination.

F-d25a receives 4D. F-d25b follows without explicit arguments. Its result supplies the first argument of the ordinary chain F-d26a through F-d26e. Designate that chain’s result **G**.

The byte at E + 1 is read again. The selected destination receives the low eight bits of D masked to 30 bits and then shifted logically right by 20. Consequently, the stored byte contains D bits 20 through 27. The mask does not cause bits 28 or 29 to survive the final byte truncation.

The ordinary chain F-d27a through F-d27e initially receives 4G. Designate its final unsigned 32-bit result **J2**.

The function reads the byte at E + 1 a third time, saves that selector, and stores the low eight bits of 4J2 at the selected destination. F-d28a receives the saved selector zero-extended to 32 bits—not J2 or the destination address. F-d28b follows without explicit arguments.

The result of F-d28b supplies the first argument of the ordinary chain F-d29a through F-d29e. Designate its final result **K2**.

The next selector source is 8J2 + 1, **not E + 1**. The byte read there selects a destination that receives the low eight bits of J2 masked to 30 bits and shifted logically right by 20.

Finally, the ordinary chain F-d30a through F-d30e initially receives 4K2. The supplied representation then reaches its invalid-instruction boundary.

## Side effects

### Direct memory-access inventory

Conditional on reaching every listed operation, the visible body contains **sixteen four-byte stores total**:

| Category | Count | Contents and destinations |
|---|---:|---|
| Initial payload stores | 2 | P at B + 36 and 4S + 36. |
| Zero stores at offset 68 | 4 | C + 68, 4V + 68, 2H + 68, and 4W + 68. |
| Zero stores at absolute address 36 | 4 | Four separate writes at distinct points in the trace. |
| Zero stores at offset 20 | 2 | 4Y + 20 and 4Z + 20. |
| Scaled-value stores | 4 | 4J at 16J + 20; retained 4J at 4K + 20; 4N at 16N + 20; 128M at 4A + 20. |

Thus, **fourteen four-byte stores follow the two initial payload stores**: ten zero stores and four scaled-value stores. A scaled value may itself be zero for particular inputs or results, but its store remains in the scaled-value category.

The visible body also contains:

- Nine byte stores: four explicit zero stores and five truncated-value stores.
- Eight byte-selector reads.
- One explicit four-byte read, at 4S + 116.

These counts exclude all accesses performed inside dependencies. They count access events, not distinct addresses.

### Aliasing and partial execution

No disjointness is established among the entry destination, selector sources, selected destinations, word destinations, or dependency-accessed storage. Word writes may overlap byte selectors or other words. A direct write may therefore alter a later read even without dependency interference.

The low selected addresses and absolute address 36 are literal addresses in the supplied representation. They are not established offsets relative to an omitted object base.

No transaction, rollback, locking operation, or restoration is visible. If execution stops partway through, earlier effects are not established to be undone.

### Ownership and lifetime

The visible body establishes no allocation, release, ownership transfer, or lifetime extension. Dependencies may perform such actions, but their contracts are unavailable. Successful passage of a value to a dependency does not establish whether it is a pointer, handle, index, owned resource, or borrowed reference.

## Dependencies

F-d01a through F-d30e are opaque call-site IDs; only the members explicitly described above are present. Single-call and two-call groups do not imply omitted calls.

The first target depends on incoming link-related state. Later targets use embedded address-like values and fixed displacements. Their implementations, signatures, possible shared targets, and calling conventions remain unresolved.

The contract does not assume that a dependency:

- Returns or reports failure through a status value.
- Preserves globals, selector bytes, hidden registers, or other memory.
- Produces a valid address or bounded index.
- Acquires, retains, transfers, or releases ownership.
- Avoids additional externally observable effects.

The trace records represented value retention across calls. Whether those retained values accurately model machine-register behavior remains unresolved.

## Acceptance examples

These are conditional consequences of the static representation, not executed tests.

1. **Payload bits and ordering.** If both initial destinations are writable and the intervening chain returns, both payload stores carry the complete 32-bit P, including its high bit. If the destinations coincide, they remain two separate ordered writes.

2. **Selector payload truncation.** For S = 0xC0000000, shifting right by 22 yields 0x300. The byte store receives 0x00, not a ten-bit value.

3. **Selector re-read.** If the first read at 4S + 1 yields 7 and the later clearing read yields 9, the first destination is address 24 and the clear targets address 26. Clearing address 24 is not required by that later access.

4. **Saved selector.** If C = 255, the saved-selector word clear begins at address 323. Changes to the selector source during F-d08a through F-d08e do not change this represented destination.

5. **Masked versus unmasked input.** S = 0 and S = 0x02000000 both produce M = 0. Their Q values differ, so equality of the masked-selector arguments does not imply equality of earlier Q-derived addresses.

6. **Paired result separation.** If L = 3 and H = 5, F-d13a receives 12 and the subsequent word clear begins at address 78. F-d14a’s second argument remains undetermined without the additional register-derived result.

7. **Final byte payloads.** If D = 0x12345678, its first derived byte is 0xE0 and its later derived byte is 0x23. Separate selector reads determine their destinations.

8. **Store accounting despite aliasing.** If every operation is reached, the direct trace contains sixteen word-store events even when some destinations coincide. Fourteen follow the initial payload stores; ten of those fourteen explicitly store zero.

9. **Interrupted execution.** If a dependency does not return, later direct accesses are not required by the trace. Even if every listed dependency returns, the supplied evidence does not establish normal function return.

## Uncertainties

The following remain **documented_only**:

- Validity of the recovered instruction stream and function boundary.
- Runtime meaning of the final invalid-instruction marker.
- Exact dependency identities, signatures, hidden inputs, and error conventions.
- Accuracy of represented value retention across indirect calls.
- Meaning and preservation of the additional register-derived result.
- Address-space validity, permissions, alignment, and endianness.
- Signed-intermediate overflow, shifts, and conversion behavior where applicable.
- Application-level layouts, field meanings, synchronization, ownership, and lifetimes.
- Reachability of the complete trace.

Initialization-like intent does not resolve these unknowns or justify inventing object types, validation, safety guarantees, or success conditions.

## Evidence

**Established static evidence:** the supplied representation explicitly shows access widths, argument expressions, indirect-call ordering, represented result flows, sixteen word stores, nine byte stores, and a truncated invalid-instruction boundary. The word-store inventory follows from grouping the visible destinations and payloads, not from assuming any application semantics.

**Inferred intent:** the repeated clearing and scaled-value stores resemble initialization or reconfiguration. That interpretation is not required by this contract.

**Unknown behavior:** dependency effects, machine-level validity, and successful execution remain unresolved.

No implementation lookup, dependency inspection, runtime execution, smoke test, or other runtime test has been performed. The acceptance examples are conditional static checks, not observations of an executing implementation.
