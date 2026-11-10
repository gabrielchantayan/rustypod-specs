<!-- Trusted provenance: F-6a0ebb7bca73; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6a0ebb7bca73

## Purpose

The supplied representation describes a state-mutating routine dominated by indirect calls, word-sized stores, and byte stores through unusually small derived addresses. It does not establish a recognizable application-level purpose.

The observable contract is therefore an ordered sequence of memory accesses and dependency invocations, conditional on the recovered control flow being valid. The representation explicitly reports invalid instruction data and terminates at an analysis boundary rather than a demonstrated normal return. Consequently, this document does not establish that the entire sequence is executable or that its apparent endpoint is an actual processor halt.

**Established static evidence:** three explicit 32-bit inputs, additional ambient execution-state inputs, forty represented indirect calls, multiple direct stores, and no demonstrated normal return.

**Inferred intent:** some values may be indices or packed fields, given repeated scaling by four and extraction of bit ranges. No object, table, serialization format, or application role is established.

## Interface

### Explicit inputs

The represented interface has three explicit inputs:

| Input | Represented width | Direct use |
|---|---:|---|
| First input | 32 bits | No direct use appears in the supplied body. |
| Second input | 32 bits | No direct use appears in the supplied body. |
| Third input | 32 bits, represented as signed | Stored without scaling, then shifted left by three for subsequent use. |

The absence of direct uses of the first two inputs does not establish that indirect dependencies cannot observe their original register values or other entry-state remnants. The actual calling convention is unknown.

### Ambient inputs

Three additional entry-state values are materially used:

- **Ambient base:** an address-like value used for the first store and the following word load.
- **Ambient payload:** a 32-bit word later stored at an address derived from a dependency result.
- **Ambient link:** an address-like value used to form the first indirect call target by adding hexadecimal `0xFFE`.

These are not initialized within the represented body. A faithful caller or execution environment must supply meaningful values; the explicit three-input interface alone is insufficient.

### Output and completion

No explicit return value is represented. The visible body does not reach a normal return.

Results are observable through:

- Direct memory writes.
- Arguments and execution state presented to indirect dependencies.
- Dependency side effects.
- Any fault, transfer of control, or terminal behavior encountered along the path.

There is no visible status code, validation result, exception handler, or rollback mechanism.

### Width and address model

The representation uses:

- 32-bit words for ordinary loads, stores, arguments, and most dependency results.
- 64-bit containers for two dependency results, each separated into low and high 32-bit words.
- Unsigned 8-bit loads for byte-selected addresses.
- 8-bit stores for the final byte writes.

The formulas below use a **conditional 32-bit bit-vector interpretation**: arithmetic retains the low 32 bits, right shifts of unsigned words insert zeros, and byte stores retain the low eight bits. This interpretation makes the represented operations precise, but the supplied material does not independently establish processor arithmetic, pointer width, alignment rules, or signed-shift behavior. In particular, left shifts represented through signed values must not be treated as portable source-language arithmetic guarantees.

## Behavior

### Dependency labels and sequencing

Opaque labels **F-D01** through **F-D40** identify indirect call occurrences in execution order. They are document-local occurrence labels, not verified identities of forty distinct functions. The same target could potentially serve multiple occurrences.

Calls described as having “no displayed arguments” have no explicit argument expressions in the supplied representation. That does not establish a parameterless machine-level interface.

Each later step is conditional on earlier memory accesses succeeding and earlier dependencies returning to the represented continuation.

### 1. Publish the original input and obtain a secondary base

The first direct side effect is a 32-bit store of the third explicit input at:

**ambient base + `0x24`**

Only after that store, the routine reads a 32-bit word from:

**ambient base + `0x74`**

Call the loaded word the **secondary base**. It is subsequently used both as an address-like value and as an integer argument.

The third input is then shifted left by three, producing the **scaled input**.

F-D01 is invoked through the target derived from the ambient link. Four arguments are displayed, in this order:

1. Secondary base.
2. Secondary base shifted left by one.
3. Scaled input.
4. Secondary base with its highest bit cleared, then shifted right by two.

The fourth argument is therefore the value of original secondary-base bits 2 through 30, interpreted unsigned.

F-D02, F-D03, F-D04, and F-D05 follow, in that order, with no displayed arguments.

### 2. Publish the scaled input and read another field

After F-D05 returns, the routine stores the scaled input as a 32-bit word at:

**secondary base + `0x24`**

It then reads a 32-bit word from:

**secondary base + `0x74`**

That word is supplied as the single displayed argument to F-D06.

The second load is not taken from a snapshot established before the calls. F-D01 through F-D05 may have altered the memory it reads. Likewise, the preceding store may affect that load if address aliasing permits it.

F-D07, F-D08, and F-D09 follow with no displayed arguments.

### 3. Consume the first two-word result

F-D10 is then called with no displayed arguments. Its represented result is 64 bits wide.

The result is split into:

- **First low word:** bits 0 through 31.
- **First high word:** bits 32 through 63.

The first low word is shifted left by two, producing the **retained scaled word**.

Before another call occurs, the routine stores the ambient payload as a 32-bit word at:

**4 × first high word + `0x14`**

F-D11 follows with four displayed arguments:

1. Retained scaled word.
2. Retained scaled word.
3. Retained scaled word.
4. First high word shifted right by three.

The three identical arguments are distinct argument positions. No relationship between the store destination and those arguments is established beyond their arithmetic derivation.

### 4. Consume the second two-word result

F-D12 is called with no displayed arguments and produces another represented 64-bit result.

F-D13 receives four displayed arguments derived from that result:

1. Second low word.
2. Second high word.
3. Second low word again.
4. Second high word shifted right by three.

F-D14, F-D15, and F-D16 follow with no displayed arguments.

F-D17 then returns a represented 32-bit word, the **first returned index**.

The routine stores the retained scaled word from F-D10 at:

**4 × first returned index + `0x14`**

F-D18 is called with one displayed argument:

**4 × first returned index**

The retained value written here is not derived from F-D12 or F-D17. It remains the scaled low word of F-D10’s result.

### 5. Perform two byte writes using separately loaded selectors

F-D19, F-D20, and F-D21 are called with no displayed arguments.

F-D22 returns a represented unsigned 32-bit word, the **byte-source word**. Shifting it left by two produces the **byte-working word**.

The routine then:

1. Reads one unsigned byte from **secondary base + 1**.
2. Adds `0x11` to the byte value.
3. Stores the low eight bits of the byte-working word at that resulting address.

The loaded byte is the address selector itself. It is not added back to the secondary base. Consequently, the direct destination lies between `0x11` and `0x110`, inclusive, under the stated address interpretation.

F-D23 and F-D24 follow with no displayed arguments. F-D25 then returns another represented 32-bit word.

Before consuming that return value in a call, the routine performs a second byte-selected store:

1. It reads one unsigned byte from **secondary base + 1** again.
2. It adds `0x11` to this newly read byte.
3. It stores the low eight bits of the byte-source word after clearing its top two bits and shifting right by twenty.

The stored byte corresponds to original byte-source bits 20 through 27. Bits 28 and 29 survive the shift initially but are discarded by the byte store.

The selector is re-read; the second destination need not equal the first. Intervening dependencies may modify the selector. The first store itself could also change the selector if its destination aliases the selector address.

F-D26 is then called with the F-D25 result shifted left by two.

### 6. Perform the first zero-byte write

F-D27, F-D28, and F-D29 follow with no displayed arguments. F-D30 returns a represented 32-bit word.

The routine computes a selector-load address from the retained byte-working word:

**byte-working word × `0x400` + 1**

It reads an unsigned byte there, adds `0x11`, and writes a zero byte to that resulting address.

F-D31 is then invoked with four displayed arguments:

1. Byte-working word shifted left by three.
2. Byte-working word shifted left by three again.
3. Byte-working word.
4. F-D30’s result shifted left by two.

The byte-working word remains derived from F-D22. F-D30’s result does not replace it.

### 7. Repeat the zero-byte operation and finish the visible sequence

F-D32, F-D33, and F-D34 follow with no displayed arguments. F-D35 returns a represented 32-bit word.

The routine again reads an unsigned byte from:

**byte-working word × `0x400` + 1**

It again writes zero to the address obtained by adding `0x11` to the loaded byte.

This is a fresh selector read. The two zero stores are not guaranteed to share a destination.

F-D36 receives F-D35’s result shifted left by two.

Finally, F-D37, F-D38, F-D39, and F-D40 are invoked, in that order, with no displayed arguments.

The supplied representation then ends at an invalid-instruction boundary. Behavior beyond that point is not established.

## Side effects

The visible body contains eight direct stores, ordered as follows:

| Order | Width | Destination | Stored content |
|---:|---:|---|---|
| 1 | 32 bits | Ambient base + `0x24` | Original third input |
| 2 | 32 bits | Secondary base + `0x24` | Third input shifted left by three |
| 3 | 32 bits | Four times F-D10’s high word + `0x14` | Ambient payload |
| 4 | 32 bits | Four times F-D17’s result + `0x14` | F-D10’s low word shifted left by two |
| 5 | 8 bits | First secondary-base selector + `0x11` | Low byte of byte-working word |
| 6 | 8 bits | Second secondary-base selector + `0x11` | Byte-source bits 20 through 27 |
| 7 | 8 bits | First scaled-address selector + `0x11` | Zero |
| 8 | 8 bits | Second scaled-address selector + `0x11` | Zero |

These are access-order requirements, not guarantees of eight distinct storage locations. Aliasing can make later stores overwrite earlier results.

No memory allocation, release, ownership transfer, or lifetime extension is directly visible. Passing a word to a dependency does not establish whether it is a pointer or whether ownership changes. Dependencies may modify globals, modify reachable memory, allocate, release, or fail; their bodies are unavailable.

The visible path contains no null checks, bounds checks, address-range checks, alignment checks, or rollback. Valid mapped memory and valid call targets are execution prerequisites, not enforced conditions. Actual failure behavior depends on the execution environment.

## Dependencies

The complete displayed call order is F-D01 through F-D40.

Result-dependent occurrences are:

| Occurrence | Displayed result use |
|---|---|
| F-D10 | Two 32-bit words from a 64-bit result |
| F-D12 | Two 32-bit words from a 64-bit result |
| F-D17 | 32-bit index-like value |
| F-D22 | Unsigned 32-bit source for byte operations |
| F-D25 | 32-bit value subsequently shifted by two |
| F-D30 | 32-bit value subsequently shifted by two |
| F-D35 | 32-bit value subsequently shifted by two |

No explicit result is consumed from other occurrences. This does not establish that they return no value or have no side effects.

The first target depends on ambient entry state. Later targets are formed from embedded address-like values plus fixed displacements. The source does not establish target validity, executable mappings, calling conventions, preservation rules, or whether any call returns normally.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests. They assume valid memory, valid returning dependencies, the represented control flow, and the stated 32-bit arithmetic model.

### Original versus scaled input

For a third input of 7:

- The first direct store writes 7.
- The corresponding later store through the secondary base writes 56.
- F-D01’s third displayed argument is 56.

Using 56 in the first store, or using 7 in the second store, would not match the represented behavior.

### Initial argument masking

For a secondary-base word of `0x8000000C`, F-D01 receives:

- First argument: `0x8000000C`.
- Second argument: `0x00000018`.
- Fourth argument: 3.

The fourth argument is not an arithmetic right shift of the signed interpretation.

### Splitting and retaining a result

If F-D10 yields low word 5 and high word 3:

- The ambient payload is stored at `0x20`.
- F-D11 receives 20, 20, 20, and 0.
- If F-D17 later returns 9, the next indexed word store writes 20 at `0x38`.
- F-D18 receives 36.

These expectations require the low-derived value to remain available across the intervening calls.

### Byte extraction and selector re-reading

If F-D22 returns `0x01234567`, the first byte value is `0x9C` and the second byte value is `0x12`.

If the first selector read yields 4 and the second yields 9, the stores target `0x15` and `0x1A`, respectively. Reusing the first selector would violate the represented access sequence.

### Repeated zero writes

If F-D22 returns zero, the later scaled selector-load address is 1. If the two reads there produce 2 and then 5, the zero stores target `0x13` and `0x16`.

Identical selector-load addresses do not imply identical selector values across intervening calls.

### Conditional overflow behavior

For a third input with bit pattern `0x20000000`, the conditional 32-bit model gives a scaled input of zero.

This example does not establish that signed shifting in the unknown execution environment has that behavior.

## Uncertainties

The following remain **documented_only**:

- The application-level purpose and intended data structures.
- Whether invalid instruction data caused spurious calls, operations, or control-flow reconstruction.
- Whether the visible endpoint corresponds to any actual terminal instruction.
- The exact machine ABI, pointer width, alignment rules, and arithmetic semantics.
- The concrete identities and contracts of all indirect dependencies.
- Whether apparently argument-free calls consume live machine state.
- Whether dependencies preserve globals, memory, registers, or ownership.
- Whether the 64-bit results reflect native returns or paired machine values.
- Whether small byte-derived addresses are intentional, environment-specific, or reconstruction artifacts.
- The behavior of faults, exceptions, nonreturning dependencies, and execution beyond the visible boundary.

## Evidence

This contract is based solely on the supplied static representation: its explicit widths, memory accesses, arithmetic expressions, call order, result uses, and invalid-instruction warnings.

Access ordering is established by that representation. Application meaning and machine-level validity are not.

No runtime tests have been performed. The acceptance examples are arithmetic consequences of the documented conditional model, not observations of execution.
